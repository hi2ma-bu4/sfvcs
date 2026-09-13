# sfvcs 基本設計書: パックファイル (.pack/.idx) & Thin Delta 差分探索
(`bd-storage-packfile.md`)

---

# 1. 概要と目的

本設計書は、多数のルーズオブジェクトを単一の圧縮アーカイブに束ねて I/O およびディスク容量を削減する `.pack` ファイル、高速ランダムアクセスのための `.idx` インデックス、および Thin Delta 差分探索アルゴリズムの基本設計書である。

本書は `doc/specs/spec-storage.md` の第4節 (4.1, 4.2), 第16節 (16.1) および `doc/architecture/sfvcs-design.md` の第171.5, 171.14節の仕様を完全網羅し、カプセル化された Packfile Storage Layer モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Storage Core Layer** に属し、上位の Storage Service に対して並行かつ透過的なオブジェクト取得インターフェースを提供する。

```
+-----------------------------------------------------------------------+
|               Object Storage Manager / Storage Service                |
+-----------------------------------------------------------------------+
                                   | Read Object (CID)
                                   v
+-----------------------------------------------------------------------+
|             bd-storage-packfile (Pack Engine Service)                 |
|  - TypeScript: Packfile Reader & Index Search Controller              |
|  - Rust/WASM Core: Binary Pack Index Parser & Delta Inflator Engine   |
+-----------------------------------------------------------------------+
                                   | Read Byte Range
                                   v
+-----------------------------------------------------------------------+
|                   FileSystem / OPFS Storage Layer                     |
+-----------------------------------------------------------------------+
```

---

# 3. パックファイル (.pack) & パックインデックス (.idx) レイアウト

### 3.1 `.pack` ファイル構造
```
+------------------------------------------------------------------------+
| Header: "SFPK" (4B) | Version (u32 LE: 1) | Object Count (u32 LE)      |
+------------------------------------------------------------------------+
| Object Entry 1 | Object Entry 2 | ... | Object Entry N                |
+------------------------------------------------------------------------+
| Checksum: BLAKE3 Hash (32B)                                            |
+------------------------------------------------------------------------+
```

- **Object Entry Layout**: `[Type & Size (Varint)]` + `[Base Offset / Base CID (Thin Delta の場合)]` + `[Compressed Payload (Zstandard / Deflate)]`
- **Thin Delta 制限規則**: デルタチェインの最大深さを `MAX_DELTA_DEPTH = 50` に制限し、循環参照を禁止する。

### 3.2 `.idx` (Version 2) ファイル構造
- **Header**: `\xFFtOc` (4B) + `Version: 2` (4B LE)
- **Fanout Table**: 256 個の 32-bit LE 累積カウントテーブル（第 1 バイトハッシュ値による二分探索高速化）。
- **CID Table**: ソート済み 32-byte CID 配列。
- **CRC32 Table**: チェックサム配列。
- **Offset Table**: 32-bit (または 64-bit LSB) オフセット位置テーブル。

---

# 4. Thin Delta 差分探索アルゴリズム (Pack Clustering & Sliding Window)

パックファイル生成時（`sfvcs repack` / `sfvcs gc`）、圧縮率を最大化するために以下のアルゴリズムで差分基底候補を探索する。

1. **パス & 拡張子ソート**: オブジェクトをファイルパスおよび拡張子順にソート（類似ファイルを集約）。
2. **MinHash / サイズソート**: 同一ウィンドウ内でファイルサイズおよび MinHash 指紋の近い順に整列。
3. **Sliding Window Search**: サイズ 10 以上のスライディングウィンドウを適用し、最も Delta サイズが小さくなるオブジェクトを Base オブジェクトとして決定。

---

# 5. Rust / WASM Core & TypeScript インターフェース

```rust
pub struct PackIndexEntry {
    pub cid: [u8; 32],
    pub crc32: u32,
    pub offset: u64,
}

pub fn search_pack_index(fanout: &[u32; 256], cids: &[[u8; 32]], target_cid: &[u8; 32]) -> Option<usize> {
    let first_byte = target_cid[0] as usize;
    let start = if first_byte == 0 { 0 } else { fanout[first_byte - 1] as usize };
    let end = fanout[first_byte] as usize;
    
    // Binary search within [start..end]
    cids[start..end].binary_search(target_cid).ok().map(|idx| start + idx)
}
```

```typescript
export interface PackObjectHeaderModel {
  type: number;
  size: bigint;
  offset: bigint;
  baseCid?: Uint8Array;
}

export interface PackfileReaderFacade {
  findObjectOffset(cid: Uint8Array): Promise<bigint | null>;
  readObjectData(offset: bigint): Promise<Uint8Array>;
  inflateDelta(baseData: Uint8Array, deltaData: Uint8Array): Uint8Array;
}
```
