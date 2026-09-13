# sfvcs 基本設計書: Bisect & Blame (Token-level Provenance) 追跡
(`bd-algo-bisect-blame.md`)

---

# 1. 概要と目的

本設計書は、コミット履歴 DAG (Directed Acyclic Graph) の二分探索によってバグ混入コミットを最小ステップで迅速特定する Bisect アルゴリズム、および Prolly Tree / Winnowing / トークンレベル Meyers Alignment を組み合わせた高度 Blame (Provenance 行・トークン単位著者追跡) アルゴリズムの基本設計書である。

本書は `doc/specs/spec-algorithms.md` の第9節 (9.1), 第10節 (10.1), 第18節 (18.1) および `doc/architecture/sfvcs-design.md` の第171.1, 171.14節の仕様を完全網羅し、カプセル化された History Mining Service モジュールとして定義する。

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

# 4. サブ行レベル (Sub-line / Token-level) Provenance 追跡アルゴリズム

単なる行全体の変更追跡にとどまらず、同一行内での変数名変更やリファクタリングを追跡するために「トークンレベル Meyers Alignment」を結合する。

### 4.1 トークン抽出 & アライメントステップ
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

# 5. Rust / WASM モジュール & TypeScript インターフェース

```rust
pub struct TokenProvenance {
    pub token: String,
    pub commit_cid: [u8; 32],
    pub author: String,
}

pub struct BlameLine {
    pub line_number: usize,
    pub commit_cid: [u8; 32],
    pub author: String,
    pub timestamp: i64,
    pub line_content: String,
    pub tokens: Vec<TokenProvenance>,
}

pub fn calculate_blame_subline(
    old_content: &str,
    new_content: &str,
    commit_cid: &[u8; 32],
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
  blameFile(commitCid: Uint8Array, filepath: string, options?: { sublineTokenLevel?: boolean }): AsyncIterableIterator<BlameLineModel>;
}
```
