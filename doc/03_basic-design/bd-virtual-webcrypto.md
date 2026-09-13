# sfvcs 基本設計書: WebCrypto 電子署名 & 信頼キーリング (`SFKR`) & 鍵失効リスト (`SFRL`)
(`bd-virtual-webcrypto.md`)

---

# 1. 概要と目的

本設計書は、WebCrypto API / Node.js Crypto / WASM を用いてコミットオブジェクトに対する電子署名・署名検証・鍵管理を行う WebCrypto 電子署名仕様、信頼キーリング (`SFKR`) フォーマット、および鍵失効リスト (`SFRL`) 検証の基本設計書である。

本書は `doc/specs/spec-virtual-vcs.md` の第3.3節, 第9節, 第12節、`doc/specs/spec-binary-format.md` の第5.2, 5.3節、および `doc/01_architecture/sfvcs-design.md` の第171.12節の仕様を完全網羅し、カプセル化された Cryptographic Authentication Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Cryptographic Security & Identity Layer** に位置し、コミット作成時および `fsck` 検証時の署名検証を担当する。

```
+-----------------------------------------------------------------------+
|             Commit Service / Audit Service / Security Validator       |
+-----------------------------------------------------------------------+
                                   | Sign / Verify Request
                                   v
+-----------------------------------------------------------------------+
|    bd-virtual-webcrypto (Crypto Authentication Service)               |
|  - TypeScript: WebCryptoSigner & Keyring / CRL Manager                |
|  - Rust/WASM Core: Ed25519 / ECDSA P-256 Crypto Verification Engine   |
+-----------------------------------------------------------------------+
                                   | IndexedDB Keyring / WebCrypto API
                                   v
+-----------------------------------------------------------------------+
|                 SubtleCrypto / Web Storage Layer                      |
+-----------------------------------------------------------------------+
```

---

# 3. WebCrypto によるコミット電子署名・検証仕様 (`WebCryptoSigner`)

### 3.1 署名アルゴリズム
- **Ed25519** (RFC 8032) または **ECDSA (P-256 with SHA-256)** を使用。
- 署名対象データ: コミットオブジェクトから署名フィールドを除いた決定論的バイナリペイロード。

### 3.2 署名および検証シーケンス
1. **コミット時 (Signing)**:
   - 登録済みの `CryptoKeyPair` より秘密鍵を取得（IndexedDB `IndexedDBKeyring` に不揮発保存）。
   - `crypto.subtle.sign()` により電子署名バイト列を生成し、`SFCM` オブジェクトの Signature フィールドに格納。
2. **検証時 (Verification)**:
   - コミットオブジェクトから公開鍵 ID (8-byte Key ID) および署名バイト列を抽出。
   - 信頼キーリング (`SFKR`) および 鍵失効リスト (`SFRL`) と照合後、`crypto.subtle.verify()` を用いて真正性を検証。

---

# 4. 信頼キーリング (`SFKR`) & 鍵失効リスト (`SFRL`) 検証フロー

### 4.1 検証手順
```
        [Commit Verification Request]
                     |
                     v
   +------------------------------------+
   |  Check Key ID in CRL (SFRL)?       | ----(YES: Revoked)----> [FAIL: Key Revoked]
   +------------------------------------+
                     | (NO)
                     v
   +------------------------------------+
   |  Key in Trust Keyring (SFKR)?      | ----(NO: Untrusted)---> [WARN: Untrusted Key]
   +------------------------------------+
                     | (YES)
                     v
   +------------------------------------+
   |  WebCrypto Signature Match?        | ----(NO)--------------> [FAIL: Invalid Signature]
   +------------------------------------+
                     | (YES)
                     v
             [SUCCESS: Verified]
```

---

# 5. TypeScript / WASM インターフェース

```typescript
export interface KeyringEntryModel {
  keyId: Uint8Array; // 8 bytes
  publicKeyBytes: Uint8Array;
  algorithm: "Ed25519" | "ECDSA-P256";
  trustLevel: number;
}

export interface WebCryptoSignerFacade {
  generateKeyPair(): Promise<{ keyId: Uint8Array; publicKey: Uint8Array }>;
  signCommit(commitPayload: Uint8Array, keyId: Uint8Array): Promise<Uint8Array>;
  verifyCommit(commitPayload: Uint8Array, signature: Uint8Array, keyId: Uint8Array): Promise<boolean>;
}
```
