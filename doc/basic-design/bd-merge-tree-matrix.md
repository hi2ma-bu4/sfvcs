# sfvcs 基本設計書: Directory Tree & File Level 3-Way Merge & Fast-Path スキップ
(`bd-merge-tree-matrix.md`)

---

# 1. 概要と目的

本設計書は、ベースコミット (Base)、自ブランチ (Ours)、他方ブランチ (Theirs) の 3 つのツリー構造から変更箇所を自動判定し統合を行う Directory Tree & File Level 3-Way Merge、エントリ変更マトリクス、および階層的 Fast-Path スキップの基本設計書である。

本書は `doc/specs/spec-merge.md` の第1節 (1.1), 第2節 (2.1), 第3節 (3.1) および `doc/architecture/sfvcs-design.md` の関連仕様を完全網羅し、カプセル化された Tree Merge Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Domain Service & Merge Core Layer** に属し、ツリー構造の結合とコンフリクト判定を担当する。

```
+-----------------------------------------------------------------------+
|              CLI (sfvcs merge / rebase / cherry-pick)                 |
+-----------------------------------------------------------------------+
                                   | Request Tree Merge (Base, Ours, Theirs)
                                   v
+-----------------------------------------------------------------------+
|        bd-merge-tree-matrix (Tree Merge Engine Service)               |
|  - TypeScript: Tree Merger & Conflict Matrix Resolver Controller       |
|  - Rust/WASM Core: Prolly Tree Hierarchical Fast-Path Engine          |
+-----------------------------------------------------------------------+
                                   | Merged Tree CID / Conflicts List
                                   v
+-----------------------------------------------------------------------+
|               Working Tree / Storage Layer                            |
+-----------------------------------------------------------------------+
```

---

# 3. 階層的 Fast-Path スキップ (Hierarchical Fast-Path Skip)

マージ処理の計算量を削減するため、ディレクトリノード (`SFDR`) またはファイルノード (`SFFL`) の CID (BLAKE3 ハッシュ) の完全一致を利用してサブツリー全体の評価を高速スキップする。

### 3.1 スキップ条件ルール
1. **Ours == Theirs**: 双方で全く同じ変更または変更なし $\implies$ **Ours 採用** ($O(1)$ スキップ)。
2. **Base == Ours**: 自ブランチでは変更なし、他方で変更あり $\implies$ **Theirs 採用** ($O(1)$ スキップ)。
3. **Base == Theirs**: 他方ブランチでは変更なし、自側で変更あり $\implies$ **Ours 採用** ($O(1)$ スキップ)。
4. **Base $\neq$ Ours $\land$ Base $\neq$ Theirs $\land$ Ours $\neq$ Theirs**: 競合・双方向変更の可能性 $\implies$ 直下のエントリを展開し、3-Way 判定ロジックへ移行。

---

# 4. Directory Tree & File Level 3-Way エントリ変更マトリクス

エントリごとの状態変化マトリクスによる判定テーブル:

| Base       | Ours (A)     | Theirs (B)   | 統合結果                     | 判定ロジック                    |
| ---------- | ------------ | ------------ | ---------------------------- | ------------------------------- |
| Unmodified | Modified (A) | Unmodified   | Modified (A)                 | Fast-Path (A 採用)              |
| Unmodified | Unmodified   | Modified (B) | Modified (B)                 | Fast-Path (B 採用)              |
| Unmodified | Modified (A) | Modified (A) | Modified (A)                 | 一致変更 (A 採用)               |
| Unmodified | Modified (A) | Modified (B) | **3-Way Content Merge**      | 内部コンテンツ 3-Way マージ実行 |
| Unmodified | Deleted (A)  | Unmodified   | Deleted                      | A 削除採用                      |
| Unmodified | Unmodified   | Deleted (B)  | Deleted                      | B 削除採用                      |
| Unmodified | Deleted (A)  | Modified (B) | **Conflict (Delete/Modify)** | 削除/変更競合                   |
| Unmodified | Modified (A) | Deleted (B)  | **Conflict (Modify/Delete)** | 変更/削除競合                   |
| Absent     | Added (A)    | Absent       | Added (A)                    | A 追加採用                      |
| Absent     | Absent       | Added (B)    | Added (B)                    | B 追加採用                      |
| Absent     | Added (A)    | Added (B)    | **Conflict (Add/Add)**       | 追加衝突 (CID 不一致の場合)     |

---

# 5. Rust / WASM Core & TypeScript インターフェース

```rust
pub enum MergeResultAction {
    AcceptOurs,
    AcceptTheirs,
    Conflict,
    RecurseSubtree,
}

pub fn evaluate_tree_fastpath(base: &[u8; 32], ours: &[u8; 32], theirs: &[u8; 32]) -> MergeResultAction {
    if ours == theirs {
        return MergeResultAction::AcceptOurs;
    }
    if base == ours {
        return MergeResultAction::AcceptTheirs;
    }
    if base == theirs {
        return MergeResultAction::AcceptOurs;
    }
    MergeResultAction::RecurseSubtree
}
```

```typescript
export interface MergeEntryResultModel {
  path: string;
  status: "clean" | "conflict";
  mergedCid?: Uint8Array;
  conflictDetails?: {
    baseCid?: Uint8Array;
    oursCid?: Uint8Array;
    theirsCid?: Uint8Array;
    reason: "content" | "delete_modify" | "modify_delete" | "add_add";
  };
}

export interface TreeMergeEngineFacade {
  mergeTrees(baseCid: Uint8Array, oursCid: Uint8Array, theirsCid: Uint8Array): AsyncIterableIterator<MergeEntryResultModel>;
}
```
