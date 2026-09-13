# sfvcs 基本設計書: ワーキングツリー状態キャッシュ (`.sfvcs/index`) & Racy Git 回避
(`bd-storage-index.md`)

---

# 1. 概要と目的

本設計書は、ワーキングツリーとオブジェクトストレージ間の高速ミリ秒単位同期・状態キャッシュを司る `.sfvcs/index` ファイルのバイナリ構造、Sparse Index（部分チェックアウト用集約インデックス）、および Racy Git 問題の回避アルゴリズムの基本設計書である。

本書は `doc/specs/spec-storage.md` の第2節 (2.1), 第13節 (13.1), 第15節 (15.1) および `doc/01_architecture/sfvcs-design.md` の第171.9, 171.13, 171.14節の仕様を完全網羅し、カプセル化された Index Manager モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Storage Cache & State Layer** に位置し、ステータス走査 (`sfvcs status`) やコミット処理 (`sfvcs commit`) の際に高速なインメモリキャッシュ兼アトミックディスク状態を提供する。

```
+-----------------------------------------------------------------------+
|               CLI (status / add / commit) / Core API                  |
+-----------------------------------------------------------------------+
                                   | Read / Mutate Index State
                                   v
+-----------------------------------------------------------------------+
|             bd-storage-index (Index Manager Service)                  |
|  - TypeScript: Index File Transaction & Cache State Manager           |
|  - Rust/WASM Core: Binary Parser / Serializer & Racy Checker Engine   |
+-----------------------------------------------------------------------+
                                   | Stat / File Hash Verification
                                   v
+-----------------------------------------------------------------------+
|                     Physical / Virtual FS Layer                       |
+-----------------------------------------------------------------------+
```

---

# 3. インデックス・バイナリレイアウト仕様 (`.sfvcs/index`)

`.sfvcs/index` ファイルは、決定論的な可変長バイナリフォーマットで構成される。

```
+------------------------------------------------------------------------+
| Header: "SFIX" (4B) | Version (u32 LE: 1) | Entry Count (u32 LE)       |
+------------------------------------------------------------------------+
| Entry 1: [ctime (12B)] [mtime (12B)] [stat (32B)] [CID (32B)]          |
|          [flags (u16 LE)] [path len (Varint)] [path (UTF-8 NFC)]       |
| Entry 2: ...                                                           |
+------------------------------------------------------------------------+
| Extensions (Sparse Index / Split Index)                                |
+------------------------------------------------------------------------+
| Checksum: BLAKE3 Hash (32B)                                            |
+------------------------------------------------------------------------+
```

### 3.1 インデックスエントリ構造 (`IndexEntry`)
- `ctime_sec` (i64 LE), `ctime_nsec` (u32 LE): ファイル作成タイムスタンプ。
- `mtime_sec` (i64 LE), `mtime_nsec` (u32 LE): 最終更新タイムスタンプ。
- `dev` (u32 LE), `ino` (u64 LE), `mode` (u32 LE), `uid` (u32 LE), `gid` (u32 LE), `file_size` (u64 LE): ファイルシステムメタデータ。
- `cid` (32B BLAKE3): ファイルコンテンツのオブジェクト CID。
- `flags` (u16 LE):
  - `0x0001`: `ASSUME_VALID`
  - `0x0002`: `EXTENDED`
  - `0x000C`: `STAGE_MASK` (0: Normal, 1: Base, 2: Ours, 3: Theirs)
  - `0x0010`: `SKIP_WORKTREE`
  - `0x0020`: `SPARSE_DIR` (ディレクトリ集約エントリ)
- `path`: Unicode NFC 正規化された UTF-8 文字列。

---

# 4. Sparse Index (部分チェックアウトインデックス) 拡張構造

Sparse Checkout 有効時、展開されていないサブディレクトリを個別のファイルエントリとして展開せず、`SFDR` (Directory Node) 単位の 1 エントリとしてインデックス内に集約保持する（`SPARSE_DIR` フラグ）。これによりインデックスサイズとステータス走査速度が大幅に改善される。

---

# 5. Racy Git 問題の回避アルゴリズム (Racy-Free Inspection)

ファイル作成・更新タイムスタンプがインデックス書き込みタイムスタンプと同一秒（または同一ミリ秒）内に発生した場合、ファイルが変更されているにもかかわらず `mtime` と `file_size` の一致により変更を見逃す「Racy Git 問題」を防止する。

### 5.1 判定ロジック
ファイル走査時、以下の条件を満たすエントリを **Racy Entry** と判定する。

$$\text{RacyCondition} = (\text{entry.mtime} \ge \text{index.write\_time}) \lor (\text{entry.mtime} == 0)$$

- **回避動作**: Racy Entry については `mtime` の一致のみで Clean と判定せず、強制的にファイルの実データハッシュ (BLAKE3) を再計算して完全比較（Dirty 強制検証）を行う。

---

# 6. Rust / WASM Core & TypeScript インターフェース

```rust
pub struct IndexEntry {
    pub mtime_sec: i64,
    pub mtime_nsec: u32,
    pub file_size: u64,
    pub mode: u32,
    pub cid: [u8; 32],
    pub flags: u16,
    pub path: String,
}

pub fn is_racy_dirty(entry: &IndexEntry, current_mtime_sec: i64, index_write_sec: i64, current_hash: &[u8; 32]) -> bool {
    if entry.mtime_sec >= index_write_sec {
        return entry.cid != *current_hash;
    }
    entry.cid != *current_hash
}
```

```typescript
export interface IndexEntryModel {
  path: string;
  cid: Uint8Array;
  mtimeSec: bigint;
  mtimeNsec: number;
  fileSize: bigint;
  mode: number;
  flags: number;
}

export interface IndexManagerFacade {
  loadIndex(): Promise<void>;
  saveIndexAtomic(): Promise<void>;
  updateEntry(path: string, stat: Stat, cid: Uint8Array): void;
  isEntryDirty(path: string, currentStat: Stat): Promise<boolean>;
}
```
