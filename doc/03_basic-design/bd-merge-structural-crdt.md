# sfvcs 基本設計書: 汎用言語非依存セマンティック 3-Way Structural Merge & Fugue CRDT リアルタイム協調編集
(`bd-merge-structural-crdt.md`)

---

# 1. 概要と目的

本設計書は、JSON / YAML / Lockfile / TOML などの構造化データに対する言語非依存セマンティック 3-Way Structural Merge、および Fugue Sequence CRDT によるリアルタイム協調編集アライメントの基本設計書である。

本書は `doc/02_specs/spec-merge.md` の第7節 (7.1), 第8節 (8.1) および `doc/01_architecture/sfvcs-design.md` の第171.6, 171.8節の仕様を完全網羅し、カプセル化された Semantic & Realtime Merge Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Domain Service & Advanced Merge Layer** に属し、テキスト行差分では破壊される構造化ドキュメントを AST / Key-Value レベルでマージする。

```
+-----------------------------------------------------------------------+
|                Merge Engine / Realtime Sync Service                   |
+-----------------------------------------------------------------------+
                                   | Request Semantic / CRDT Merge
                                   v
+-----------------------------------------------------------------------+
| bd-merge-structural-crdt (Semantic & CRDT Alignment Engine)           |
|  - TypeScript: JSON/YAML Parser & Key-Value Map Merger Controller     |
|  - Rust/WASM Core: Fugue CRDT Sequence Align & AST Comparator Engine  |
+-----------------------------------------------------------------------+
                                   | Merged Structured Payload
                                   v
+-----------------------------------------------------------------------+
|                    Storage / File Buffer Layer                        |
+-----------------------------------------------------------------------+
```

---

# 3. 汎用言語非依存セマンティック 3-Way Structural Merge (JSON/YAML/Lockfile/TOML)

改行やキーの順序変更による不要なテキストコンフリクトを回避するため、構造化ファイルを AST / Key-Value ツリーにパースしてマージを行う。

### 3.1 キーバリューマップ (Key-Value Map) マージルール
1. **キーの順序非依存性**: JSON / YAML 内の Map のキー順序変更は差分とみなさず無衝突でマージ。
2. **オブジェクトプロパティの独立変更**:
   - Base: `{"a": 1, "b": 2}`
   - Ours: `{"a": 10, "b": 2}`
   - Theirs: `{"a": 1, "b": 20}`
   - **Merged Output**: `{"a": 10, "b": 20}` (自動競合解消)
3. **キーの削除と更新の競合**:
   - 一方がキーの値を更新し、他方がそのキーを削除した場合は、構造化コンフリクトファイル `.sfvcs/merge-state` に記録。
4. **配列 (Array) の 3-Way マージ**:
   配列要素に対しては Sequence Tree Alignment または Fugue CRDT を適用して行・要素の順序整合性を維持。

---

# 4. Fugue Sequence CRDT リアルタイム協調編集アライメント

複数ユーザーや分散環境での並行編集時に、インターリーブ（文字・要素の交互混入バグ）を防止する最新の Fugue Sequence CRDT アライメントを統合する。

- **Fugue 原理**: 各文字/ノードの挿入時に、親ノードの ID と祖先ツリーの深さを関連付け、並列挿入されたシーケンスを非インターリーブで決定論的に決定。

---

# 5. Rust / WASM Core & TypeScript インターフェース

```rust
pub fn align_fugue_sequence(base: &[u8], ours: &[u8], theirs: &[u8]) -> Vec<u8> {
    // Non-interleaving Fugue CRDT alignment engine in Rust WASM
    Vec::new()
}
```

```typescript
export interface SemanticMergeFacade {
  mergeJson(baseStr: string, oursStr: string, theirsStr: string): { mergedContent: string; isClean: boolean };
  mergeYaml(baseStr: string, oursStr: string, theirsStr: string): { mergedContent: string; isClean: boolean };
}

export interface FugueCrdtFacade {
  alignSequence<T>(baseSeq: T[], oursSeq: T[], theirsSeq: T[]): T[];
}
```
