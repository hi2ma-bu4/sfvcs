# sfvcs リモート同期 ・ Wire Protocol 詳細仕様書

本書は `sfvcs` におけるリモートリポジトリとの通信プロトコル（Wire Protocol）、Prolly Tree の根（Root Node）比較による超高速な最小差分オブジェクト探索交渉アルゴリズム、および `fetch` / `push` / `clone` 同期処理のバイナリメッセージ仕様を定義する詳細仕様書である。

---

# 1. 通信プロトコル概要

`sfvcs` のリモート同期は、トランスポート層（HTTP/HTTPS または SSH）上で動作するスマート転送プロトコル（Smart Transfer Protocol）を採用する。

Git などの従来の VCS プロトコルとの最大の違いは、**Prolly Tree / Sequence Tree の Content-Addressed な性質を最大限に活用し、ツリーのルート CID 同士を階層的に比較することで、転送が必要な不一致オブジェクト集合を対数時間 $O(\log N)$ で特定できる点** である。

---

# 2. Wire Protocol メッセージ構造

すべての転送メッセージはフレーム化されたバイナリ形式とし、多重化通信（Multiplexing）に対応する。

## 2.1 フレームレイアウト
```
+-------------------------------------------------------+
| Frame Type (uint8)                                    |
+-------------------------------------------------------+
| Channel ID (uint16 BE)                                |
+-------------------------------------------------------+
| Payload Length (uint32 BE)                            |
+-------------------------------------------------------+
| Payload Bytes (Payload Length バイト)                 |
+-------------------------------------------------------+
```

### Frame Type 一覧
| Type ID | 名称 | 説明 |
|---|---|---|
| `0x01` | `MSG_REF_DISCOVERY` | 参照 (Refs) 一覧の広告・要求 |
| `0x02` | `MSG_TREE_NEGOTIATE_REQ` | 差分オブジェクト探索交渉リクエスト |
| `0x03` | `MSG_TREE_NEGOTIATE_RESP` | 差分オブジェクト探索交渉レスポンス |
| `0x04` | `MSG_PACKFILE_DATA` | パックデータ（オブジェクト群）ストリーム |
| `0x05` | `MSG_REF_UPDATE_REQ` | リモート Ref の CAS 更新リクエスト |
| `0x06` | `MSG_REF_UPDATE_RESP` | リモート Ref 更新結果応答 |
| `0x0E` | `MSG_PROGRESS` | 進行状況テキスト（"Counting objects...", etc.） |
| `0x0F` | `MSG_ERROR` | プロトコルレベルエラーメッセージ |

---

# 3. Prolly Tree 最小差分オブジェクト交渉アルゴリズム (Tree Negotiation)

クライアントがリモートからデータを取得 (`fetch`) または送信 (`push`) する際、両者間で「既に相手が持っているオブジェクト」と「不足しているオブジェクト」の最小共通集合（Minimal Packset）を交渉決定する。

## 3.1 交渉シーケンスフロー (`fetch` 例)

```
Client (Local)                                        Server (Remote)
      │                                                      │
      │ 1. MSG_REF_DISCOVERY Request                         │
      ├─────────────────────────────────────────────────────>│
      │                                                      │
      │ 2. MSG_REF_DISCOVERY Response (Remote Refs map)      │
      │<─────────────────────────────────────────────────────┤
      │                                                      │
      │ 3. Determine target commits:                         │
      │    Wants: Remote HEAD CIDs                           │
      │    Haves: Local Commit CIDs                          │
      │                                                      │
      │ 4. MSG_TREE_NEGOTIATE_REQ (Want/Have Commit/Tree CIDs)
      ├─────────────────────────────────────────────────────>│
      │                                                      │
      │ 5. Tree-level fast recursive negotiation:            │
      │    If Tree_CID match -> Remote skips whole subtree!   │
      │    If Tree_CID mismatch -> Traverse missing child CIDs│
      │                                                      │
      │ 6. MSG_TREE_NEGOTIATE_RESP (Missing CID list)       │
      │<─────────────────────────────────────────────────────┤
      │                                                      │
      │ 7. Pack construction & streaming                     │
      │ 8. MSG_PACKFILE_DATA (Streamed Packfile)             │
      │<─────────────────────────────────────────────────────┤
```

## 3.2 高速サブツリースキップ判定擬似コード

```python
def find_missing_objects(client_haves: Set[CID], target_cid: CID, missing_list: List[CID]):
    # 相手がすでにこの CID を持っている場合はサブツリー全体を即座にスキップ ($O(1)$)
    if target_cid in client_haves or remote_storage_has(target_cid):
        return

    missing_list.append(target_cid)
    obj = load_object(target_cid)

    if obj.type == "Directory":
        for entry in obj.entries:
            find_missing_objects(client_haves, entry.cid, missing_list)
    elif obj.type == "Sequence":
        for child_cid in obj.child_cids:
            find_missing_objects(client_haves, child_cid, missing_list)
    elif obj.type == "Commit":
        find_missing_objects(client_haves, obj.root_directory_cid, missing_list)
        for parent_cid in obj.parents:
            find_missing_objects(client_haves, parent_cid, missing_list)
```

---

# 4. 同期オペレーション詳細仕様

## 4.1 `clone` (完全複製)
1. クライアントが `MSG_REF_DISCOVERY` を送信。
2. サーバーが全 Reference (HEAD, heads/*, tags/*) とそれぞれの Commit CID を返答。
3. クライアントが指定ブランチの Root Commit CID を `Want` としてリクエスト。
4. サーバーは全到達可能オブジェクトを包含するパックファイルを作成・ストリーミング送信。

## 4.2 `fetch` (差分受信)
1. ローカルの `refs/remotes/<remote>/` とリモートの `refs/heads/` の差分を計算。
2. 差分 Commit に紐づく Tree CID について、上記 Prolly Tree 最小差分交渉を実行。
3. 不足している Chunk / Node / Directory / Commit のみを含む Thin Pack を受信し、ローカルの `.sfvcs/objects/pack/` に保存。

## 4.3 `push` (差分送信)
1. クライアントがローカルの更新コミット CID を準備。
2. リモートに対して `MSG_TREE_NEGOTIATE_REQ` を送信し、リモート側に不足しているオブジェクト CID 一覧を取得。
3. クライアントが必要なオブジェクトのみを集約した Packfile を生成して送信 (`MSG_PACKFILE_DATA`)。
4. 送信完了後、アトミック参照更新リクエスト (`MSG_REF_UPDATE_REQ`: `Expected_Old_CID` vs `New_CID`) を送信。
5. サーバーが CAS 条件を検証し、成功すれば Ref を更新。競合した場合は `ERR_NON_FAST_FORWARD` を返却。
