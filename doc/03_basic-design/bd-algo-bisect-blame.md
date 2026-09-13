# sfvcs 基本設計書: Bisect & Blame (Token-level Provenance) 追跡
(`bd-algo-bisect-blame.md`)

---

# 1. 概要と目的

本設計書は、コミット履歴 DAG (Directed Acyclic Graph) の二分探索によってバグ混入コミットを最小ステップで迅速特定する Bisect アルゴリズム、複数マージベース (Criss-Cross Merge) における仮想マージベース (Virtual Merge Base) 自動生成アルゴリズム、および Prolly Tree / Winnowing / トークンレベル Meyers Alignment を組み合わせた高度 Blame (Provenance 行・トークン単位著者追跡) アルゴリズムの基本設計書である。

本書は `doc/02_specs/spec-algorithms.md` の第9節 (9.1), 第10節 (10.1), 第18節 (18.1), 第19節 (19.1) および `doc/01_architecture/sfvcs-design.md` の第171.1, 171.14節の仕様を完全網羅し、カプセル化された History Mining Service モジュールとして定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Domain Service (History Analysis) Layer** に属し、複雑な履歴走査・探索ロジックおよびトークンレベルのデータアライメントを完全にカプセル化する。

```
+-----------------------------------------------------------------------+
|             CLI (sfvcs bisect / sfvcs blame) / Web GUI                |
+-----------------------------------------------------------------------+
                                   | Request History Mining
                                   v
+-----------------------------------------------------------------------+
|           bd-algo-bisect-blame (History Mining Service)               |
|  - TypeScript: Graph Navigation & Result Formatting                   |
|  - Rust/WASM Core: DAG Midpoint Evaluator & Token Meyers Engine       |
+-----------------------------------------------------------------------+
                                   | Fetch Commit Graph / Objects
                                   v
+-----------------------------------------------------------------------+
|                 Commit Graph & Object Store Engine                    |
+-----------------------------------------------------------------------+
```

---

# 3. Bisect (二分探索バグ特定) 中点選定アルゴリズム

バグが存在する `bad` コミットと正常な `good` コミットの間の DAG において、候補コミット数を半減させる最適中点 (Mid-point) を以下の評価関数に基づき選定する。

$$C_{\text{mid}} = \arg\min_{c \in \text{Candidates}} \left| \text{Ancestors}(c) - \frac{|\text{Candidates}|}{2} \right|$$

- **マージコミット (Octopus Merge / Criss-Cross Merge)**:
  - 複雑な分岐が存在する場合、Generation Number (世代番号) およびビットマップ到達可能集合を用いて高速評価を行い、探索候補数を正確に半減させるコミットを選定する。

---

# 4. 複数マージベース (Criss-Cross Merge) における仮想マージベース (Virtual Merge Base) 自動生成

両ブランチ間で共通祖先が複数存在する交差マージ (Criss-Cross Merge) において、単一の祖先を選択すると誤ったコンフリクトが発生する問題を解決するため、仮想マージベースを自動合成する。

### 4.1 仮想マージベース生成の手順
1. **Lowermost Common Ancestors (LCA) の抽出**:
   両ブランチ（`Branch A`, `Branch B`）の直近の共通祖先コミット集合 $L = \{C_1, C_2, \dots, C_k\}$ を全探索。
2. **再帰的 3-Way Structural Merge**:
   $|L| > 1$ の場合、祖先コミット群同士を 3-Way Structural Merge して合成ディレクトリツリー $T_{\text{virtual}}$ を構築。
3. **インメモリ Commit Node の生成**:
   合成ツリー $T_{\text{virtual}}$ を参照するインメモリ型仮想コミット $C_{\text{virtual}}$ を生成し、これをマージベースとして本線マージを遂行。

---

# 5. サブ行レベル (Sub-line / Token-level) Provenance 追跡アルゴリズム

単なる行全体の変更追跡にとどまらず、同一行内での変数名変更やリファクタリングを追跡するために「トークンレベル Meyers Alignment」を結合する。

### 5.1 トークン抽出 & アライメントステップ
1. **Prolly Tree Alignment**: Prolly Tree 境界を利用して、巨大ファイルの追跡ブロックを高速アラインメント。
2. **Tokenization (字句解析)**: コードを識別子・演算子・リテラル等のトークン列に分割。
3. **Meyers Token Diff**: トークン列間での Meyers Diff を実行し、トークン単位の著者・コミット provenance を特定。

```
Commit A: let count = calc();  (Author: Alice)
Commit B: let score = calc();  (Author: Bob - changed count -> score)

[Token Line Blame Output]:
"let "       -> Commit A (Alice)
"score"     -> Commit B (Bob)
" = calc();" -> Commit A (Alice)
```

---

# 6. Rust / WASM モジュール & TypeScript インターフェース

```rust
pub struct TokenProvenance {
    pub token: String,
    pub commit_cid: [u8; 33],
    pub author: String,
}

pub struct BlameLine {
    pub line_number: usize,
    pub commit_cid: [u8; 33],
    pub author: String,
    pub timestamp: i64,
    pub line_content: String,
    pub tokens: Vec<TokenProvenance>,
}

pub fn calculate_blame_subline(
    old_content: &str,
    new_content: &str,
    commit_cid: &[u8; 33],
    author: &str,
) -> Vec<TokenProvenance> {
    // Rust WASM Meyers Diff on token streams
    Vec::new()
}
```

```typescript
export interface BlameLineModel {
  lineNumber: number;
  commitCid: Uint8Array;
  author: string;
  timestamp: number;
  lineContent: string;
  tokenProvenance?: Array<{ token: string; commitCid: Uint8Array; author: string }>;
}

export interface HistoryMiningFacade {
  calculateBisectMidpoint(goodCids: Uint8Array[], badCid: Uint8Array): Promise<Uint8Array>;
  generateVirtualMergeBase(branchACid: Uint8Array, branchBCid: Uint8Array): Promise<Uint8Array>;
  blameFile(commitCid: Uint8Array, filepath: string, options?: { sublineTokenLevel?: boolean }): AsyncIterableIterator<BlameLineModel>;
}
```
