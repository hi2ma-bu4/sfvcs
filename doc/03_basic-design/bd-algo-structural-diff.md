# sfvcs 基本設計書: Multi-resolution Structural Diff エンジン
(`bd-algo-structural-diff.md`)

---

# 1. 概要と目的

本設計書は、Prolly Tree 構造を走査し、ノード CID (BLAKE3 ハッシュ) の一致を利用して一致サブツリーの走査を高速スキップする Multi-resolution Structural Diff エンジン、並びに Sequence Alignment / Unified Diff 変換およびマルチバイト安全処理の基本設計書である。

本書は `doc/specs/spec-algorithms.md` の第3節 (3.1, 3.2), 第4節 (4.1), 第15節 (15.1) および `doc/01_architecture/sfvcs-design.md` の第167.7節の仕様を完全網羅し、カプセル化された Diff Service モジュールとして具体的な処理フロー、変換ステップ、Rust/WASM アルゴリズム、および TypeScript インターフェースを定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Domain Service Layer** に属し、上位の CLI `sfvcs diff` コマンド、IDE プラグイン、および REST/GraphQL API コントローラに対して構造化差分および Unified Diff フォーマットを提供する。

```
+-----------------------------------------------------------------------+
|            CLI (sfvcs diff) / IDE Plugin / API Controller             |
+-----------------------------------------------------------------------+
                                   | Request Diff (Old CID, New CID)
                                   v
+-----------------------------------------------------------------------+
|              bd-algo-structural-diff (Diff Service Layer)             |
|  - TypeScript: Diff Service & Encoding Normalization & Formatter       |
|  - Rust/WASM Core: Multi-resolution Fast-Skip & Meyers Diff Alignment |
+-----------------------------------------------------------------------+
                                   | Fetch Objects (SFSQ / SFCK)
                                   v
+-----------------------------------------------------------------------+
|               Object Store / Storage Adapter Layer                    |
+-----------------------------------------------------------------------+
```

---

# 3. 解像度下降比較 (Multi-resolution Fast-Skip) アルゴリズム

本エンジンは、CID によるデータ不変性を活かし、一致するサブツリーを $O(1)$ 時間で完全スキップする。

### 3.1 階層スキップ判定ステップ
1. **Level $L$ (Root ノード比較)**:
   - $A_{\text{root}}.\text{cid} == B_{\text{root}}.\text{cid}$ の場合、全サブツリーが完全に一致するため即座に `DiffResult::Unchanged` を返却する ($O(1)$ スキップ)。
2. **階層下降 (Resolution Descent)**:
   - CID が一致しないノードペアについてのみ、直下の子ノード（Level $L-1$）を展開して比較を行う。
3. **Sequence Alignment**:
   - 変更のある葉ノード (Level 0 Chunk) について、Meyers Diff または Longest Common Subsequence (LCS) アルゴリズムを適用して細部差分を算出。

```
       [Node A1 (CID_A1)] vs [Node B1 (CID_B1)] -> MISMATCH
                       /        \
       (Descent to Level L-1) (Descent to Level L-1)
                     /            \
  [A1_1 (CID_X)] vs [B1_1 (CID_X)] -> MATCH (FAST-SKIP!)
  [A1_2 (CID_Y)] vs [B1_2 (CID_Z)] -> MISMATCH (Descent to Leaf Chunk Diff)
```

---

# 4. Sequence Tree 差分から Unified Diff への変換 & マルチバイト安全処理

### 4.1 変換フロー
1. **Chunk レベル差分抽出**: 変更のあった `SFCK` (Chunk Object) ペアを特定。
2. **文字コード自動判定 (Encoding Detection)**:
   - UTF-8, UTF-16LE, UTF-16BE, Shift_JIS, EUC-JP 等の BOM および統計的文字コード検知器（`chardet`）を適用し、UTF-8 内部表現に統一変換。
3. **マルチバイト安全行分割 (Multibyte-safe Line Splitter)**:
   - Unicode NFC 正規化を維持したまま、サロゲートペアや絵文字、結合文字を分割破壊することなく `\n` または `\r\n` 改行境界で安全に行を抽出。
4. **Unified Diff フォーマット生成**:

```diff
--- a/path/to/file.txt
+++ b/path/to/file.txt
@@ -10,6 +10,7 @@
 context line 1
 context line 2
-deleted line
+inserted line
 context line 3
```

---

# 5. Rust / WASM Core 擬似コード & 計算複雑性

```rust
pub struct DiffHunk {
    pub old_start: usize,
    pub old_lines: usize,
    pub new_start: usize,
    pub new_lines: usize,
    pub lines: Vec<String>,
}

pub fn compare_prolly_nodes(
    old_node: &ProllyNode,
    new_node: &ProllyNode,
    storage: &dyn StorageAdapter,
) -> Vec<DiffHunk> {
    if old_node.cid == new_node.cid {
        return Vec::new(); // Fast-skip
    }

    if old_node.level == 0 && new_node.level == 0 {
        return meyers_diff_align(&old_node.data, &new_node.data);
    }

    // Descend level L-1
    let mut hunks = Vec::new();
    // Recursive child traversal ...
    hunks
}
```

---

# 6. TypeScript インターフェース

```typescript
export interface DiffHunkModel {
  oldStart: number;
  oldLines: number;
  newStart: number;
  newLines: number;
  lines: string[];
}

export interface StructuralDiffFacade {
  compareTrees(oldRootCid: Uint8Array, newRootCid: Uint8Array): AsyncIterableIterator<DiffHunkModel>;
  generateUnifiedDiffText(oldCid: Uint8Array, newCid: Uint8Array, options?: { contextLines?: number }): Promise<string>;
}
```
