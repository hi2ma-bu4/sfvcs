# sfvcs 基本設計書: `clone` / `fetch` / `push` 同期オペレーション & ネットワーク最適化
(`bd-network-sync-ops.md`)

---

# 1. 概要と目的

本設計書は、リモートリポジトリとの同期を実現する `clone`, `fetch`, `push` の詳細処理フロー、Sparse Submodule Sync, Dynamic Range Object Loading, Prefetch Window プロトコル, および QUIC / HTTP-3 セッション再開トークン (Session Resumption Token) の基本設計書である。

本書は `doc/02_specs/spec-network.md` の第4節 (4.1, 4.2, 4.3), 第5節, 第6節, 第7節, 第8節 (8.1), 第9節 (9.1) および `doc/01_architecture/sfvcs-design.md` の関連仕様を完全網羅し、カプセル化された Remote Synchronization Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Network Synchronization Service Layer** に位置し、リクエストの組み立てと転送の進行状況管理を担当する。

```
+-----------------------------------------------------------------------+
|               CLI (clone / fetch / push) / Remote API Facade          |
+-----------------------------------------------------------------------+
                                   | Invoke Network Sync Task
                                   v
+-----------------------------------------------------------------------+
|      bd-network-sync-ops (Sync Service Controller)                    |
|  - TypeScript: Progress Pipeline & Session Resumption Guard Manager   |
|  - Rust/WASM Core: Pack Building & Stream Multiplexing Engine         |
+-----------------------------------------------------------------------+
                                   | Wire Protocol Frames
                                   v
+-----------------------------------------------------------------------+
|                 Wire Protocol Transport Layer                         |
+-----------------------------------------------------------------------+
```

---

# 3. 同期オペレーション詳細仕様 (`clone`, `fetch`, `push`)

### 3.1 `clone` (完全複製)
1. 初期ハンドシェイク `MSG_HANDSHAKE_CAPABILITIES` (`0x00`) 交換。
2. `MSG_REF_DISCOVERY` (`0x01`) を送信し、リモート `refs/` 一覧（HEAD, heads/*, tags/*）および Commit CID を取得。
3. 目的のブランチ Root Commit CID を `MSG_TREE_NEGOTIATE_REQ` (`0x02`) で指定。
4. サーバーから到達可能全オブジェクトを含む Pack Stream (`MSG_PACKFILE_DATA`: `0x04`) を受領・ローカルインデックス化。

### 3.2 `fetch` (差分受信)
1. ローカルの `refs/remotes/<remote>/` とリモートの `refs/heads/` 差分を計算。
2. 差分 Commit CID につき Prolly Tree 最小差分交渉 (`MSG_TREE_NEGOTIATE_REQ` / `MSG_TREE_NEGOTIATE_RESP`) を実行。
3. 不足 Chunk / Node / Directory / Commit のみを含む Thin Pack をストリーミング受信し `.sfvcs/objects/pack/` に永続化。

### 3.3 `push` (差分送信)
1. ローカルの更新 Commit CID 一覧を準備。
2. リモートへ `MSG_TREE_NEGOTIATE_REQ` を送信し、相手側の欠落 CID リストを特定。
3. 不足オブジェクトのみを集約した Thin Packfile を構築し `MSG_PACKFILE_DATA` でストリーミング送信。
4. 送信完了後、アトミック参照更新リクエスト `MSG_REF_UPDATE_REQ` (`0x05`: `Expected_Old_CID` vs `New_CID`) を送信。
5. CAS 条件を検証し成功なら `MSG_REF_UPDATE_RESP` (`0x06`) で受理。競合時は `ERR_NON_FAST_FORWARD` エラー。

---

# 4. 高度プロトコル & パフォーマンス拡張仕様

### 4.1 Sparse Submodule Sync & LFS ストリーミング
- **Sparse Submodule Sync**: 親クローン時に `CAP_LAZY_FETCH` を有効化し、`ENTRY_SUBMODULE` ルートコミットのみ取得。配下オブジェクトはアクセス時に `MSG_LAZY_FETCH_REQ` (`0x07`) で動的取得。
- **LFS ストリーミング**: `MSG_LFS_POINTER_REQ` (`0x09`) で OID を指定し `MSG_LFS_DATA` (`0x0A`) で Range Request (RFC 7233) 方式による分割受領。

### 4.2 Dynamic Range Object Loading Protocol (`0x0B`, `0x0C`)
未取得の巨大 Sequence Node / Subtree オブジェクトアクセス時、`MSG_DYNAMIC_RANGE_REQ` (`0x0B`: CID, Offset, Length) を発行し、サーバーより `MSG_DYNAMIC_RANGE_RESP` (`0x0C`) で直接部分受信する。

### 4.3 Submodule Recursive Sync Protocol
`ENTRY_SUBMODULE` 検出時、親リポジトリとネストした各サブモジュールの交渉メッセージを `Channel ID` (`0x0001`, `0x0002`...) ごとに分離・多重化し、並列交渉を実行。

### 4.4 N+1 ラウンドトリップ回避バッチプリフェッチ (`MSG_PREFETCH_BATCH_REQ: 0x0D`)
欠落 CID 要求が発生した際、50 ms または 100 CIDs までバッファリングし、単一の `MSG_PREFETCH_BATCH_REQ` (`0x0D`) として送信。サーバーは `Depth_Limit` (2階層) までの配下ツリーを先回り一括送信する。

### 4.5 QUIC / HTTP-3 セッション再開トークン (`0x10`, `0x11`)
サーバーより発行された 24時間有効な `MSG_SESSION_TOKEN` (`0x10`) を保持し、切断復旧時に 0-RTT パケットへ `MSG_RESUME_STREAM_REQ` (`0x11`) を埋め込み、中断オフセット以降のバイトストリームを再開する。

---

# 5. TypeScript / WASM インターフェース

```typescript
export interface SyncProgressModel {
  phase: "handshake" | "discovering" | "negotiating" | "downloading" | "indexing";
  receivedBytes: bigint;
  totalBytes: bigint;
  processedObjects: number;
}

export interface SyncServiceFacade {
  clone(remoteUrl: string, targetPath: string, onProgress?: (p: SyncProgressModel) => void): Promise<void>;
  fetch(remoteUrl: string, options?: { branch?: string }): Promise<void>;
  push(remoteUrl: string, options?: { force?: boolean }): Promise<void>;
}
```
