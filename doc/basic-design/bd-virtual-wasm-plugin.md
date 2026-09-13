# sfvcs 基本設計書: WASM サンドボックス & Web Worker ストリーミング & Quota Eviction
(`bd-virtual-wasm-plugin.md`)

---

# 1. 概要と目的

本設計書は、信頼できないサードパーティプラグインやサーバーフックを安全に分離実行する WebAssembly サンドボックスアーキテクチャ、ブラウザ環境での Web Worker ストリーミング処理、および IndexedDB / OPFS の Storage Quota Management & LRU Eviction の基本設計書である。

本書は `doc/specs/spec-virtual-vcs.md` の第4.1, 4.2節, 第5.1, 5.2節, 第8節, 第11節および `doc/architecture/sfvcs-design.md` の第171.11節の仕様を完全網羅し、カプセル化された Plugin Sandbox & Memory Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Execution Environment & Plugin Isolation Layer** に位置し、WASM プラグインおよび Worker メモリ管理を担当する。

```
+-----------------------------------------------------------------------+
|               CLI / Extension / Web VCS Client Application            |
+-----------------------------------------------------------------------+
                                   | Execute Plugin Hook / Worker Job
                                   v
+-----------------------------------------------------------------------+
|    bd-virtual-wasm-plugin (Sandbox & Worker Execution Engine)         |
|  - TypeScript: WASI Sandbox Host & Web Worker Pool Controller         |
|  - Rust/WASM Core: Pluggable Host Imports & LRU Quota Monitor Engine  |
+-----------------------------------------------------------------------+
                                   | Sandboxed WASM Memory / Worker Channel
                                   v
+-----------------------------------------------------------------------+
|                  WASI Runtime / Web Workers Layer                     |
+-----------------------------------------------------------------------+
```

---

# 3. WebAssembly サンドボックスプラグインアーキテクチャ (WASI)

WASM プラグインはホスト環境のメモリやファイルシステムに直接アクセスできない分離サンドボックス空間で動作させる。

1. **WASI 抽象化**: プラグインからの I/O 要求はすべてホスト側フック関数（Host Imports）を介して厳格に認可・制御する。
2. **メモリ・CPU 上限保護**: WASM モジュールの最大使用可能メモリ領域を制限（例: 256MiB）し、無限ループ対策として命令カウンタ（Fuel Limit）を設定する。

---

# 4. Web Worker ストリーミング処理 & OPFS / IndexedDB Chunked Handling

ブラウザメインスレッドの UI フリーズを完全に回避するため、大容量処理を Web Worker バックグラウンドストリームにオフロードする。

- **Chunked Handling**: 100MB 以上の巨大アセット処理時、データを 4MB 単位の Chunk に分割し、`ReadableStream` / `WritableStream` を用いて非同期ストリーミングパイプラインで処理する。

---

# 5. IndexedDB / OPFS Storage Quota Management & LRU Eviction

ブラウザのストレージ容量制限（Storage Quota Exceeded Error）を回避するため、LRU (Least Recently Used) キャッシュ削除機構を構築する。

1. **Storage Estimate 監視**: `navigator.storage.estimate()` を呼び出し、空き容量が 10% 未満になった場合にトリガー。
2. **LRU Eviction**: パックファイルキャッシュや古いコミットツリーのうち、アクセス日時の最も古い一次キャッシュオブジェクトを優先削除する（ローカルの Uncommitted データや Refs は保護）。

---

# 6. TypeScript インターフェース

```typescript
export interface WasmSandboxOptionsModel {
  maxMemoryMb?: number;
  fuelLimit?: bigint;
  allowedCapabilities?: string[];
}

export interface WasmPluginInstanceFacade {
  callExport(functionName: string, inputData: Uint8Array): Promise<Uint8Array>;
  terminate(): void;
}

export interface StorageQuotaManagerFacade {
  checkQuotaUsage(): Promise<{ usedBytes: bigint; totalBytes: bigint }>;
  evictLruCache(targetFreeBytes: bigint): Promise<bigint>;
}
```
