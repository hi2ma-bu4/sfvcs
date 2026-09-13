# sfvcs 基本設計書: ガベージコレクション (GC) & Tri-Color Lock-Free Parallel GC
(`bd-storage-gc.md`)

---

# 1. 概要と目的

本設計書は、不要になった孤立（到達不能）オブジェクトをアトミックかつ安全に削除する Mark-and-Sweep GC、到達可能性ビットマップ (Reachability Bitmap) 高速化、並行コミット中の誤削除を防ぐ Grace Period（猶予期間）保護、並行 Tri-Color Marking & SATB Write Barrier アルゴリズム、および GC In-flight Operation Generation Register による並行コミット保護手順の基本設計書である。

本書は `doc/02_specs/spec-storage.md` の第6節 (6.1), 第8節 (8.1), 第12節 (12.1, 12.2), 第18節 (18.1), 第19節 (19.1) および `doc/01_architecture/sfvcs-design.md` の第171.7, 171.14節の仕様を完全網羅し、カプセル化された Garbage Collector Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Domain Maintenance Service Layer** に属し、バックグラウンド定期タスクまたは明示的コマンド (`sfvcs gc`) で安全に起動される。

```
+-----------------------------------------------------------------------+
|                CLI (sfvcs gc) / Background Maintenance Task           |
+-----------------------------------------------------------------------+
                                   | Request GC Execution
                                   v
+-----------------------------------------------------------------------+
|             bd-storage-gc (Garbage Collector Service)                 |
|  - TypeScript: GC Task Scheduler & Lock/Epoch Manager                 |
|  - Rust/WASM Core: Tri-Color Marking Engine & Bitmap Purger           |
+-----------------------------------------------------------------------+
                                   | Bitmap Marking & Atomic Purge
                                   v
+-----------------------------------------------------------------------+
|              Loose Objects & Packfile Storage Layer                   |
+-----------------------------------------------------------------------+
```

---

# 3. Reachability Bitmap & Mark-and-Sweep & Grace Period（猶予期間）保護

### 3.1 到達可能性ビットマップ (Reachability Bitmap) 構造
全オブジェクトの到達可能状態をビット配列（1-bit per object）としてメモリ上に展開し、マーク走査のオーバーヘッドを $O(N)$ に抑える。

### 3.2 Mark-and-Sweep シーケンスステップ
1. **Root Set 収集**: 全ての Reference（`refs/heads/*`, `refs/tags/*`, `HEAD`, `Reflog`）およびインデックス (`.sfvcs/index`) から到達可能な CID を再帰的にマーク（到達可能ビットマップ `ReachabilityBitmap` の生成）。
2. **Grace Period 評価**: マークされなかった孤立オブジェクトのうち、最終アクセス・更新日時（`mtime`）が Grace Period（デフォルト 14 日間）以内のものは削除をスキップ。
3. **Sweep**: 猶予期間を超過した孤立ルーズオブジェクトを物理削除。

---

# 4. ロックフリー並列 GC: Tri-Color Marking & SATB Write Barrier

並行して書き込み（コミットやインデックス更新）が発生している環境下でも、一貫性を保ちつつ並列に GC を実行するため、SATB (Snapshot-At-The-Beginning) 形式の Tri-Color Marking を適用する。

### 4.1 オブジェクト色の定義
- **White (白)**: 未訪問オブジェクト（Sweep 候補）。
- **Grey (灰)**: 訪問済みだが、子ノードの走査が完了していないオブジェクト。
- **Black (黒)**: 自身および全ての子ノードの走査が完了した到達可能オブジェクト。

### 4.2 SATB Write Barrier (書き込みバリア)
GC マーク中に新しいオブジェクトが参照追加された場合、直ちにそのオブジェクトを White から Grey に昇格させ、誤削除を絶対的に防ぐ。

```rust
use std::sync::atomic::{AtomicU8, Ordering};

pub struct AtomicGcState {
    pub is_marking: bool,
}

pub fn satb_write_barrier(obj_cid: &[u8; 33], gc_state: &AtomicGcState, grey_queue: &mut Vec<[u8; 33]>) {
    if gc_state.is_marking {
        grey_queue.push(*obj_cid);
    }
}
```

---

# 5. GC In-flight Operation Generation Register & 並行 Packfile 再構築

### 5.1 GC In-flight Operation Generation Register
現在進行中の並行コミット処理やインデックス操作が生成・参照している世代番号 (Generation Number) をアトミックレジスタに保持し、GC 実行時にこの世代内の生成中オブジェクトを保護する。

### 5.2 Multi-pack-index (MIDX) アトミック置換
古い Packfile 群を再構築（Compaction）する際、アトミックに `multi-pack-index` を生成・置換し、旧 Packfile の参照がゼロになったタイミング（Epoch 終了時）で物理削除を行う。

---

# 6. TypeScript / WASM インターフェース

```typescript
export interface GcOptionsModel {
  gracePeriodMs?: number;
  packReachable?: boolean;
}

export interface GarbageCollectorFacade {
  runGc(options?: GcOptionsModel): Promise<{ purgedObjectsCount: number; freedBytes: bigint }>;
}
```
