# sfvcs 基本設計書: Sparse Checkout / Shallow History / Pathspec Trie 検索
(`bd-algo-sparse-shallow-trie.md`)

---

# 1. 概要と目的

本設計書は、大容量リポジトリの一部のみを効率的にローカル展開する Sparse Checkout 境界判定、コミット履歴の取得深度を制御する Shallow History 境界判定・`fsck` 検証、およびファイルパス検索を $O(K)$ で高速化する Pathspec Trie 検索の基本設計書である。

本書は `doc/specs/spec-algorithms.md` の第7節 (7.1), 第8節 (8.1, 8.2), 第11節 (11.1) および `doc/01_architecture/sfvcs-design.md` の第171.9, 171.10, 171.14節の仕様を完全網羅し、カプセル化された Filtering & Navigation Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Domain Service & Navigation Layer** に属し、ワーキングツリー展開、オブジェクト取得、CLI パス検索時のフィルタリングを一括管理する。

```
+-----------------------------------------------------------------------+
|               CLI (status / checkout / fetch) / WorkingTree           |
+-----------------------------------------------------------------------+
                                   | Request Path Match / Shallow Check
                                   v
+-----------------------------------------------------------------------+
|        bd-algo-sparse-shallow-trie (Filter & Navigation Service)      |
|  - TypeScript: Pathspec Pattern State & Shallow Boundary Evaluator    |
|  - Rust/WASM Core: Radix Trie ($O(K)$) & Sparse Boundary Engine      |
+-----------------------------------------------------------------------+
                                   | Filtered Paths / Boundary Status
                                   v
+-----------------------------------------------------------------------+
|                 Working Tree / Object Storage Layer                   |
+-----------------------------------------------------------------------+
```

---

# 3. Pathspec Trie ($O(K)$ プレフィックスツリー) 検索アルゴリズム

パス文字列のパターンマッチングにおいて、従来の全ファイル走査 $O(N \cdot K)$ を回避し、パス文字長 $K$ に依存する Radix Trie (Trie) を用いて $O(K)$ 走査を実現する。

### 3.1 Radix Trie ノード表現 (Rust / WASM Core)
```rust
use std::collections::HashMap;

pub struct PathspecTrieNode {
    pub segment: String,
    pub is_terminal: bool,
    pub is_wildcard: bool,
    pub children: HashMap<String, PathspecTrieNode>,
}

impl PathspecTrieNode {
    pub fn new(segment: &str) -> Self {
        Self {
            segment: segment.to_string(),
            is_terminal: false,
            is_wildcard: false,
            children: HashMap::new(),
        }
    }

    pub fn insert_path(&mut self, path: &str) {
        let segments: Vec<&str> = path.split('/').filter(|s| !s.is_empty()).collect();
        let mut current = self;
        for seg in segments {
            current = current.children.entry(seg.to_string()).or_insert_with(|| PathspecTrieNode::new(seg));
        }
        current.is_terminal = true;
    }

    pub fn matches_prefix(&self, path: &str) -> bool {
        let segments: Vec<&str> = path.split('/').filter(|s| !s.is_empty()).collect();
        let mut current = self;
        for seg in segments {
            if let Some(next_node) = current.children.get(seg) {
                current = next_node;
            } else if current.children.contains_key("*") {
                return true;
            } else {
                return false;
            }
        }
        true
    }
}
```

---

# 4. Sparse Checkout (部分チェックアウト) 境界判定ルール

### 4.1 境界判定ルール
`.sfvcs/sparse-checkout` に定義されたパターンに合致しないディレクトリツリーに差し掛かった場合、`SFDR` (Directory Node) は読み込むが配下の `SFFL` / `SFSQ` / `SFCK` オブジェクトのダウンロードおよびワーキングツリーへの展開をスキップする。

---

# 5. Shallow History (浅い履歴 clone) 境界判定および `fsck` 検証

### 5.1 Shallow Boundary 構造
`.sfvcs/shallow` ファイルに記録された Shallow Commit Hash List を「浅い履歴の境界 (Shallow Roots)」として保持する。

### 5.2 Shallow `fsck` 検証ルール
1. Shallow Commit の親コミットが存在しない場合でも、`fsck` は整合性エラー（Missing Parent Commit Error）を出力せず、`Shallow Boundary Reached` として正常終了する。
2. ネットワーク経由での追加コミット取得時（`fetch --unshallow` など）、境界が自動的に更新・解除される。

---

# 6. TypeScript / WASM インターフェース

```typescript
export interface PathspecMatcherFacade {
  buildTrie(patterns: string[]): void;
  matchPath(path: string): boolean;
  matchDirectoryPrefix(dirPath: string): boolean;
}

export interface ShallowManagerFacade {
  isShallowRoot(commitCid: Uint8Array): boolean;
  verifyShallowGraphIntegrity(): Promise<boolean>;
  updateShallowRoots(newShallowCids: Uint8Array[]): Promise<void>;
}
```
