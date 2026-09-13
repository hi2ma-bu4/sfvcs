# sfvcs 基本設計書: Tree Move 競合 & Kleppmann CRDT アルゴリズム
(`bd-merge-crdt-move.md`)

---

# 1. 概要と目的

本設計書は、分散環境や並行ブランチ操作においてディレクトリツリーの移動が衝突した際に、閉環（Cycle Conflict）や重複移動（Double Move Conflict）を代数的データ構造により自動解決する Kleppmann et al. Tree Move CRDT アルゴリズムの基本設計書である。

本書は `doc/specs/spec-merge.md` の第4節 (4.1, 4.2), 第9節 (9.1) および `doc/architecture/sfvcs-design.md` の第171.13, 171.14節の仕様を完全網羅し、カプセル化された CRDT Tree Movement Resolver モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Tree CRDT Resolution Layer** に位置し、複雑なディレクトリツリー再配置操作の一貫性を保証する。

```
+-----------------------------------------------------------------------+
|                     Tree Merge Service Layer                          |
+-----------------------------------------------------------------------+
                                   | Resolve Concurrent Move Conflicts
                                   v
+-----------------------------------------------------------------------+
|    bd-merge-crdt-move (Tree Move CRDT Resolver Service)               |
|  - TypeScript: Operation Log & State Machine Manager                  |
|  - Rust/WASM Core: Cycle Detection & Lamport Clock Evaluator Engine   |
+-----------------------------------------------------------------------+
                                   | Resolved Valid Tree Graph
                                   v
+-----------------------------------------------------------------------+
|                    Storage / Index Layer                              |
+-----------------------------------------------------------------------+
```

---

# 3. Concurrent Tree Move 競合パターンと CRDT 解決

### 3.1 閉環競合 (Cycle Conflict) と解決
- **発生例**: ブランチ A でディレクトリ `/X` を `/Y` の下に移動し、並行してブランチ B でディレクトリ `/Y` を `/X` の下に移動（$/X \to /Y \land /Y \to /X$）。単純に統合するとツリー構造が閉鎖ループし、根ノードを失う。
- **Kleppmann CRDT 解決アルゴリズム**:
  1. 全移動操作に（Lamport Timestamp, Client ID）を割り当てる。
  2. アトミックな操作ログ（Operation Log）をタイムスタンプ順に決定論的適用。
  3. 閉環を検出した場合、タイムスタンプが小さい（古い）方の移動操作を取り消し（Undo/Fallback）、安全な元の親ディレクトリに復元する。

### 3.2 重複移動競合 (Double Move Conflict)
- **発生例**: 同一ファイル/ディレクトリ `/A/file.txt` を、ブランチ A が `/B/file.txt` に、ブランチ B が `/C/file.txt` に移動。
- **解決ルール**: タイムスタンプが優位な方をプライマリ位置に移動し、他方を衝突マーカーまたは対話的フォールバック（例: `/C/file.txt.conflict_theirs`）として記録する。

---

# 4. Concurrent Move / Delete 競合解法 (移動と削除の平行衝突)

一方がファイルを別パスに移動し、他方が元パスのファイルを削除した場合:

$$\text{Priority Rule}: \text{Move Operation} > \text{Delete Operation}$$

- **自動判定**: 移動されたコンテンツを優先生存させ、移動先のパスに最新コンテンツを保存した上で削除操作を無効化する。

---

# 5. Rust / WASM Core & TypeScript インターフェース

```rust
pub struct CrdtMoveOp {
    pub lamport_ts: u64,
    pub client_id: String,
    pub node_id: String,
    pub new_parent_id: String,
}

pub fn detect_tree_cycle(tree: &std::collections::HashMap<String, String>, start_node: &str, target_parent: &str) -> bool {
    let mut curr = target_parent;
    while let Some(parent) = tree.get(curr) {
        if parent == start_node {
            return true; // Cycle detected!
        }
        curr = parent;
    }
    false
}
```

```typescript
export interface MoveOperationModel {
  opId: string;
  timestamp: bigint;
  clientId: string;
  targetNodeId: string;
  newParentId: string;
  newName: string;
}

export interface TreeMoveCrdtFacade {
  applyOperations(ops: MoveOperationModel[]): Promise<{ resolvedTree: Map<string, string>; undoneOps: MoveOperationModel[] }>;
}
```
