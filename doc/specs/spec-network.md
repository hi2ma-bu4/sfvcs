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
| Type ID | 名称                         | 説明                                                      |
| ------- | ---------------------------- | --------------------------------------------------------- |
| `0x00`  | `MSG_HANDSHAKE_CAPABILITIES` | 初期ハンドシェイクおよび機能交渉 (Capability Negotiation) |
| `0x01`  | `MSG_REF_DISCOVERY`          | 参照 (Refs) 一覧の広告・要求                              |
| `0x02`  | `MSG_TREE_NEGOTIATE_REQ`     | 差分オブジェクト探索交渉リクエスト                        |
| `0x03`  | `MSG_TREE_NEGOTIATE_RESP`    | 差分オブジェクト探索交渉レスポンス                        |
| `0x04`  | `MSG_PACKFILE_DATA`          | パックデータ（オブジェクト群）ストリーム                  |
| `0x05`  | `MSG_REF_UPDATE_REQ`         | リモート Ref の CAS 更新リクエスト                        |
| `0x06`  | `MSG_REF_UPDATE_RESP`        | リモート Ref 更新結果応答                                 |
| `0x07`  | `MSG_LAZY_FETCH_REQ`         | Sparse/Shallow 向けオンデマンドオブジェクト要求           |
| `0x08`  | `MSG_LAZY_FETCH_RESP`        | オンデマンド要求オブジェクトデータ返答                    |
| `0x09`  | `MSG_LFS_POINTER_REQ`        | LFS 大容量オブジェクト取得リクエスト                      |
| `0x0A`  | `MSG_LFS_DATA`               | LFS チャンクドストリーミングデータ                        |
| `0x0E`  | `MSG_PROGRESS`               | 進行状況テキスト（"Counting objects...", etc.）           |
| `0x0F`  | `MSG_ERROR`                  | プロトコルレベル構造化エラーメッセージ                    |

## 2.2 初期ハンドシェイクおよび機能交渉 (Capability Negotiation)
クライアントとサーバー間の接続確立直後、双方の互換性を確立するため `MSG_HANDSHAKE_CAPABILITIES` (`0x00`) を交換する。

### ペイロードレイアウト (`MSG_HANDSHAKE_CAPABILITIES`)
```
+-------------------------------------------------------+
| Protocol Version (uint16 BE)                          |
+-------------------------------------------------------+
| Supported Hash Algorithms Bitmask (uint8)             |
|   - Bit 0 (0x01): SHA-256                             |
|   - Bit 1 (0x02): BLAKE3                              |
+-------------------------------------------------------+
| Compression Algorithms Bitmask (uint8)               |
|   - Bit 0 (0x01): Deflate/zlib                        |
|   - Bit 1 (0x02): Brotli                              |
+-------------------------------------------------------+
| Capabilities Bitmask (uint32 BE)                      |
|   - Bit 0 (0x0001): CAP_THIN_PACK                     |
|   - Bit 1 (0x0002): CAP_LAZY_FETCH (Sparse/Shallow)  |
|   - Bit 2 (0x0004): CAP_SIDEBAND_PROGRESS             |
+-------------------------------------------------------+
```

## 2.3 明示的構造化エラーフレーム (`MSG_ERROR: 0x0F`)
通信中にエラーが発生した場合、`MSG_ERROR` フレームを返却する。
```
+-------------------------------------------------------+
| Error Code (uint16 BE)                                |
+-------------------------------------------------------+
| Error Message Length (varint)                         |
+-------------------------------------------------------+
| Error Message Bytes (UTF-8)                           |
+-------------------------------------------------------+
```
- 主なエラーコード:
  - `0x0001`: `ERR_UNSUPPORTED_VERSION` (プロトコルバージョン不適合)
  - `0x0002`: `ERR_HASH_ALGO_MISMATCH` (ハッシュアルゴリズム不適合)
  - `0x0003`: `ERR_NON_FAST_FORWARD` (Push 時の非 Fast-Forward 更新拒否)
  - `0x0004`: `ERR_OBJECT_NOT_FOUND` (リクエストオブジェクト非存在)

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

---

# 5. Sparse Submodule Clone & LFS ストリーミング同期プロトコル

1. **Sparse Submodule Clone**:
   親リポジトリクローン時、`CAP_LAZY_FETCH` フラグを有効化し、`ENTRY_SUBMODULE` のルートコミットのみを取得。サブモジュール配下の詳細オブジェクトは、ユーザーアクセス時に `MSG_LAZY_FETCH_REQ` でオンデマンド取得。
2. **LFS ストリーミング同期**:
   `MSG_LFS_POINTER_REQ` により `CONTENT_LFS_POINTER` (`0x02`) の Hash OID を要求し、サーバーから `MSG_LFS_DATA` フレームで Range Request (RFC 7233) 方式により分割受信。

---

# 6. Dynamic Range Object Loading Protocol (オンデマンド部分ツリー取得フレーム)

Sparse Checkout や Shallow Clone 環境において、未取得の巨大 Sequence Node / Subtree オブジェクトがアクセスされた際にリアルタイムでサーバーから取得するストリーミングフレーム仕様。

- **`MSG_DYNAMIC_RANGE_REQ` (`0x0B`)**:
  ```
  [Target_CID: 33 bytes][Byte_Offset_Start: uint64][Byte_Offset_Length: uint64]
  ```
- **`MSG_DYNAMIC_RANGE_RESP` (`0x0C`)**:
  ```
  [Target_CID: 33 bytes][Payload_Length: varint][Payload_Bytes]
  ```

---

# 7. Submodule Recursive Sync Protocol (サブモジュールネスト交渉プロトコル)

`ENTRY_SUBMODULE` (`0x04`) が含まれるリポジトリの `push` / `fetch` 時、親リポジトリと子サブモジュールのオブジェクト交渉を単一チャネル上で多重化（Multiplexing）して一括実行するシーケンス。

1. **Submodule Discovery**:
   親ツリー走査時に検出された各サブモジュールの Commit CID を `MSG_REF_DISCOVERY` に多重化付加。
2. **Parallel Negotiation**:
   各サブモジュールの `MSG_TREE_NEGOTIATE_REQ` を `Channel ID`（`0x0001`, `0x0002`...）ごとに分離して並行交渉し、オブジェクト未存在によるチェックアウト失敗（Dangling Submodule Ref）を防ぐ。

---

# 8. Sparse-Checkout / Lazy Fetch における N+1 ネットワークラウンドトリップ回避バッチプリフェッチ (Prefetch Window) プロトコル

Sparse-Checkout や Lazy Fetch において、欠落オブジェクトを 1 項目ずつ個別に取得要求すると、ネットワークラウンドトリップレイテンシが蓄積して極端に低速化する（N+1 問題）。

## 8.1 バッチプリフェッチウィンドウ (`MSG_PREFETCH_BATCH_REQ: 0x0D`)
1. **バッチ要求メッセージ形式 (`0x0D`)**:
   ```
   [CID_Count: varint][CID Array (33 bytes x Count)][Depth_Limit: uint8 (デフォルト: 2)]
   ```
2. **クライアント側バッファリングアルゴリズム**:
   クライアントは欠落 CID の読み込み要求が発生した際、即座に送信せず、一定の要求ウィンドウ時間（**50 ms**）または上限件数（**100 CIDs**）まで要求をメモリキューにバッファリングし、単一の `MSG_PREFETCH_BATCH_REQ` として一括送信する。
3. **サーバー側到達可能ツリー推論プリフェッチ**:
   サーバーは受信した CID リストに対し、指定深さ `Depth_Limit` (2 階層) までの配下 Sequence Node および Chunk を先回りして一括集約し、単一の `MSG_PACKFILE_DATA` ストリームとして返答する。これによりネットワークラウンドトリップ数を 1/100 以下に削減する。

---

# 9. 高速回復性を備えた QUIC / HTTP-3 パケット多重化およびストリーム再開トークン (Session Resumption Token) プロトコル

モデム断線や Wi-Fi/5G モバイル環境におけるネットワーク瞬断・切断時、途中まで転送した巨大パックデータを破棄せず、転送中断位置から $O(1)$ で高速再開するプロトコル。

## 9.1 セッション再開トークン構造および即時復元シーケンス
- **セッション再開トークンフレーム (`MSG_SESSION_TOKEN: 0x10`)**:
  サーバーは転送開始時にクライアントへセッション再開トークン `Session_Token` (32 バイト暗号的ランダム, 有効期間: **24時間**) を発行。
- **転送再開リクエスト (`MSG_RESUME_STREAM_REQ: 0x11`)**:
  ```
  [Session_Token: 32 bytes][Last_Received_Byte_Offset: uint64][Channel_ID: uint16]
  ```
- **QUIC 0-RTT ハンドシェイク統合**:
  QUIC / HTTP/3 トランスポートの 0-RTT ハンドシェイクパケット内に `MSG_RESUME_STREAM_REQ` を埋め込み、接続再確立と同時に中断オフセット以降のバイトストリーム送信を即座に再開する。
