# sfvcs 基本設計書: ゼロ知識暗号化 (Encryption-at-Rest) & KEK/DEK 暗号鍵ローテーション
(`bd-virtual-crypto.md`)

---

# 1. 概要と目的

本設計書は、クライアントサイドで透明にリポジトリオブジェクトを暗号化し、サーバーや不認可ストレージへの漏洩を防ぐゼロ知識・リポジトリ暗号化 (Encryption-at-Rest)、エンベロープ暗号化 (Envelope Encryption)、および KEK / DEK 鍵ローテーション仕様の基本設計書である。

本書は `doc/specs/spec-virtual-vcs.md` の第7節 (7.1), 第10節 (10.1), 第12節 (12.1) および `doc/01_architecture/sfvcs-design.md` の第171.4節の仕様を完全網羅し、カプセル化された Envelope Encryption Engine モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Security & Encryption Service Layer** に位置し、Storage Adapter の直上または透過的な暗号ラッパーとして機能する。

```
+-----------------------------------------------------------------------+
|              VCS Storage Manager / Storage Adapter Layer              |
+-----------------------------------------------------------------------+
                                   | Read / Write Encrypted Object
                                   v
+-----------------------------------------------------------------------+
|     bd-virtual-crypto (Envelope Encryption Engine Service)            |
|  - TypeScript: KEK / DEK Key Rotation & Master Passphrase Manager     |
|  - Rust/WASM Core: AES-SIV (Synthetic IV) Stream Encryptor Engine     |
+-----------------------------------------------------------------------+
                                   | Encrypted Byte Streams
                                   v
+-----------------------------------------------------------------------+
|                 Physical Storage / Network Wire Layer                 |
+-----------------------------------------------------------------------+
```

---

# 3. エンベロープ暗号化 (Envelope Encryption) 構造 & KEK / DEK ローテーション

### 3.1 鍵構造
- **Master Password**: ユーザーのパスフレーズ。PBKDF2 / Argon2id により Key Encryption Key (KEK) を導出。
- **Key Encryption Key (KEK)**: データ暗号化鍵 (DEK) を暗号化保護するための上位鍵。
- **Data Encryption Key (DEK)**: 個々のオブジェクトバイナリを高速暗号化するための 256-bit 鍵。

### 3.2 KEK / DEK 暗号鍵ローテーション手順 (Envelope Key Rotation)
ユーザーのパスフレーズ変更時、巨大な全オブジェクトデータを暗号化し直すことなく、$O(1)$ の時間で暗号鍵を即座にローテーションする。

1. 新パスフレーズから新 KEK ($\text{KEK}_{\text{new}}$) を生成。
2. 旧 KEK ($\text{KEK}_{\text{old}}$) で暗号化されていた DEK を復号化。
3. DEK を $\text{KEK}_{\text{new}}$ で再暗号化し、ヘッダーに保存。全オブジェクトデータ（DEK で暗号化済み）の書き換えは一切不要。

---

# 4. IV 再利用防止暗号化 (AES-SIV / Synthetic IV: RFC 5297)

マルチスレッド環境や Web Worker 並列実行環境において、同一の初期化ベクター (IV) が誤って再利用される致命的な暗号破綻を防止するため、AES-SIV (Deterministic Authenticated Encryption / Synthetic IV, RFC 5297) を採用する。

$$\text{IV}_{\text{synthetic}} = \text{CMAC}(\text{DEK}, \text{Plaintext} \mathbin{\Vert} \text{AssociatedData})$$

- プレーンテキストの内容自体から合成 IV を決定論的に生成するため、カウンタ状態の衝突や Worker 間での IV 再利用による解読リスクが理論上 0% に抑えられる。

---

# 5. Rust / WASM Core & TypeScript インターフェース

```rust
pub fn aes_siv_encrypt(dek: &[u8; 32], plaintext: &[u8], assoc_data: &[u8]) -> Vec<u8> {
    // Synthetic IV generation (RFC 5297) and AES-CTR encryption in Rust WASM
    Vec::new()
}
```

```typescript
export interface KeyHeaderModel {
  encryptedDek: Uint8Array;
  keyDerivationSalt: Uint8Array;
  iterations: number;
}

export interface EnvelopeEncryptionFacade {
  rotateMasterPassphrase(oldPass: string, newPass: string, keyHeader: KeyHeaderModel): Promise<KeyHeaderModel>;
  encryptObjectPayload(dek: Uint8Array, payload: Uint8Array, assocData: Uint8Array): Promise<Uint8Array>;
  decryptObjectPayload(dek: Uint8Array, encryptedPayload: Uint8Array, assocData: Uint8Array): Promise<Uint8Array>;
}
```
