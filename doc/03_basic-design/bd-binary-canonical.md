# sfvcs 基本設計書: バイナリフォーマット・カノニカルシリアライズ
(`bd-binary-canonical.md`)

---

# 1. 概要と目的

本設計書は、分散バージョン管理システム `sfvcs` (Structure-aware Fast Version Control System) における全オブジェクト（Chunk, Sequence Node, Symlink, File Node, Directory Node, Commit Object, LFS Pointer, Keyring, CRL）のカノニカル（決定性）バイナリシリアライズ・デコード・構造バリデーション、並びにバイトレベルの型決定性の基本設計書である。

本書は `doc/specs/spec-binary-format.md` の全仕様（第1節〜第5.3節）を完全網羅し、一切の省略や後回しを行わず、カプセル化された Model & Codec レイヤーとして具体的なデータ構造、フィールドバイトオフセット、Rust/WASM コーデック処理、および TypeScript API 境界を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Model & Canonical Codec Layer** に属し、上位の Business Service（CommitService, StorageService, MergeEngine 等）に対して完全不変（Immutable）なオブジェクトモデルと非同期エンコード/デコード API を提供する。

```
+-----------------------------------------------------------------------+
|                Domain Business Services (Commit / GC / Merge)        |
+-----------------------------------------------------------------------+
                                   | (Domain Model Objects)
                                   v
+-----------------------------------------------------------------------+
|         bd-binary-canonical (Model & Canonical Codec Layer)           |
|  - TypeScript Facade: Data Models & High-level Type Validation        |
|  - Rust/WASM Core: Zero-copy Binary Encoder / Decoder (nom / bincode) |
+-----------------------------------------------------------------------+
                                   | (Canonical Bytes / CID)
                                   v
+-----------------------------------------------------------------------+
|                 Storage Adapter / Transport Layer                     |
+-----------------------------------------------------------------------+
```

### 2.1 責任と設計原則
1. **決定論的シリアライズ (Canonical Determinism)**:
   - 同一の内容を持つオブジェクトは、生成環境（Node.js, Browser, WASM, OS）や実行時刻、バイト順序に依存せず、常に 100% 同一のバイナリ表現および CID (Content Identifier) を生成する。
2. **完全カプセル化 (Strict Encapsulation)**:
   - オブジェクト内部のプライベートデータ構造やエンコーディング詳細は本モジュール内に隠蔽し、ドメインサービス層には読み取り専用インターフェースおよびビルダーパターンのみを露出する。
3. **Rust/WASM 高速アクセラレーション**:
   - パース処理およびシリアライズは Rust の `nom` クレートを用いて実装し、WASM メモリ内でゼロコピーデコードを実行する。

---

# 3. カノニカルエンコーディング仕様

### 3.1 整数および可変長表現
1. **固定長整数 (Fixed-width Integers)**: すべての多バイト固定長数値（u16, u32, u64, i16, i64）は `spec-binary-format.md` に従い **ビッグエンディアン (Big-Endian / Network Byte Order)** で統一表現する。
2. **可変長整数 (Varint)**: オブジェクト長や配列要素数、フラグなどの可変長表現には **Protocol Buffers 形式の Unsigned LEB128 (varint)** を使用する。
   - LEB128 は各バイトの下位 7 ビットをデータ、最上位ビット (MSB) を継続フラグ（1: 続く, 0: 終了）とする。

### 3.2 文字列表現および Unicode NFC パス正規化ルール
1. **文字コード**: すべての文字列（ファイルパス、コミットメッセージ、著者名、属性キー/値など）は **UTF-8** で符号化する。
2. **パス正規化 (Unicode NFC)**:
   - macOS 等で発生する NFD (Normalization Form D: 結合文字分解) や非正規化 UTF-8 パスは、エンコード前に必ず **Unicode Canonical Decomposition followed by Canonical Composition (NFC)** に変換する。
   - ディレクトリ区切り文字は常にスラッシュ `/` (`0x2F`) に統一し、バックスラッシュ `\` (`0x5C`) や末尾スラッシュは除去する。

### 3.3 CID (Content Identifier) 表現および BLAKE3 デフォルト仕様
オブジェクトのハッシュ識別子 CID は、暗号学的衝突攻撃の防止およびオブジェクト型の型安全性を保証するため、ドメイン分離タグをプレフィックスとして付与して計算する。

$$\text{CID} = \text{BLAKE3}(\text{DomainTag} \mathbin{\Vert} \text{SerializedPayload})$$

- **デフォルトハッシュアルゴリズム**: BLAKE3 (`algo_id = 0x02`, 256-bit / 32 bytes)
- **互換ハッシュアルゴリズム**: SHA-256 (`algo_id = 0x01`, 256-bit / 32 bytes)
- **CID 33バイト構造**: `[algo_id: uint8 (1 byte)][hash_bytes: 32 bytes]`

| オブジェクト種別 | ドメインタグ (Ascii String) | ドメインタグ (Hex バイト列) | Format Version |
| ---------------- | --------------------------- | --------------------------- | -------------- |
| Chunk Object     | `SFCK`                      | `0x53, 0x46, 0x43, 0x4B`    | `0x01`         |
| Sequence Node    | `SFSQ`                      | `0x53, 0x46, 0x53, 0x51`    | `0x01`         |
| Symlink Object   | `SFSL`                      | `0x53, 0x46, 0x53, 0x4C`    | `0x01`         |
| File Node        | `SFFL`                      | `0x53, 0x46, 0x46, 0x4C`    | `0x01`         |
| Directory Node   | `SFDR`                      | `0x53, 0x46, 0x44, 0x52`    | `0x01`         |
| Commit Object    | `SFCM`                      | `0x53, 0x46, 0x43, 0x4D`    | `0x01`         |

### 3.4 シリアライズヘッダ構造
全オブジェクトの最先頭には以下の 4 フィールド共通ヘッダが付与される。
```
+-------------------------------------------------------+
| Domain Tag (4 bytes ASCII: SFCK/SFSQ/SFSL/SFFL/SFDR/SFCM) |
+-------------------------------------------------------+
| Format Version (uint8: 0x01)                          |
+-------------------------------------------------------+
| Payload Length (varint)                               |
+-------------------------------------------------------+
| Object Type Payload (可変長)                          |
+-------------------------------------------------------+
```

---

# 4. オブジェクト別ペイロード定義

### 4.1 Chunk Object (`SFCK`)
データファイルの実コンテンツを FastCDC で分割した最小生バイナリブロック。
- **Payload Layout**:生バイナリデータバイト列 ($N$ bytes)。
- **CID 計算式**: $\text{CID} = \text{BLAKE3}(\text{"SFCK"} \mathbin{\Vert} \text{raw\_bytes})$

### 4.2 Sequence Node (`SFSQ`) - Prolly Tree 内部/葉ノード
Prolly Tree (Sequence Tree) の節ノードまたは葉ノードを構成するバイナリ構造。
```
+------------------------------------------------------------------------+
| Node Flags (uint8)                                                     |
+------------------------------------------------------------------------+
| Logical Subtree Byte Length (varint)                                   |
+------------------------------------------------------------------------+
| Children Count (varint: N)                                             |
+------------------------------------------------------------------------+
| Array of Child Subtree Logical Lengths (varint x N)                    |
+------------------------------------------------------------------------+
| Array of Child CIDs (33 bytes x N: [algo_id][32B hash])                |
+------------------------------------------------------------------------+
```
- **Node Flags 仕様**:
  - `0x01` (`FLAG_LEAF_CHILDREN`): 子要素がすべて `Chunk` (`SFCK`) である場合
  - `0x02` (`FLAG_INTERNAL_CHILDREN`): 子要素がすべて下位の `Sequence Node` (`SFSQ`) である場合

### 4.3 Symlink Object (`SFSL`)
シンボリックリンクの参照先ターゲットパスを記録するノード。
```
+------------------------------------------------------------------------+
| Target Path Length (Varint) | Target Path (UTF-8 NFC String)           |
+------------------------------------------------------------------------+
```

### 4.4 File Node (`SFFL`)
メタデータ、拡張属性 (xattr)、およびコンテンツ参照を保持するノード。
```
+------------------------------------------------------------------------+
| File Flags (uint8)                                                     |
+------------------------------------------------------------------------+
| Unix Mode / Permissions (uint32 BE)                                    |
+------------------------------------------------------------------------+
| Total File Size in Bytes (uint64 BE)                                   |
+------------------------------------------------------------------------+
| Extended Attributes (xattr) Count (varint)                             |
+------------------------------------------------------------------------+
| Array of xattr entries (キーのUTF-8バイト昇順ソート済み):              |
|   [key: String][val_len: varint][val_bytes: bytes]                     |
+------------------------------------------------------------------------+
| Content Storage Type (uint8)                                           |
+------------------------------------------------------------------------+
| Content Payload (インラインバイト OR Content Root CID OR LFS Spec)     |
+------------------------------------------------------------------------+
```
- **File Flags 仕様**:
  - `0x01` (`FILE_EXEC`): 実行可能ファイル権限 (`0755`)
  - `0x02` (`FILE_HAS_XATTR`): 拡張属性 (xattr) を保持している場合
- **Content Storage Type 仕様**:
  - `0x00` (`CONTENT_INLINE`): ファイルサイズ 1,024 バイト以下。Payload 構造: `[length: varint][raw_bytes: length]`
  - `0x01` (`CONTENT_SEQUENCE_ROOT`): 通常ファイル。Payload 構造: `[content_root_cid: 33 bytes]`
  - `0x02` (`CONTENT_LFS_POINTER`): LFS ポインタ。Payload 構造: `[lfs_oid_algo: uint8][lfs_oid_bytes: 32 bytes][size_uint64: uint64 BE]`
- **xattr 決定性ソート規則**: Key は UTF-8 バイト順（Lexicographical）で厳密に昇順ソートしてエンコードする。重複キーは禁止。

### 4.5 Directory Node (`SFDR`)
ディレクトリエントリのリストを記録するノード。
```
+------------------------------------------------------------------------+
| Directory Permissions / Mode (uint32 BE)                               |
+------------------------------------------------------------------------+
| Entry Count (varint)                                                   |
+------------------------------------------------------------------------+
| Array of Entries (名前のUTF-8バイト昇順ソート済み):                    |
|   [Entry Name Len: varint][Entry Name: UTF-8 NFC]                      |
|   [Entry Type: uint8][Target CID: 33 bytes]                            |
+------------------------------------------------------------------------+
```
- **Entry Type 仕様**:
  - `0x01` (`ENTRY_FILE`): File Node (`SFFL`)
  - `0x02` (`ENTRY_DIRECTORY`): Directory Node (`SFDR`)
  - `0x03` (`ENTRY_SYMLINK`): Symlink Object (`SFSL`)
  - `0x04` (`ENTRY_SUBMODULE`): サブモジュール（Commit `SFCM` CID）
- **ソート規則**: Entry は `Entry Name` の UTF-8 バイト昇順（Lexicographical order）でソートを強制。重複名は禁止。

### 4.6 Commit Object (`SFCM`)
リポジトリ履歴のスナップショットノード。
```
+------------------------------------------------------------------------+
| Root Directory CID (33 bytes)                                          |
+------------------------------------------------------------------------+
| Parent Count (uint8)                                                   |
+------------------------------------------------------------------------+
| Parent Commit CIDs (33 bytes x Parent Count)                           |
+------------------------------------------------------------------------+
| Author Name (Length-prefixed UTF-8 NFC String)                         |
+------------------------------------------------------------------------+
| Author Email (Length-prefixed UTF-8 String)                            |
+------------------------------------------------------------------------+
| Author Timestamp Unix Epoch Seconds (int64 BE)                         |
+------------------------------------------------------------------------+
| Author Timezone Offset Minutes (int16 BE)                              |
+------------------------------------------------------------------------+
| Committer Name (Length-prefixed UTF-8 NFC String)                      |
+------------------------------------------------------------------------+
| Committer Email (Length-prefixed UTF-8 String)                         |
+------------------------------------------------------------------------+
| Committer Timestamp Unix Epoch Seconds (int64 BE)                      |
+------------------------------------------------------------------------+
| Committer Timezone Offset Minutes (int16 BE)                           |
+------------------------------------------------------------------------+
| Commit Message Bytes (Length-prefixed UTF-8 String)                    |
+------------------------------------------------------------------------+
| Signature Type (uint8)                                                 |
+------------------------------------------------------------------------+
| Signature Bytes Length (varint)                                        |
+------------------------------------------------------------------------+
| Signature Bytes (可変長)                                               |
+------------------------------------------------------------------------+
```
- **Signature Type 仕様**:
  - `0x00` (`SIG_NONE`): 署名なし
  - `0x01` (`SIG_ED25519`): Ed25519 生署名バイト (64バイト)
  - `0x02` (`SIG_SSH`): SSH 署名フォーマット文字列
  - `0x03` (`SIG_GPG`): GPG / PGP ASCII Armor 署名ブロック文字列

---

# 5. LFS ポインタ, キーリング (`SFKR`), 鍵失効リスト (`SFRL`) バイナリ規格

### 5.1 LFS ポインタフォーマット
```text
version https://sfvcs.org/spec/v1/lfs
oid sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
size 104857600
```

### 5.2 信頼キーリングバイナリ規格 (`SFKR`)
`.sfvcs/keyring` のバイナリレイアウト構造。
```
+------------------------------------------------------------------------+
| Magic "SFKR" (4 bytes)                                                 |
+------------------------------------------------------------------------+
| Format Version: 0x01 (uint8)                                           |
+------------------------------------------------------------------------+
| Key Count (varint)                                                     |
+------------------------------------------------------------------------+
| Key Entries (キーID昇順ソート済み):                                    |
|   - Key ID Length (varint)                                             |
|   - Key ID Bytes (UTF-8, Key Fingerprint)                              |
|   - Key Type (0x01: Ed25519, 0x02: RSA, 0x03: ECDSA)                   |
|   - Public Key Bytes Length (varint)                                   |
|   - Public Key Raw Bytes                                               |
+------------------------------------------------------------------------+
```

### 5.3 鍵失効リスト規格 (`SFRL`)
失効したコミット署名鍵のインデックスを保持する `.sfvcs/crl` バイナリレイアウト構造。
```
+------------------------------------------------------------------------+
| Magic "SFRL" (4 bytes)                                                 |
+------------------------------------------------------------------------+
| Format Version: 0x01 (uint8)                                           |
+------------------------------------------------------------------------+
| Revoked Key Count (varint)                                             |
+------------------------------------------------------------------------+
| Revoked Key Fingerprints (32 bytes x Count)                            |
+------------------------------------------------------------------------+
| Revocation Reason Code Array (uint8 x Count)                           |
+------------------------------------------------------------------------+
```

---

# 6. TypeScript API & Rust/WASM インターフェース定義

```typescript
export interface FileNodeModel {
  size: bigint;
  mode: number;
  flags: number;
  contentType: number;
  contentCid: Uint8Array;
  xattrs: Map<string, Uint8Array>;
}

export interface CommitObjectModel {
  treeRootCid: Uint8Array;
  parentCids: Uint8Array[];
  author: { name: string; email: string; timestamp: bigint; tzOffset: number };
  committer: { name: string; email: string; timestamp: bigint; tzOffset: number };
  signatureType: number;
  signatureBytes?: Uint8Array;
  message: string;
}

export interface CanonicalCodecEngine {
  encodeFileNode(model: FileNodeModel): Uint8Array;
  decodeFileNode(bytes: Uint8Array): FileNodeModel;
  encodeCommit(model: CommitObjectModel): Uint8Array;
  decodeCommit(bytes: Uint8Array): CommitObjectModel;
  computeCid(domainTag: string, payload: Uint8Array, algo?: "blake3" | "sha256"): Uint8Array;
}
```

```rust
#[wasm_bindgen]
pub struct CanonicalCodec {
    // Rust WASM high-speed zero-copy encoder/decoder
}

#[wasm_bindgen]
impl CanonicalCodec {
    pub fn encode_file_node(val: JsValue) -> Result<Vec<u8>, JsValue> { ... }
    pub fn decode_file_node(bytes: &[u8]) -> Result<JsValue, JsValue> { ... }
    pub fn compute_cid(domain_tag: &[u8], payload: &[u8], algo_id: u8) -> Vec<u8> { ... }
}
```
