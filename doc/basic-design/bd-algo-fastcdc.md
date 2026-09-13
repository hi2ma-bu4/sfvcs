# sfvcs 基本設計書: Content-Defined Chunking (FastCDC & Gear Hash)
(`bd-algo-fastcdc.md`)

---

# 1. 概要と目的

本設計書は、任意長のバイナリデータストリームから可変長データブロック境界を高速かつ決定論的に抽出する Content-Defined Chunking (CDC) エンジン、特に FastCDC および Gear Hash アルゴリズムの基本設計書である。

本書は `doc/specs/spec-algorithms.md` の第1節（1.1, 1.2, 1.3）および `doc/architecture/sfvcs-design.md` の第167.3〜167.6節の仕様を完全網羅し、カプセル化された Chunking Engine モジュールとして具体的なアルゴリズム表現、擬似コード、パラメータ仕様、SIMD/WASM オフロード構造、および計算複雑性を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Core Algorithm Layer** に属し、ファイルシステムやネットワーク、高レベル状態オブジェクトから完全に隔離された純粋計算エンジンとして設計する。

```
+-----------------------------------------------------------------------+
|               Domain Services (CommitService / Storage)              |
+-----------------------------------------------------------------------+
                                   | Data Stream (Uint8Array)
                                   v
+-----------------------------------------------------------------------+
|              bd-algo-fastcdc (Chunking Service & Engine)              |
|  - TypeScript Wrapper: Buffer Partitioning & Option Normalizer        |
|  - WASM / Rust Core: SIMD-accelerated FastCDC & Gear Hash Loop        |
+-----------------------------------------------------------------------+
                                   | Chunk Boundaries & CIDs
                                   v
+-----------------------------------------------------------------------+
|               ProllyTree Service / Storage Layer                      |
+-----------------------------------------------------------------------+
```

---

# 3. FastCDC パラメータ仕様 & Gear Hash テーブル

### 3.1 既定パラメータ群
- **Target Chunk Size (平均目標値 $S_{\text{avg}}$)**: $16\,\text{KiB} = 16,384\,\text{bytes}$
- **Minimum Chunk Size (最小カットオフ $S_{\text{min}}$)**: $4\,\text{KiB} = 4,096\,\text{bytes}$
- **Maximum Chunk Size (最大限界値 $S_{\text{max}}$)**: $64\,\text{KiB} = 65,536\,\text{bytes}$
- **Window Size**: 48 bytes
- **Gear Hash Table**: 256 個の 64-bit 一様擬似乱数定数表 (`GEAR_TABLE[256]`)。

### 3.2 Gear Hash 定数生成基準
Gear Hash テーブルのエントリ `GEAR_TABLE[0..255]` は、64ビットの一様に分散された乱数定数（明示的シード `0x123456789ABCDEF0` に基づく決定性擬似乱数）で構成され、入力バイト値 $b$ に対して以下のロール演算を適用する。

$$\text{Hash}_{i} = (\text{Hash}_{i-1} \ll 1) + \text{GEAR\_TABLE}[b]$$

---

# 4. 境界判定および Dynamic Normalization アルゴリズム

FastCDC では、固定の境界判定マスクを使用する場合に生じる「境界判定速度の低下」と「チャンクサイズの偏り」を克服するため、二段階の Dynamic Mask (動的マスク) 制御および $S_{\text{min}}$ 境界スキップを適用する。

### 4.1 二段階マスク制御
- **Normalized Normal Mask ($M_{\text{norm}}$)**: `0x0003FFFF00000000ULL` (18-bit ゼロマスク, 期待平均 16KB)
- **Normalized Small Mask ($M_{\text{small}}$)**: `0x00007FFF00000000ULL` (15-bit ゼロマスク, 早期境界検出用)

### 4.2 境界検出処理フロー & Rust / WASM Core 実装

```rust
pub struct ChunkBoundary {
    pub start: usize,
    pub end: usize,
    pub length: usize,
}

pub fn fast_cdc_chunk_stream(data: &[u8], min_size: usize, avg_size: usize, max_size: usize) -> Vec<ChunkBoundary> {
    let mut boundaries = Vec::new();
    let n = data.len();
    let mut offset = 0;

    let mask_small: u64 = 0x0000_7FFF_0000_0000;
    let mask_norm: u64 = 0x0003_FFFF_0000_0000;

    while offset < n {
        let remaining = n - offset;
        if remaining <= min_size {
            boundaries.push(ChunkBoundary {
                start: offset,
                end: n,
                length: remaining,
            });
            break;
        }

        let max_len = std::cmp::min(max_size, remaining);
        let mut fp: u64 = 0;
        let mut i = min_size;

        // Skip bytes below min_size to achieve O(1) minimum performance guarantee
        while i < max_len {
            let b = data[offset + i];
            fp = (fp << 1).wrapping_add(GEAR_TABLE[b as usize]);

            let mask = if i < avg_size { mask_small } else { mask_norm };
            if (fp & mask) == 0 {
                i += 1;
                break;
            }
            i += 1;
        }

        boundaries.push(ChunkBoundary {
            start: offset,
            end: offset + i,
            length: i,
        });
        offset += i;
    }

    boundaries
}
```

---

# 5. エッジケース & 計算複雑性・メモリ管理

1. **小規模データ (< 4 KiB)**:
   - 入力バイト数が $S_{\text{min}}$ 未満の場合は FastCDC ループを実行せず、単一の `ChunkBoundary`（長さ $N$）を出力。
2. **大容量データ (> 1 GiB) ストリーミング**:
   - Web Worker または Node.js ストリームにおいて、WASM メモリ上のリングバッファ (`SharedArrayBuffer` または `WebAssembly.Memory`) を介してメモリ再割り当てなしで 4MB 単位のチャンク分割を実行。
3. **計算複雑性**:
   - 時間複雑性: 最良・平均 $O(N / S_{\text{min}})$, 最悪 $O(N)$。
   - 空間複雑性: 境界オフセット配列の $O(N / S_{\text{avg}})$ メモリ。

---

# 6. TypeScript / WASM アダプタインターフェース

```typescript
export interface FastCdcOptions {
  minSize?: number; // default: 4096
  avgSize?: number; // default: 16384
  maxSize?: number; // default: 65536
}

export interface ChunkBoundaryInfo {
  offset: number;
  length: number;
  hash: Uint8Array;
}

export interface FastCdcEngineFacade {
  chunkBuffer(data: Uint8Array, options?: FastCdcOptions): ChunkBoundaryInfo[];
  chunkStream(stream: ReadableStream<Uint8Array>, options?: FastCdcOptions): AsyncIterableIterator<ChunkBoundaryInfo>;
}
```
