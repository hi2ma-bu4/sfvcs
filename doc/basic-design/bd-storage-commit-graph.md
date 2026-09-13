# sfvcs 基本設計書: Commit Graph & Split-Block Bloom Filter (SBF) & FSMonitor 統合
(`bd-storage-commit-graph.md`)

---

# 1. 概要と目的

本設計書は、コミット DAG (Directed Acyclic Graph) の到達可能性探索および Generation Number 判定をミリ秒単位に高速化する Commit Graph (`.sfvcs/objects/info/commit-graph`) バイナリ構造、Split-Block Bloom Filter (SBF)、および OS ファイルシステム監視デーモン (FSMonitor / inotify / FSEvents) 統合の基本設計書である。

本書は `doc/specs/spec-storage.md` の第10節 (10.1, 10.2), 第17節 (17.1), 第20節 (20.1) および `doc/architecture/sfvcs-design.md` の第171.2, 171.9, 171.14節の仕様を完全網羅し、カプセル化された Acceleration & File Watching Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Storage Performance & Monitoring Layer** に位置し、DAG 探索および変更検出のアクセラレータとして機能する。

```
+-----------------------------------------------------------------------+
|              Domain Services (Bisect / Status / Log / Merge)          |
+-----------------------------------------------------------------------+
                                   | Fast DAG Lookup / FS Events
                                   v
+-----------------------------------------------------------------------+
|    bd-storage-commit-graph (Graph & Watcher Acceleration Service)     |
|  - TypeScript: OS Event Backend (inotify/FSEvents/socket) Controller  |
|  - Rust/WASM Core: Commit Graph Reader & SBF Evaluator Engine         |
+-----------------------------------------------------------------------+
                                   | Disk Binary / OS Socket
                                   v
+-----------------------------------------------------------------------+
|                   Storage & OS Event Socket Layer                     |
+-----------------------------------------------------------------------+
```

---

# 3. Commit Graph バイナリレイアウト & Generation Number

### 3.1 バイナリ構造
- **Header**: `CGPH` (4B) + `Version: 1` (1B) + `Hash Version: 1` (1B) + `Chunk Count` (1B)
- **Chunks**:
  - `OIDL` (OID List): ソート済み Commit CID 配列。
  - `OIDF` (OID Fanout): 第 1 バイトインデックス表。
  - `CDAT` (Commit Data): Tree CID, Parent 1 Index, Parent 2 Index, Generation Number (30-bit), Commit Time.
  - `EDGE` (Extra Edge List): 3 つ以上の親を持つ Octopus Merge コミット用親リスト。

### 3.2 Generation Number による到達可能性高速判定
二つのコミット $A, B$ の到達可能性を判定する際、Generation Number $G(A), G(B)$ を用いて即座に判定する。

$$\text{If } G(A) \le G(B) \implies A \text{ cannot be a strict descendant of } B$$

到達可能性探索（`is_ancestor(A, B)`）において、探索キュー内のコミットが $G(A)$ より小さい場合は探索を打ち切ることで、無駄な DAG 走査を 99% 削減する。

---

# 4. Split-Block Bloom Filter (SBF) コミットグラフ高速化

各コミットにおいて変更されたファイルパスの集合をコミットグラフ内に Split-Block Bloom Filter (SBF) として埋め込む。特定のファイルパスに対する `sfvcs log <path>` 実行時、SBF を評価してパスが含まれないコミットを $O(1)$ でスキップする。

---

# 5. OS ファイルシステム監視デーモン (FSMonitor / inotify / FSEvents) 統合

ステータス走査 $O(N)$（全ファイル `stat`）を $O(\text{変更数})$ に高速化する。

1. **プラットフォーム別バックエンド**: Linux (`inotify`), macOS (`FSEvents`), Windows (`ReadDirectoryChangesW`)。
2. **IPC 通信**: UNIX ドメインソケットまたは Named Pipe を用いてバックグラウンド監視デーモンとやり取りし、変更ファイルパス一覧を迅速に取得して `.sfvcs/index` を部分更新する。

---

# 6. Rust / WASM Core & TypeScript インターフェース

```rust
pub fn is_ancestor_generation(gen_a: u32, gen_b: u32) -> bool {
    if gen_a <= gen_b {
        return false;
    }
    true
}
```

```typescript
export interface CommitGraphEvaluatorFacade {
  isAncestor(possibleAncestorCid: Uint8Array, descendantCid: Uint8Array): Promise<boolean>;
  findLca(cidA: Uint8Array, cidB: Uint8Array): Promise<Uint8Array | null>;
}

export interface FsMonitorWatcherFacade {
  startWatching(repoPath: string): void;
  getChangedPathsSince(timestamp: bigint): Promise<string[]>;
}
```
