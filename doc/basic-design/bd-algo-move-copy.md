# sfvcs 基本設計書: Move / Rename / Copy 検出 & Fingerprint 類似度検索
(`bd-algo-move-copy.md`)

---

# 1. 概要と目的

本設計書は、リポジトリツリー変更時におけるファイルの移動 (Move)、名称変更 (Rename)、大文字小文字変更 (Case-only Rename)、および複製 (Copy) を高速かつ高精度に検出するアルゴリズムの基本設計書である。

本書は `doc/specs/spec-algorithms.md` の第5節 (5.1, 5.2), 第6節 (6.1, 6.2, 6.3) および `doc/architecture/sfvcs-design.md` の第167.8, 171.14節の仕様を完全網羅し、カプセル化された Move Detection Service モジュールとして詳細設計を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Domain Service Layer** に位置し、Diff エンジンや Merge エンジンから移動検知リクエストを受け取り、移動追跡結果を共有する独立サービスとして動作する。

```
+-----------------------------------------------------------------------+
|              Diff Service / Merge Service / Unified Diff              |
+-----------------------------------------------------------------------+
                                   | Candidate Deleted & Added Files
                                   v
+-----------------------------------------------------------------------+
|               bd-algo-move-copy (Move Detector Service)               |
|  - TypeScript: Candidate Pair Matching & Score Controller             |
|  - Rust/WASM Core: Winnowing Fingerprint & Subtree Match Engine       |
+-----------------------------------------------------------------------+
                                   | Identified Moves & Confidence Scores
                                   v
+-----------------------------------------------------------------------+
|                     Tree Merge / Output Formatter                     |
+-----------------------------------------------------------------------+
```

---

# 3. Winnowing 近似 Fingerprint 計算 & 小規模ファイルフォールバック

### 3.1 Winnowing 指紋計算ステップ
1. **$k$-gram 生成**: ファイルの内容を $k$-gram ($k = 16$ バイト) のウィンドウでスライド走査し、各 16 バイトに対して 64-bit ハッシュ値を計算。
2. **Sliding Window Selection**: ウィンドウサイズ $w = 32$ のスライディングウィンドウを適用し、各ウィンドウ内の最小ハッシュ値を「指紋 (Fingerprint)」として選出。
3. **Jaccard 類似度**: 削除候補ファイル $A$ と追加候補ファイル $B$ の指紋集合 $F_A, F_B$ に対し、Jaccard 類似度を計算:
   $$J(A, B) = \frac{|F_A \cap F_B|}{|F_A \cup F_B|}$$

### 3.2 小規模ファイル (Short Files < 1 KiB) フォールバック
ファイルサイズが $1\,\text{KiB}$ 未満の場合、Winnowing では指紋数が不十分となるため、BLAKE3 完全ハッシュ一致または文字トークン完全一致率に自動フォールバックする。

---

# 4. $O(1)$ ディレクトリサブツリー一括リネーム & Case-only Rename

### 4.1 $O(1)$ サブツリー移動判定 (Directory Subtree Rename)
ディレクトリエントリ全体の CID (`SFDR` ノードハッシュ) が削除木と追加木の間で完全に一致する場合、配下の全ファイル個別の指紋計算を省略し、$O(1)$ でディレクトリエントリ一括移動と判定する。

### 4.2 大文字小文字のみの変更判定 (Case-only Rename)
パス文字列の Unicode NFC 正規化および Unicode Case-folding（小文字化）を実行し、文字コード長が同一かつ大文字小文字表現のみが異なるケースを別判定フラグ (`is_case_only_rename: true`) として検出する。

---

# 5. 複合信頼度スコア計算 (Composite Confidence Score)

二つのファイル $A, B$ が同質・移動関係にあるかの最終スコア $S(A, B)$ を以下の重み付け計算式により算出する（閾値 $S_{\text{threshold}} = 0.70$ 以上で移動と認定）。

$$S(A, B) = w_1 \cdot J(A, B) + w_2 \cdot \text{PathSimilarity}(A, B) + w_3 \cdot \text{ParentDirectorySimilarity}(A, B) + w_4 \cdot \text{SizeSimilarity}(A, B)$$

- $w_1 = 0.50$ (Content Jaccard Score)
- $w_2 = 0.20$ (Path Distance Score)
- $w_3 = 0.20$ (Parent Folder Score)
- $w_4 = 0.10$ (Size Delta Score)

---

# 6. Rust / WASM モジュール & TypeScript インターフェース

```rust
pub struct MoveCandidate {
    pub path: String,
    pub cid: [u8; 32],
    pub size: u64,
}

pub struct MoveMatchResult {
    pub source_path: String,
    pub target_path: String,
    pub score: f64,
    pub is_subtree_move: bool,
    pub is_case_only_rename: bool,
    pub is_copy: bool,
}

pub fn detect_moves_fast(
    deleted: &[MoveCandidate],
    added: &[MoveCandidate],
    threshold: f64,
) -> Vec<MoveMatchResult> {
    // Rust/WASM parallel winnowing Jaccard estimator
    Vec::new()
}
```

```typescript
export interface MoveMatchInfo {
  sourcePath: string;
  targetPath: string;
  confidenceScore: number;
  isSubtreeMove: boolean;
  isCaseOnlyRename: boolean;
  isCopy: boolean;
}

export interface MoveDetectorFacade {
  detectMoves(deletedFiles: string[], addedFiles: string[], options?: { threshold?: number }): Promise<MoveMatchInfo[]>;
}
```
