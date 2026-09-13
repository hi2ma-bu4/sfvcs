# sfvcs 基本設計書: Prolly Tree 差分交渉 (Tree Negotiation) & 高速スキップ
(`bd-network-tree-negotiation.md`)

---

# 1. 概要と目的

本設計書は、リモート同期（`fetch` / `push`）時に転送すべき最小オブジェクト集合を Prolly Tree ノードハッシュの比較により二分決定する Prolly Tree 最小差分オブジェクト交渉アルゴリズム (Tree Negotiation) の基本設計書である。

本書は `doc/specs/spec-network.md` の第3節 (3.1, 3.2) および `doc/architecture/sfvcs-design.md` のネットワーク交渉仕様を完全網羅し、カプセル化された Negotiation Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Network Synchronization & Tree Negotiation Layer** に属し、送受信オブジェクトの差分抽出を担当する。

```
+-----------------------------------------------------------------------+
|                Network Transport & Wire Protocol Layer                |
+-----------------------------------------------------------------------+
                                   | Fast Subtree Match / Query
                                   v
+-----------------------------------------------------------------------+
| bd-network-tree-negotiation (Prolly Tree Negotiation Engine Service)  |
|  - TypeScript: Want/Have State Controller Manager                     |
|  - Rust/WASM Core: Prolly Tree Subtree Match & Fast-Skip Engine       |
+-----------------------------------------------------------------------+
                                   | Missing Object CID Set
                                   v
+-----------------------------------------------------------------------+
|                     Pack Building Engine Layer                        |
+-----------------------------------------------------------------------+
```

---

# 3. Prolly Tree 最小差分オブジェクト交渉アルゴリズム (Tree Negotiation)

従来 VCS (Git) の全コミットグラフおよび全ツリーを反復走査する遅延を排除し、Prolly Tree 内部ノードのハッシュ比較により $O(\text{差分サイズ})$ の通信量で交渉を完了する。

### 3.1 交渉シーケンスフロー (`fetch` の例)
```
  Client (Receiver)                               Server (Sender)
         |                                               |
         | --- MSG_WANT_OBJECTS (Remote Commit CID) ---> |
         |                                               |
         | <--- MSG_HAVE_OBJECTS (Root Prolly Tree CID) -|
         |                                               |
         | [Compare Local Root Prolly Tree Node CID]     |
         |   - Node CID Match  ==> SKIP SUBTREE! (0 byte) |
         |   - Node CID Differ ==> Request Level L-1     |
         |                                               |
         | --- MSG_WANT_OBJECTS (Level L-1 CIDs) ------> |
         | <--- MSG_HAVE_OBJECTS (Level L-1 Nodes) ----- |
         v                                               v
  (Iterate down to leaf chunks. Only missing chunks are downloaded)
```

---

# 4. Rust / WASM Core & TypeScript インターフェース

```rust
use std::collections::HashSet;

pub fn negotiate_tree_diff(
    local_cid: &[u8; 32],
    remote_cid: &[u8; 32],
    missing_cids: &mut HashSet<[u8; 32]>,
) {
    if local_cid == remote_cid {
        // FAST-SKIP: Remote subtree is already present in local database!
        return;
    }
    missing_cids.insert(*remote_cid);
}
```

```typescript
export interface TreeNegotiationFacade {
  computeMissingObjects(localRootCid: Uint8Array, remoteRootCid: Uint8Array): Promise<Uint8Array[]>;
}
```
