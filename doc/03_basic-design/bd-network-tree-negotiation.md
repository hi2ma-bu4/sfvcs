# sfvcs 基本設計書: Prolly Tree 差分交渉 (Tree Negotiation) & 高速スキップ
(`bd-network-tree-negotiation.md`)

---

# 1. 概要と目的

本設計書は、リモート同期（`fetch` / `push`）時に転送すべき最小オブジェクト集合を Prolly Tree ノードハッシュの比較により決定する Prolly Tree 最小差分オブジェクト交渉アルゴリズム (Tree Negotiation) およびサブモジュール再帰交渉プロトコルの基本設計書である。

本書は `doc/02_specs/spec-network.md` の第3節 (3.1, 3.2) および第7節並びに `doc/01_architecture/sfvcs-design.md` のネットワーク交渉仕様を完全網羅し、カプセル化された Negotiation Service モジュールとして詳細を定義する。

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

### 3.1 交渉シーケンスフロー (`fetch` / `push` の例)
```
  Client (Local)                                      Server (Remote)
         │                                                   │
         │ 1. MSG_REF_DISCOVERY                              │
         ├──────────────────────────────────────────────────>│
         │ 2. MSG_REF_DISCOVERY Resp (Remote Refs)           │
         │<──────────────────────────────────────────────────┤
         │                                                   │
         │ 3. Determine Wants & Haves                        │
         │                                                   │
         │ 4. MSG_TREE_NEGOTIATE_REQ (Want/Have CIDs)        │
         ├──────────────────────────────────────────────────>│
         │                                                   │
         │ 5. Tree-level Fast Recursive Negotiation:         │
         │    If Node CID Match  -> SKIP SUBTREE! (0 byte)   │
         │    If Node CID Differ -> Traverse Child CIDs      │
         │                                                   │
         │ 6. MSG_TREE_NEGOTIATE_RESP (Missing CID Array)    │
         │<──────────────────────────────────────────────────┤
         │                                                   │
         │ 7. Pack construction & streaming                  │
         │ 8. MSG_PACKFILE_DATA (Streamed Packfile)          │
         │<──────────────────────────────────────────────────┤
```

### 3.2 サブモジュール再帰交渉プロトコル (Submodule Recursive Sync Protocol)
親ツリー走査時に `ENTRY_SUBMODULE` (`0x04`) を検出した場合、親リポジトリとネストした各サブモジュールの Commit CID を `Channel ID` (`0x0001`, `0x0002`...) ごとに分離して多重化 `MSG_TREE_NEGOTIATE_REQ` を送信し、並列交渉によってサブモジュール未存在（Dangling Ref）のチェックアウト失敗を防止する。

---

# 4. Rust / WASM Core & TypeScript インターフェース

```rust
use std::collections::HashSet;

pub fn negotiate_tree_diff(
    local_haves: &HashSet<[u8; 33]>,
    target_cid: &[u8; 33],
    missing_cids: &mut Vec<[u8; 33]>,
    storage_provider: &impl StorageProvider,
) {
    if local_haves.contains(target_cid) || storage_provider.has_object(target_cid) {
        // FAST-SKIP: Subtree is identical and already present in target storage!
        return;
    }
    missing_cids.push(*target_cid);
    if let Some(obj) = storage_provider.load_object(target_cid) {
        if obj.is_sequence_or_directory() {
            for child_cid in obj.child_cids() {
                negotiate_tree_diff(local_haves, &child_cid, missing_cids, storage_provider);
            }
        }
    }
}
```

```typescript
export interface TreeNegotiationFacade {
  computeMissingObjects(
    localHaves: Uint8Array[],
    targetCid: Uint8Array,
    channelId?: number
  ): Promise<Uint8Array[]>;
}
```
