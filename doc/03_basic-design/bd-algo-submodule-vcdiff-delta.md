# sfvcs 基本設計書: サブモジュール操作 & 循環参照防止 & 差分基底 / VCDIFF アルゴリズム
(`bd-algo-submodule-vcdiff-delta.md`)

---

# 1. 概要と目的

本設計書は、ネストされたサブモジュール (`ENTRY_SUBMODULE`) の再帰的操作・循環参照検出防止アルゴリズム、MinHash / SuperMinHash スケッチによる Packfile 差分基底 (Thin Delta Base) 高速選定アルゴリズム、および VCDIFF (RFC 3284) メモリフットプリント制限付きストリーミングエンコード/デコードアルゴリズムの基本設計書である。

本書は `doc/02_specs/spec-algorithms.md` の第12節 (12.1), 第14節 (14.1), 第17節 (17.1), 第20節 (20.1) および `doc/01_architecture/sfvcs-design.md` の第171.5, 171.10節の仕様を完全網羅し、カプセル化された Submodule & Pack Optimization Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Domain Service & Pack Optimization Layer** に属し、外部リポジトリ連携およびデータ圧縮の基盤を担当する。

```
+-----------------------------------------------------------------------+
|              CLI (submodule / repack) / Pack Building Service         |
+-----------------------------------------------------------------------+
                                   | Request Submodule Op / Delta Build
                                   v
+-----------------------------------------------------------------------+
|    bd-algo-submodule-vcdiff-delta (Pack & Submodule Service)          |
|  - TypeScript: Submodule Cycle Detector Stack & Stream Pipe           |
|  - Rust/WASM Core: MinHash Jaccard Estimator & VCDIFF Engine (RFC 3284)|
+-----------------------------------------------------------------------+
                                   | Submodule State / Delta Bytes
                                   v
+-----------------------------------------------------------------------+
|                  Storage Engine / Wire Protocol Layer                 |
+-----------------------------------------------------------------------+
```

---

# 3. サブモジュール循環参照検出・防止アルゴリズム (Submodule Cyclic Dependency Detection)

ネストされたサブモジュール操作（`sfvcs submodule update --recursive`）において、リポジトリ参照が循環（$R_A \to R_B \to R_C \to R_A$）している場合の無限ループおよびスタックオーバーフローを防ぐ。

### 3.1 循環検出アルゴリズムデータ構造および探索フロー
```rust
pub struct SubmoduleGraphNode {
    pub repo_uuid_or_canonical_path: String,
    pub commit_cid: [u8; 33],
}

pub struct SubmoduleCycleDetector {
    visited_stack: Vec<SubmoduleGraphNode>,
}

impl SubmoduleCycleDetector {
    pub fn enter_submodule(&mut self, node: SubmoduleGraphNode) -> Result<(), SubmoduleCycleError> {
        if self.visited_stack.iter().any(|v| v.repo_uuid_or_canonical_path == node.repo_uuid_or_canonical_path) {
            return Err(SubmoduleCycleError::CyclicDependencyDetected {
                cycle_path: self.visited_stack.iter().map(|n| n.repo_uuid_or_canonical_path.clone()).collect(),
                conflict_node: node.repo_uuid_or_canonical_path,
            });
        }
        self.visited_stack.push(node);
        Ok(())
    }

    pub fn leave_submodule(&mut self) {
        self.visited_stack.pop();
    }
}
```

---

# 4. MinHash / SuperMinHash に基づく Thin Delta Base 高速選定

Packfile 圧縮時に、全オブジェクト対比較 $O(N^2)$ を回避するため、MinHash / SuperMinHash スケッチを用いた Jaccard 類似度予測を行う。

1. **$k$-gram 分割とスケッチ算定**:
   オブジェクトの $k$-gram 集合から 64 個の独立ハッシュ関数により MinHash スケッチ $H(O)$ を算出。SuperMinHash でスケッチ空間を圧縮。
2. **Jaccard 類似度判定**:
   $$J(A, B) \approx \frac{|\{i \mid H_i(A) = H_i(B)\}|}{K}$$
3. **高速候補選定**:
   ハッシュ値およびパスの類似度でソートされたスライディングウィンドウ内で、最も MinHash 類似度が高いオブジェクトを Thin Delta Base として選出 ($O(N \log N)$)。

---

# 5. VCDIFF メモリフットプリント制限付きストリーミングエンコード/デコード (RFC 3284)

巨大バイナリファイルに対してもメモリ上限（例: 64MiB）を超えずに VCDIFF (RFC 3284) を適用する。

### 5.1 チャンクドスライディングウィンドウ
- **Target Data Windowing**: 16MiB 単位のブロックに分割して VCDIFF ウィンドウを作成。
- **Instruction Execution**: `ADD`, `RUN`, `COPY` 命令を生成。`COPY` 命令の探索範囲をスライディングウィンドウ（Address Cache: `NEAR` / `SAME`）内に制限し、メモリフットプリントを一定に維持。

---

# 6. Rust / WASM Core & TypeScript インターフェース

```rust
pub struct VcdiffWindow {
    pub source_offset: u64,
    pub source_length: u64,
    pub target_length: u64,
    pub instructions: Vec<u8>,
}

pub fn vcdiff_encode_window(source: &[u8], target: &[u8]) -> Vec<u8> {
    // RFC 3284 VCDIFF encoding logic in Rust WASM
    Vec::new()
}
```

```typescript
export interface SubmoduleCycleGuardFacade {
  pushRepositoryPath(repoPath: string, commitCid: Uint8Array): void;
  popRepositoryPath(): void;
  assertNoCycle(targetPath: string): void;
}

export interface VcdiffFacade {
  encodeStream(source: ReadableStream<Uint8Array>, target: ReadableStream<Uint8Array>): ReadableStream<Uint8Array>;
  decodeStream(source: ReadableStream<Uint8Array>, delta: ReadableStream<Uint8Array>): ReadableStream<Uint8Array>;
}
```
