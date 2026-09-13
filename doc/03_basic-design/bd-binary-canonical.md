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
1. **固定長整数 (Fixed-width Integers)**: すべての固定長数値（u16, u32, u64, i64）は **Little-Endian** で表現する。
2. **可変長整数 (Varint)**: オブジェクト長や配列要素数、フラグなどの可変長表現には **LEB128 (Unsigned LEB128)** を使用する。
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

| オブジェクト種別 | ドメインタグ (Ascii String) | ドメインタグ (Hex バイト列) |
| ---------------- | --------------------------- | --------------------------- |
| Chunk Object     | `SFCK`                      | `0x53, 0x46, 0x43, 0x4B`    |
| Sequence Node    | `SFSQ`                      | `0x53, 0x46, 0x53, 0x51`    |
| Symlink Object   | `SFSL`                      | `0x53, 0x46, 0x53, 0x4C`    |
| File Node        | `SFFL`                      | `0x53, 0x46, 0x46, 0x4C`    |
| Directory Node   | `SFDR`                      | `0x53, 0x46, 0x44, 0x52`    |
| Commit Object    | `SFCM`                      | `0x53, 0x46, 0x43, 0x4D`    |

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
| Level (u8) | Flags (u8) | Entry Count (Varint)                         |
+------------------------------------------------------------------------+
| Entry 1: [Child CID (32B)] [Subtree Length (Varint)] [Boundary (32B)] |
| Entry 2: [Child CID (32B)] [Subtree Length (Varint)] [Boundary (32B)] |
| ...                                                                    |
+------------------------------------------------------------------------+
```
- **Level**: 0 は Leaf Node (Chunk を指す), 1 以上は Internal Node (下位 Sequence Node を指す)。
- **Flags**:
  - `0x01`: `IS_COMPRESSED` (ペイロードが Zstandard で圧縮されている)
  - `0x02`: `HAS_CUSTOM_BOUNDARY` (病的入力フォールバックフラグ)

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
| File Size (u64 LE) | Mode (u32 LE) | Flags (u8) | Content Type (u8)   |
| Content CID (32B)  | xattr Count (Varint)                              |
+------------------------------------------------------------------------+
| xattr Entry 1: [Key Len (Varint)] [Key (NFC)] [Val Len (Varint)] [Val] |
| ...                                                                    |
+------------------------------------------------------------------------+
```
- **Content Storage Type**:
  - `0x00`: `RAW_CHUNK` (単一 SFCK)
  - `0x01`: `PROLLY_TREE` (SFSQ ルート)
  - `0x02`: `LFS_POINTER` (LFS ポインタテキスト)
- **xattr 決定性ソート規則**: Key はバイト比較（Lexicographical）で昇順ソートしてエンコードする。

### 4.5 Directory Node (`SFDR`)
ディレクトリエントリのリストを記録するノード。
```
+------------------------------------------------------------------------+
| Entry Count (Varint)                                                   |
+------------------------------------------------------------------------+
| Entry 1: [Name Len (Varint)] [Name (NFC)] [Type (u8)] [Mode (u32 LE)]  |
|          [CID (32B)] [Size (u64 LE)]                                   |
| ...                                                                    |
+------------------------------------------------------------------------+
```
- **Entry Type**: `0x01` File (`SFFL`), `0x02` Directory (`SFDR`), `0x03` Symlink (`SFSL`), `0x04` Submodule (`SFCM` CID)
- **ソート規則**: Entry は `Name` の UTF-8 バイト昇順でソートを強制。

### 4.6 Commit Object (`SFCM`)
リポジトリ履歴のスナップショットノード。
```
+------------------------------------------------------------------------+
| Tree Root CID (32B) | Parent Count (Varint) | [Parent CIDs (32B...)]    |
| Author Name (NFC String) | Author Email String                         |
| Author Timestamp (i64 LE) | Author Timezone Offset (i16 LE)            |
| Committer Name | Committer Email | Committer Timestamp | Committer TZ |
| PGP Signature Type (u8) | Signature Length (Varint) | Signature Bytes  |
| Commit Message Length (Varint) | Commit Message (UTF-8 NFC String)     |
+------------------------------------------------------------------------+
```
- **Signature Type**: `0x00` None, `0x01` Ed25519, `0x02` ECDSA P-256, `0x03` PGP/GPG.

---

# 5. LFS ポインタ, キーリング (`SFKR`), 鍵失効リスト (`SFRL`) バイナリ規格

### 5.1 LFS ポインタフォーマット
```text
version https://sfvcs.org/spec/v1/lfs
oid sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
size 104857600
```

### 5.2 信頼キーリングバイナリ規格 (`SFKR`)
```
+------------------------------------------------------------------------+
| Magic: "SFKR" (4B) | Version (u8: 1) | Key Count (Varint)               |
+------------------------------------------------------------------------+
| Key 1: [Key ID (8B LE)] [Algo (u8)] [Public Key Bytes (Varint len)]    |
|        [Trust Level (u8)] [User Identifier String]                     |
+------------------------------------------------------------------------+
```

### 5.3 鍵失効リスト規格 (`SFRL`)
```
+------------------------------------------------------------------------+
| Magic: "SFRL" (4B) | Version (u8: 1) | Revoked Count (Varint)           |
+------------------------------------------------------------------------+
| Revoked Key 1: [Key ID (8B LE)] [Revocation Time (i64 LE)]             |
|                [Reason Code (u8)] [Description String]                 |
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
