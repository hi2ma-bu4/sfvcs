# sfvcs 基本設計書: `clone` / `fetch` / `push` 同期オペレーション & ネットワーク最適化
(`bd-network-sync-ops.md`)

---

# 1. 概要と目的

本設計書は、リモートリポジトリとの同期を実現する `clone`, `fetch`, `push` の詳細処理フロー、Sparse Submodule Sync, Dynamic Range Object Loading, Prefetch Window プロトコル, および QUIC / HTTP-3 セッション再開トークン (Session Resumption Token) の基本設計書である。

本書は `doc/specs/spec-network.md` の第4節 (4.1, 4.2, 4.3), 第5節, 第6節, 第7節, 第8節 (8.1), 第9節 (9.1) および `doc/01_architecture/sfvcs-design.md` の関連仕様を完全網羅し、カプセル化された Remote Synchronization Service モジュールとして詳細を定義する。

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
1. 初期ハンドシェイク `MSG_HANDSHAKE_CAPABILITIES`。
2. リモート `refs/` 一覧を取得し、目的の HEAD またはブランチ CID を決定。
3. リモートから全ルートオブジェクトおよび依存オブジェクトを Pack Stream として連続受信して保存。

### 3.2 `fetch` (差分受信)
1. ローカル Ref とリモート Ref を比較。
2. `bd-network-tree-negotiation` モジュールを呼び出し、差分オブジェクト CID 一覧を特定。
3. 不足オブジェクトのみを Packfile としてストリーミング受信。

### 3.3 `push` (差分送信)
1. リモートの最新 Ref を確認し、非 Fast-Forward 更新の場合は拒否（`--force` フラグを除く）。
2. ローカルで不足しているオブジェクト群を即座に Thin Packfile としてビルドし、リモートへ送信。
3. リモートで Thin Delta 解除および Reference の CAS アトミック更新。

---

# 4. Sparse-Checkout / Lazy Fetch バッチプリフェッチ & QUIC セッション再開

### 4.1 N+1 ラウンドトリップ回避バッチプリフェッチ (`MSG_PREFETCH_BATCH_REQ: 0x0D`)
Sparse Checkout や Lazy Fetch 有効時、遅延読み込みによって個別のオブジェクト取得リクエストが頻発する N+1 ラウンドトリップ問題を回避するため、必要と予測されるサブツリーノード群の CID 一覧を一括リクエストフレーム (`MSG_PREFETCH_BATCH_REQ`) でバッチ受信する。

### 4.2 QUIC / HTTP-3 ストリーム再開トークン (Session Resumption Token)
ネットワーク切断時（モバイル回線切り替え等）、接続再開トークン (`Session Resumption Token`) を用いて直前のストリーム読み取りオフセット位置から 0-RTT で同期処理をシームレスに再開する。

---

# 5. TypeScript / WASM インターフェース

```typescript
export interface SyncProgressModel {
  phase: "negotiating" | "downloading" | "indexing";
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
