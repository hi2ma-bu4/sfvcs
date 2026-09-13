# sfvcs 基本設計書: 全体アーキテクチャ・モジュール構造・拡張方針
(`bd-architecture-overview.md`)

---

# 1. 概要とシステム目的

本設計書は、次世代分散バージョン管理システム `sfvcs` (Structure-aware Fast Version Control System) の全体基本設計、サブシステム構成、カプセル化（MVC的関心事の分離）、TypeScript/Node.js CLI & Core API、Rust / WebAssembly (WASM) オフロードエンジン、および拡張アーキテクチャ要件を網羅的・詳細に定義する基本設計書である。

本書は `doc/architecture/sfvcs-design.md` の全セクション（第1節〜第171.14節）および `doc/specs/spec-*.md` の仕様を上位参照し、カプセル化された 4 階層レイヤー構造として具現化する。

---

# 2. サブシステム構成と MVC 的カプセル化境界

`sfvcs` は、責務の混在を防ぎ、他機能の変更が及ぶ影響を最小化するために以下の 4 階層レイヤー構造に分離する。

```
+-----------------------------------------------------------------------+
|                       CLI & Core API Layer                            |
|                 (TypeScript / Node.js Controller)                     |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
|                      Domain Business Service Layer                    |
|             (Merge / Diff Engine / GC / Reflog / Sync Service)        |
+-----------------------------------------------------------------------+
                                   |
         +-------------------------+-------------------------+
         |                                                   |
         v                                                   v
+-----------------------------------+   +-------------------------------+
|      Storage & Model Layer        |   |    WASM Core Engine (Rust)    |
| (Index/Pack/Loose/Virtual Adapter)|   | (CDC/ProllyTree/VCDIFF/Crypto)|
+-----------------------------------+   +-------------------------------+
```

### 2.1 各レイヤーの役割と境界設計
1. **Controller Layer (CLI / Core API)**:
   - ユーザー入力・コマンドライン引数の解析、プログレスバー表示、エラーフォーマットを担当。
   - コアのバージョン管理ロジックやデータストレージの内部表現に直接依存せず、Domain Service の抽象インターフェースのみを呼び出す。
2. **Domain Business Service Layer**:
   - `CommitService`, `MergeService`, `DiffService`, `SyncService`, `GCService` などのビジネスロジック。
   - モデルの状態を変更・操作し、整合性ルールを強制する。
3. **Storage & Model Layer**:
   - オブジェクトモデル (`Chunk`, `SequenceNode`, `FileNode`, `DirectoryNode`, `CommitObject`) の内部構造カプセル化。
   - プラガブルアダプタ (`NodeFsAdapter`, `MemoryAdapter`, `OPFSAdapter`, `IndexedDBAdapter`) を通じた永続化の隠蔽。
4. **WASM Core Engine (Rust / WebAssembly)**:
   - FastCDC, Gear Hash, Prolly Tree 構築, VCDIFF エンコード/デコード, BLAKE3 / SHA-256 ハッシュ計算, AES-SIV / Envelope 暗号化などの計算集約処理をモジュール化してRustで実装し、WASM 境界を介して高速実行する。

---

# 3. TypeScript & Rust/WASM インターフェース境界

```typescript
export interface WasmEngineFacade {
  fastCdcChunk(data: Uint8Array, minSize: number, avgSize: number, maxSize: number): Uint32Array;
  calculateBlake3(data: Uint8Array): Uint8Array;
  vcdiffEncode(source: Uint8Array, target: Uint8Array): Uint8Array;
  vcdiffDecode(source: Uint8Array, delta: Uint8Array): Uint8Array;
  prollyTreeBuild(chunks: Array<{ hash: Uint8Array; length: number }>): Array<{ nodeHash: Uint8Array; level: number }>;
  aesSivEncrypt(key: Uint8Array, plaintext: Uint8Array, assocData: Uint8Array): Uint8Array;
  aesSivDecrypt(key: Uint8Array, ciphertext: Uint8Array, assocData: Uint8Array): Uint8Array;
}
```

---

# 4. 全体アーキテクチャ要件および高度拡張設計のマッピング

| 元仕様節 (`sfvcs-design.md`)                          | 対応基本設計コンポーネント                                                        | 主なカプセル化境界                     |
| ----------------------------------------------------- | --------------------------------------------------------------------------------- | -------------------------------------- |
| 第1節〜第167.10節 (Core Principles & Data Structures) | `bd-architecture-overview.md`, `bd-binary-canonical.md`, `bd-algo-prolly-tree.md` | ドメインデータ分離・Prolly Tree 不変性 |
| 第168節〜第170節 (References)                         | `bd-architecture-overview.md`                                                     | 外部規格・標準化インターフェース       |
| 第171.1節 Bisect & Blame                              | `bd-algo-bisect-blame.md`                                                         | リビジョン探索・著者追跡サービス       |
| 第171.2節 Commit Graph インデックス                   | `bd-storage-commit-graph.md`                                                      | グラフ到達可能性アクセラレータ         |
| 第171.3節 大容量バイナリ資産管理 (LFS)                | `bd-storage-lfs-hooks.md`                                                         | LFS Pointer / Storage 分離             |
| 第171.4節 ゼロ知識・リポジトリ暗号化                  | `bd-virtual-crypto.md`                                                            | Envelope 暗号化アダプタ                |
| 第171.5節 バイナリデルタ圧縮規格 (VCDIFF)             | `bd-algo-submodule-vcdiff-delta.md`                                               | RFC 3284 エンコーダ/デコーダ           |
| 第171.6節 汎用セマンティック 3-Way Merge              | `bd-merge-structural-crdt.md`                                                     | 言語非依存 AST/KV 構造マージ           |
| 第171.7節 ロックフリー並列 GC                         | `bd-storage-gc.md`                                                                | Tri-Color SATB コレクタ                |
| 第171.8節 リアルタイム協調編集                        | `bd-merge-structural-crdt.md`                                                     | Fugue Sequence CRDT アライナー         |
| 第171.9節 Sparse Index & Pathspec Trie                | `bd-storage-index.md`, `bd-algo-sparse-shallow-trie.md`                           | $O(K)$ 境界検索エンジン                |
| 第171.10節 マルチリポジトリ・サブモジュール           | `bd-algo-submodule-vcdiff-delta.md`                                               | サブモジュール再帰コントローラ         |
| 第171.11節 WASM サンドボックス & フック               | `bd-virtual-wasm-plugin.md`                                                       | 安全実行分離サンドボックス             |
| 第171.12節 キーリング管理 & CRL                       | `bd-virtual-webcrypto.md`                                                         | 信頼鍵・失効リスト検証器               |
| 第171.13節 Racy Git & 競合モデル                      | `bd-storage-index.md`, `bd-merge-tree-matrix.md`                                  | 境界同期・アトミック判定器             |
| 第171.14節 パフォーマンス拡張                         | 全 `bd-*.md` モジュール                                                           | Rust/WASM 高速化並列コア               |

---

# 5. カプセル化と疎結合の非破壊原則

1. **仕様の完全網羅**: 既存仕様・設計を一切削減・簡略化せず、全てのデータ構造・アルゴリズム・制御構造を本基本設計に完全に落とし込む。
2. **モデル不変性**: コアオブジェクトモデルは可変状態を直接露出させず、アトミックなビルダーパターンおよびアダプタ経由でのみ状態変更を行う。
3. **環境無依存性**: CLI (Node.js/FS) でも Browser (WASM/IndexedDB/OPFS) でも、同一の Core Domain Service コードが動作する抽象インターフェースを維持する。
