# sfvcs 基本設計書: 設定ファイル (`.sfvcsconfig`) ・ `.sfvcsignore` ・ 属性 (`.sfvcsattributes`)
(`bd-config-ignore-attributes.md`)

---

# 1. 概要と目的

本設計書は、リポジトリ設定 (`.sfvcsconfig`)、除外パターン評価行列 (`.sfvcsignore` / `.gitignore` 継承)、およびファイル属性設定 (`.sfvcsattributes`) の評価・適用の基本設計書である。

本書は `doc/specs/spec-config-and-attributes.md` の全セクション（第1, 2, 3節）および `doc/01_architecture/sfvcs-design.md` の関連仕様を完全網羅し、カプセル化された Configuration & Pattern Evaluation Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Configuration & Pattern Evaluator Layer** に位置し、StatusService や WorkingTree 展開時にパターン評価と設定値を提供する。

```
+-----------------------------------------------------------------------+
|               StatusService / Checkout / Commit / Storage Layer       |
+-----------------------------------------------------------------------+
                                   | Query Ignore / Attributes / Config
                                   v
+-----------------------------------------------------------------------+
|   bd-config-ignore-attributes (Pattern & Config Service)              |
|  - TypeScript: Config Parser & Precedence Evaluator                   |
|  - Rust/WASM Core: High-speed Glob Pattern Matcher Engine             |
+-----------------------------------------------------------------------+
                                   | Read Config / Ignore Files
                                   v
+-----------------------------------------------------------------------+
|                 Physical / Virtual FS Storage Layer                   |
+-----------------------------------------------------------------------+
```

---

# 3. リポジトリ設定 (`.sfvcsconfig`) & `.gitignore` 継承

### 3.1 設定項目一覧
- `core.autocrlf`: `true` | `false` | `input`
- `core.safecrlf`: `true` | `false` | `warn`
- `core.quotepath`: `true` | `false` (Unicode NFC パス文字の引用符制御)
- `core.inheritGitignore`: `true` | `false` (Git の `.gitignore` ファイルの自動継承評価)
- `algo.chunker`: `fastcdc` | `fixed`
- `algo.hash`: `blake3` | `sha256`

---

# 4. 除外パターン評価行列 (Precedence Evaluation Matrix) & 否定パターン (`!`)

### 4.1 評価優先順位行列 (高 $\to$ 低)
1. CLI で直接渡された引数パターン。
2. 対象ファイルの配置ディレクトリからリポジトリルートに向かって最深部から走査した `.sfvcsignore` 内の否定パターン (`!pattern`)。
3. 同上の最深部からの標準除外パターン (`pattern`)。
4. `core.inheritGitignore = true` の場合の `.gitignore` パターン。
5. グローバル除外設定 (`~/.sfvcsignore`)。

### 4.2 親ディレクトリ除外の優先ルール
親ディレクトリが除外されている場合、配下のファイルに否定パターン `!parent/file.txt` が指定されていても親ディレクトリ自体がスキップされるため、親ディレクトリの再帰スキップ判定を厳格に行う。

---

# 5. ファイル属性設定 (`.sfvcsattributes`)

ファイルパスごとの処理属性を制御する。
- `*.png filter=lfs` (LFS フィルター適用)
- `*.json merge=json-semantic` (構造化 JSON マージ適用)
- `*.secret cipher=envelope` (クライアントサイド暗号化)

---

# 6. Rust / WASM Core & TypeScript インターフェース

```rust
pub fn match_glob_pattern(pattern: &str, path: &str, is_dir: bool) -> bool {
    // High-speed glob matching engine in Rust WASM
    false
}
```

```typescript
export interface ConfigValuesModel {
  autocrlf: boolean;
  inheritGitignore: boolean;
  hashAlgorithm: "blake3" | "sha256";
}

export interface PatternEvaluatorFacade {
  isIgnored(path: string, isDirectory: boolean): boolean;
  getAttributes(path: string): Record<string, string | boolean>;
}
```
