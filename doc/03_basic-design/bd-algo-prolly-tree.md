# sfvcs 基本設計書: Prolly Tree (Sequence Tree) 構築 & Dual-Mask / 偏り保護
(`bd-algo-prolly-tree.md`)

---

# 1. 概要と目的

本設計書は、可変長シーケンスデータおよび順序付きキーバリューエントリ（ファイルノード、ディレクトリエントリ）を決定論的に階層化ツリー構造に変換し、差分検索および 3-Way Structural Merge を高速化する Prolly Tree (Sequence Tree) 構築アルゴリズム、病的入力フォールバック、および Dual-Mask Normalization 制御の基本設計書である。

本書は `doc/02_specs/spec-algorithms.md` の第2節 (2.1, 2.2), 第13節 (13.1), 第16節 (16.1) および `doc/01_architecture/sfvcs-design.md` の第167.1〜167.10, 171.14節を完全網羅し、カプセル化された Sequence Tree Builder モジュールとして定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Core Model & Data Structure Layer** に位置し、上位の File Structure (SFFL/SFSQ) や Directory Structure (SFDR)、マージエンジンに対して完全不変（Immutable）な抽象木構造ノードを提供する。

```
+-----------------------------------------------------------------------+
|             File Service / Directory Service / Merge Engine           |
+-----------------------------------------------------------------------+
                                   | Chunks / Directory Entries
                                   v
+-----------------------------------------------------------------------+
|            bd-algo-prolly-tree (ProllyTree Builder Service)           |
|  - TypeScript: ProllyTreeNode Model & Cursor Iterator                 |
|  - Rust/WASM Core: Bottom-up Streaming Multi-level Tree Builder       |
+-----------------------------------------------------------------------+
                                   | Root Sequence Node CID (SFSQ)
                                   v
+-----------------------------------------------------------------------+
|                  Object Store / Storage Adapter                       |
+-----------------------------------------------------------------------+
```

---

# 3. 内部ノードの境界判定ルール & Dual-Mask Normalization

### 3.1 階層ごとの境界判定
Prolly Tree は、ボトムアップストリーミング方式で各レベル $L \ge 0$ のエントリハッシュ値を判定し、境界条件を満たす位置でノードを決定論的に分割する。
- 期待ファンアウト (Fan-out $B$): **$B = 64$** (平均 64 個のエントリごとに上位 Level $L+1$ の親ノードを作成)

### 3.2 Dual-Mask Normalization (ハッシュ偏り保護)
ハッシュ関数の病的偏りや攻撃者による人工的ハッシュ（Hash Collision Attack / Degenerate Pattern）を防ぐため、プライマリマスク $M_1$ とセカンダリマスク $M_2$ を用いた二重マスク判定を行う。

$$\text{BoundaryCondition}(H, L) = ((H \mathbin{\&} M_1(L)) == 0) \lor ((H \mathbin{\&} M_2(L)) == M_2(L))$$

- **プライマリマスク $M_1(L)$**: ファンアウト $B=64$ に相当する 下位 6-bit マスク (`0x0000003F`)。
- **セカンダリマスク $M_2(L)$**: 代替パターン検出用の補正マスク (`0x00000FC0`)。

---

# 4. 病的入力フォールバック & 最大深度制限 `MAX_TREE_DEPTH = 32`

ノードの分割が発生せずに木が無限に深くなる現象（Degenerate Tree）からシステムを保護するため、厳格な限界制御ルールを適用する。

### 4.1 制御パラメータ
1. **最大木深度**: `MAX_TREE_DEPTH = 32`
2. **単一ノード最大エントリ数**: `MAX_NODE_ENTRIES = 1024`
3. **強制分割ルールのトリガー (Hard Splitting Rules)**:
   - 木の深さが 32 に達した場合、または単一ノードのエントリ数が 1024 に達した場合、ハッシュ条件に関わらず即座に強制分割（Hard Splitting）を実行し、ノードを出力する。

---

# 5. ボトムアップ・ストリーミング木構築アルゴリズム (Rust / WASM Core)

```rust
pub struct ProllyNodeEntry {
    pub key: Vec<u8>,
    pub cid: [u8; 33],
    pub length: u64,
    pub boundary_hash: [u8; 32],
}

pub struct ProllyBuilder {
    levels: Vec<Vec<ProllyNodeEntry>>,
}

impl ProllyBuilder {
    pub fn new() -> Self {
        Self { levels: Vec::new() }
    }

    pub fn push_entry(&mut self, entry: ProllyNodeEntry) -> Result<(), &'static str> {
        let mut current = entry;
        let mut level = 0;

        loop {
            if level >= 32 {
                // Force hard split to prevent infinite depth explosion
                self.flush_level_at(level)?;
                break;
            }

            if level >= self.levels.len() {
                self.levels.push(Vec::new());
            }

            self.levels[level].push(current.clone());

            let is_boundary = self.eval_boundary(&current.boundary_hash, level);
            let is_full = self.levels[level].len() >= 1024;

            if is_boundary || is_full {
                let parent_entry = self.flush_level_at(level)?;
                current = parent_entry;
                level += 1;
            } else {
                break;
            }
        }
        Ok(())
    }

    fn eval_boundary(&self, hash: &[u8; 32], level: usize) -> bool {
        let val = u32::from_le_bytes([hash[0], hash[1], hash[2], hash[3]]);
        let m1: u32 = 0x0000003F;
        let m2: u32 = 0x00000FC0;
        ((val & m1) == 0) || ((val & m2) == m2)
    }

    fn flush_level_at(&mut self, level: usize) -> Result<ProllyNodeEntry, &'static str> {
        // Build SFSQ object for the level entries and return parent NodeEntry
        unimplemented!()
    }
}
```

---

# 6. TypeScript インターフェース

```typescript
export interface ProllyNodeEntryModel {
  key?: Uint8Array;
  cid: Uint8Array;
  length: bigint;
  boundaryHash: Uint8Array;
}

export interface ProllyTreeCursor {
  getCurrentKey(): Uint8Array;
  getCurrentCid(): Uint8Array;
  seekTo(key: Uint8Array): Promise<boolean>;
  next(): Promise<boolean>;
}

export interface ProllyTreeFacade {
  buildTree(entries: ProllyNodeEntryModel[]): Promise<Uint8Array>; // Returns Root SFSQ CID
  openCursor(rootCid: Uint8Array): Promise<ProllyTreeCursor>;
}
```
