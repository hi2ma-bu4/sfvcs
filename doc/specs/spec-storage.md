# sfvcs ストレージ構造・パックファイル詳細仕様書

本書は `sfvcs` における物理ストレージレイアウト、ルーズオブジェクト管理、パックファイル（Packfile）のフォーマット構造、パックインデックス、参照（Ref）更新メカニズム、並行制御ロック、およびガベージコレクション (GC) 手順を定義する仕様書である。

---

# 1. 管理ディレクトリ構造 (`.sfvcs/`)

`.sfvcs/` ディレクトリの配下構造は以下の通りとする。

```
.sfvcs/
├── config                     # リポジトリ設定ファイル (INI形式またはJSON)
├── HEAD                       # 現在 Checkout されている Branch または Commit CID
├── refs/
│   ├── heads/                 # ブランチ参照ファイル群 (例: main -> Commit CID)
│   └── tags/                  # タグ参照ファイル群
├── objects/
│   ├── loose/                 # ルーズオブジェクト格納ディレクトリ
│   │   ├── 01/                # CIDハッシュ先頭2文字のサブディレクトリ
│   │   │   └── 3a4f...        # 31バイトハッシュ名ファイル (圧縮済みデータ)
│   │   └── ...
│   └── pack/                  # パックファイル格納ディレクトリ
│       ├── pack-a1b2c3d4.pack # パック本体
│       └── pack-a1b2c3d4.idx  # パックインデックス
├── index/                     # ワークツリー状態キャッシュ（Derived Index）
├── derived/                   # Derived Index (Fingerprint, Line Index, Diff Cache)
│   ├── fingerprint.db
│   └── diff_cache/
├── locks/                     # アトミック操作用ロックファイル一時格納場所
└── tmp/                       # ストリーミング処理時の一時生成ファイル格納場所
```

---

# 2. ルーズオブジェクト (Loose Objects)

開発初期および新規オブジェクトの一次保存形式。

## 2.1 パス構造
CID が `0x01` (SHA-256) + `a1b2c3d4e5...` (32バイトhex: `a1b2c3d4e567890abcdef...`) の場合：
```
.sfvcs/objects/loose/a1/b2c3d4e567890abcdef...
```
- 先頭 2 hex 文字をディレクトリ名とし、残りの 62 hex 文字をファイル名とする（inode 枯渇の緩和）。

## 2.2 保存形式と圧縮
- **伸張前データ**: `doc/spec-binary-format.md` に定義されたカノニカルバイナリ表現 (`DomainTag || FormatVersion || PayloadLength || Payload`)。
- **圧縮アルゴリズム**: **zlib / Deflate**（圧縮レベルデフォルト: 6）または **Brotli**。
- **アトミック書き込み手順**:
  1. `.sfvcs/tmp/tmp_obj_XXXXXX` に一時ファイルとして出力・全データを書き込み。
  2. `fsync` を実行してストレージへ確実に同期。
  3. `.sfvcs/objects/loose/XX/YY...` へアトミックリネーム (`fs.renameSync`)。

---

# 3. パックファイルフォーマット (Packfile Layout)

大量のルーズオブジェクトを単一の集約ファイルにパッキングし、I/O性能の向上と圧縮率を高める形式。

## 3.1 `.pack` ファイルバイナリレイアウト

```
+-------------------------------------------------------+
| Header: Magic "SFPK" (4 bytes)                        |
+-------------------------------------------------------+
| Version: 0x00000001 (uint32 BE)                       |
+-------------------------------------------------------+
| Object Count: N (uint32 BE)                           |
+-------------------------------------------------------+
| Compressed Object Data Entry #1                       |
+-------------------------------------------------------+
| Compressed Object Data Entry #2                       |
+-------------------------------------------------------+
| ...                                                   |
+-------------------------------------------------------+
| Compressed Object Data Entry #N                       |
+-------------------------------------------------------+
| Packfile SHA-256 Checksum (32 bytes)                  |
+-------------------------------------------------------+
```

### パックオブジェクトエントリ構造
```
+-------------------------------------------------------+
| Uncompressed Object Size (varint)                     |
+-------------------------------------------------------+
| Compressed Data Size: C (varint)                      |
+-------------------------------------------------------+
| Compressed Payload Bytes (Deflate/zlib, C bytes)     |
+-------------------------------------------------------+
```
※ 注: CID 自体はパック本体内には重複保持せず、ペアとなる `.idx` ファイル側で保持・検索する。

---

## 3.2 パックインデックスフォーマット (`.idx` Layout)

高速な CID -> オフセット検索を実現する 2 レベルファンアウトテーブル構造。

```
+-------------------------------------------------------+
| Header: Magic "SPIX" (4 bytes)                        |
+-------------------------------------------------------+
| Version: 0x00000001 (uint32 BE)                       |
+-------------------------------------------------------+
| First-Level Fanout Table (256 x uint32 BE = 1,024 B)  |
+-------------------------------------------------------+
| Table of CIDs (N x 33 bytes, CID辞書順ソート済み)     |
+-------------------------------------------------------+
| Table of CRC32 Checksums (N x uint32 BE)              |
+-------------------------------------------------------+
| Table of Packfile Offsets (N x uint64 BE)             |
+-------------------------------------------------------+
| Checksum of Pack Index File (32 bytes SHA-256)        |
+-------------------------------------------------------+
```

### ファンアウトテーブルの仕組み
`FanoutTable[i]` には、CIDの1バイト目が `0x00` から `i` 以下のオブジェクトの累計件数が入る。
これによる O(1) ディレクトリ絞り込み後、二分探索 (Binary Search) にて平均 $O(\log N)$ で目的 CID の Offset を取得する。

---

# 4. Reference 更新と並行制御メカニズム

## 4.1 CAS (Compare-And-Swap) による Branch 更新
`refs/heads/main` などの参照ファイル更新時は、 race condition を回避するため CAS 方式を徹底する。

### 参照更新アルゴリズム
1. 現在の `refs/heads/main` から期待される旧 Commit CID `Expected_Old_CID` を読み込む。
2. ロックファイル `.sfvcs/refs/heads/main.lock` を `O_CREAT | O_EXCL` フラグで排他オープン作成。
3. 再度 `refs/heads/main` を確認し、`Expected_Old_CID` から変化していないか検証。
4. 変化がない場合、ロックファイルに新 Commit CID 文字列を書き込み `fsync`。
5. ロックファイルを `.sfvcs/refs/heads/main` へアトミックリネームしてロック解除。

---

# 5. ガベージコレクション (GC) 仕様

到達不能な不要オブジェクト（未コミットの試行オブジェクトや削除ブランチの過去データ）を削除・クリーンアップする手順。

## 5.1 Mark-and-Sweep 手順

1. **Root Set の収集 (Mark Phase)**:
   - `.sfvcs/refs/` 配下の全 Reference (heads, tags) が指す Commit CID を取得。
   - `HEAD` が指す Commit CID を取得。
2. **Reachable Graph 追跡**:
   - 収集した Commit から再帰的に Tree (SFDR), File (SFFL), Sequence (SFSQ), Chunk (SFCK), Symlink (SFSL) の CID を追跡し、**生存 CID フラグセット** に追加。
3. **Unreachable Sweep**:
   - `objects/loose/` および `objects/pack/` 内の全オブジェクトを走査。
   - 生存 CID フラグセットに含まれないオブジェクトを特定。
   - **安全保護ルール**: 作成日時が **24 時間以内**（`grace_period`）のファイルは sweep 対象外とする（並行実行中の commit 処理保護のため）。
4. **Repack & Clean**:
   - 残存する有効ルーズオブジェクトを統合パックファイル (`pack-XXXX.pack`) に集約。
   - 到達不能かつ保護期間外のルーズオブジェクト・旧パックファイルを安全削除。
