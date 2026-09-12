# sfvcs 仮想VCS (インメモリ / ブラウザ / Web環境) 詳細仕様書

本書は `sfvcs` における物理ファイルシステム非依存の **仮想VCS (Virtual VCS / In-Memory VCS)** 機能の詳細仕様を定義する。

Web ブラウザ環境（オンライン IDE、コードエディタ、プロジェクト管理ツール）や、エフェメラルなサーバーレス/テスト環境において、完全なバージョン管理履歴機能をメモリまたは各種ブラウザストレージ上で実現することを目的とする。

---

# 1. 仮想VCS の概要と設計目標

## 1.1 概要
`sfvcs` のコアロジック（CDC、Prolly Tree 構築、Multi-resolution Diff、カノニカルオブジェクトシリアライズ）は、一切の Node.js 固有 API（`node:fs` など）に依存しない。

外部環境との接続（ストレージ入出力、ハッシュ計算、データ圧縮）をプラガブルなアダプタインターフェースを介して抽象化することで、同一コードベースが Node.js、Deno、Bun、および主要 Web ブラウザ（Chrome, Firefox, Safari 等）でシームレスに動作する。

## 1.2 主なユースケース
- **Web IDE / エディタ**: ブラウザ上でプロジェクトのコミット・ブランチ切替・Diff表示・タイムトラベル機能を提供。
- **インメモリテスト・CI**: ディスク I/O を発生させず高速にリポジトリ操作をテスト。
- **ブラウザ内ストレージ連携**: `IndexedDB` や `OPFS (Origin Private File System)`、`LocalStorage` に永続化。

---

# 2. プラガブル・アダプタ構造 (Pluggable Adapters)

高モジュール性・低結合度を実現するため、以下の 3 つの抽象アダプタを定義する。

```
+-----------------------------------------------------------------+
|                        sfvcs Core Logic                         |
|   (CDC, Sequence Tree, Multi-resolution Diff, Canonical Serial) |
+-----------------------------------------------------------------+
        │                         │                        │
        ▼                         ▼                        ▼
+---------------+       +------------------+     +--------------------+
| StorageAdapter|       |   CryptoAdapter  |     | CompressionAdapter |
+---------------+       +------------------+     +--------------------+
  ├── NodeFs      ├── WebCrypto          ├── NativeZlib/Deflate
  ├── MemoryMap   ├── WASM (BLAKE3/SHA)  ├── CompressionStream
  ├── IndexedDB   └── NodeCrypto         └── PureJS Fallback
  ├── OPFS
  └── LocalStorage
```

---

# 3. 抽象アダプタインターフェース詳細

## 3.1 `StorageAdapter` インターフェース
オブジェクトおよび Reference データの永続化・読み書きを抽象化する。

```typescript
export interface StorageAdapter {
  /** オブジェクトデータの保存 (CID -> Uint8Array) */
  putObject(cid: Uint8Array, bytes: Uint8Array): Promise<void>;
  
  /** オブジェクトデータの取得 */
  getObject(cid: Uint8Array): Promise<Uint8Array | null>;
  
  /** オブジェクトの存在確認 */
  hasObject(cid: Uint8Array): Promise<boolean>;
  
  /** オブジェクトの削除 (GC用) */
  deleteObject(cid: Uint8Array): Promise<void>;

  /** 全 CID の列挙 (GC / fsck 用) */
  listObjectCids(): AsyncIterable<Uint8Array>;

  /** 参照 (Ref) の読み込み (例: "refs/heads/main" -> CID) */
  getRef(refPath: string): Promise<Uint8Array | null>;

  /** 参照 (Ref) の CAS (Compare-And-Swap) アトミック更新 */
  setRefCas(refPath: string, newCid: Uint8Array, expectedOldCid: Uint8Array | null): Promise<boolean>;
}
```

### ビルトイン Storage ドライバ一覧
1. **`MemoryStorageAdapter`**: メモリ上の `Map<string, Uint8Array>` で保持。セッション終了時に消滅。最も高速。
2. **`IndexedDbStorageAdapter`**: ブラウザの `IndexedDB` オブジェクトストアに永続化。大容量対応。
3. **`OpfsStorageAdapter`**: ブラウザの `Origin Private File System` を使用。物理ファイルシステムに近い高速 I/O。
4. **`LocalStorageAdapter`**: `localStorage` (Base64エンコード) を使用。小規模プロジェクト向け。
5. **`NodeFsStorageAdapter`**: 従来の `.sfvcs/` 物理ディレクトリ (`node:fs`) を使用。

---

## 3.2 `CryptoAdapter` インターフェース
ハッシュ計算（BLAKE3 / SHA-256 / FastCDC 用 Gear Hash）を抽象化する。

```typescript
export interface CryptoAdapter {
  /** BLAKE3 または SHA-256 ハッシュ計算 (32バイト) */
  hash(data: Uint8Array, algo?: 'blake3' | 'sha256'): Promise<Uint8Array>;

  /** Gear Hash または WebAssembly (WASM) 加速ハッシュの実行 */
  fastCdcHash?(data: Uint8Array, offset: number, length: number): number;
}
```

- **Node.js**: `node:crypto` または BLAKE3 ネイティブバインディングを使用。
- **ブラウザ**: Web Crypto API または BLAKE3 WASM モジュールを使用。
- **WASM 加速**: C/Rust からコンパイルされた WebAssembly モジュールを組み込むことで、大量ハッシュ計算および FastCDC 処理を物理ネイティブに近い性能に引き上げる。

## 3.3 WebCrypto によるコミット電子署名・鍵管理・検証仕様 (`WebCryptoSigner`)
Web ブラウザ環境において外部ツール（GPG / OpenSSH CLI）が存在しない場合、W3C 標準の **Web Crypto API** (`SubtleCrypto`) を用いて Ed25519 コミット電子署名生成および公開鍵検証を行う。

### 鍵ペア保持構造 (`IndexedDBKeyring`)
- 生成された Ed25519 鍵ペア（`CryptoKey` オブジェクト: `extractable: false`）は、`IndexedDB` の非公開オブジェクトストア `sfvcs_keyring` に保存し、ブラウザ外への秘密鍵の不正漏洩を物理的に遮断する。

### 署名および検証シーケンス (`SubtleCrypto`)
```typescript
export interface WebCryptoKeyringAdapter {
  /** Ed25519 鍵ペアの生成および IndexedDB 保存 */
  generateEd25519KeyPair(): Promise<CryptoKeyPair>;

  /** コミットデータバイト列への Ed25519 電子署名 (SIG_ED25519) 計算 */
  signCommitData(commitBytes: Uint8Array, privateKey: CryptoKey): Promise<Uint8Array>;

  /** WebCrypto による Ed25519 コミット電子署名検証 */
  verifyCommitSignature(commitBytes: Uint8Array, signatureBytes: Uint8Array, publicKey: CryptoKey): Promise<boolean>;
}
```
- **署名生成**: `window.crypto.subtle.sign({ name: 'Ed25519' }, privateKey, commitPayloadBytes)`
- **署名検証**: `window.crypto.subtle.verify({ name: 'Ed25519' }, publicKey, signatureBytes, commitPayloadBytes)`

---

## 3.3 `CompressionAdapter` インターフェース
オブジェクトデータの圧縮・解凍を抽象化する。

```typescript
export interface CompressionAdapter {
  compress(uncompressed: Uint8Array): Promise<Uint8Array>;
  decompress(compressed: Uint8Array): Promise<Uint8Array>;
}
```

- **ブラウザ**: `CompressionStream('deflate')` / `DecompressionStream('deflate')` 標準 API を優先使用。
- **Node.js**: `node:zlib` を使用。

---

# 4. 仮想ワークツリー (Virtual Working Tree) 仕様

ブラウザ上の Web IDE やコードエディタ（Monaco Editor など）では、物理ディスク上のファイルを介さず、インメモリのファイルツリー構造体から直接スナップショットを作成する。

## 4.1 メモリワークツリーデータ構造 (`VirtualWorkingTree`)

```typescript
export type VirtualFileContent = Uint8Array | string;

export interface VirtualFileEntry {
  path: string;                # 相対パス (例: "src/index.ts")
  content: VirtualFileContent; # ファイル内容 (文字列またはバイト列)
  mode?: number;               # 実行権限等 (デフォルト: 0644)
}

export type VirtualWorkingTree = VirtualFileEntry[];
```

## 4.2 インメモリ・スナップショット作成フロー

```typescript
import { VirtualRepository, MemoryStorageAdapter } from 'sfvcs';

// 1. インメモリリポジトリの初期化
const repo = await VirtualRepository.create({
  storage: new MemoryStorageAdapter(),
});

// 2. Web エディタ上の仮想ファイル群を定義
const files: VirtualWorkingTree = [
  { path: 'README.md', content: '# My Web Project\nHello World' },
  { path: 'src/app.js', content: 'console.log("Running in browser");' }
];

// 3. 仮想ワークツリーから直接コミットを生成
const commitCid = await repo.commitVirtualTree({
  tree: files,
  message: 'Initial commit from Web IDE',
  author: { name: 'Web User', email: 'user@example.com' }
});

// 4. 過去コミットの内容を仮想ワークツリーとして復元
const restoredFiles = await repo.checkoutVirtualTree(commitCid);
```

---

# 5. ブラウザ環境でのリソース制限対策・Web Worker ストリーミング処理仕様

Web ブラウザのメインスレッド（UI スレッド）のフリーズを防ぎ、巨大ファイルや大規模リポジトリを安定処理するための仕様。

## 5.1 Web Worker ストリーミング実行モデル
重い処理（FastCDC チャンク分割、BLAKE3 ハッシュ計算、Prolly Tree 構築、Diff 探索）は、専用の **Dedicated Web Worker** 内で実行する。

- メインスレッドと Worker 間は `ReadableStream` / `WritableStream` および `Transferable Objects` (ArrayBuffer の所有権移動) を用いて、零コピー (Zero-Copy) データ転送を行う。

## 5.2 OPFS / IndexedDB 巨大アセット Chunked Handling
- メモリ上限（例: 256 MB 〜 512 MB）を超過する巨大アセット処理時、ファイル全体を単一の `Uint8Array` に展開せず、一定バッファ枠（例: **4 MiB 〜 16 MiB チャンク**）ごとにストリーミング読み出しを行う。
- OPFS 利用時は `FileSystemSyncAccessHandle` を用いて、非同期 I/O オーバーヘッドを抑えた低レイテンシなチャンクドアクセスを実現する。

---

# 6. モジュール分離と依存関係ルール

1. **`@sfvcs/core`**: 外部依存ゼロ（Pure TypeScript）。ブラウザ・Node.js 双方で同一バイナリが動作。
2. **`@sfvcs/wasm`**: FastCDC、BLAKE3、Winnowing Fingerprint 高速化用 WebAssembly モジュール。
3. **`@sfvcs/adapter-browser`**: `IndexedDB`, `OPFS`, `WebCrypto`, `CompressionStream` アダプタ集。
4. **`@sfvcs/adapter-node`**: `node:fs`, `node:crypto`, `node:zlib` アダプタ集。

---

# 7. クライアントサイド暗号化ストレージ (Encryption-at-Rest) 仕様

Untrusted サーバーやブラウザ内 `IndexedDB` / `OPFS` への保存時、情報漏洩を防ぐゼロ知識（Zero-Knowledge）暗号化ストレージアダプタ `EncryptedStorageAdapter` 仕様。

## 7.1 エンベロープ暗号化 (Envelope Encryption) 鍵構造
1. **Passphrase / Master Key**: ユーザーパスフレーズから **Argon2id** (Memory: 64MB, Iterations: 3, Parallelism: 4) により Key Encryption Key (KEK) を派生。
2. **Data Encryption Key (DEK)**: ランダム生成された 256bit **AES-256-GCM** または **XChaCha20-Poly1305** 鍵。
3. **Chunk Payload Encrypted Layout**:
   ```
   [IV / Nonce (12 or 24 bytes)][Encrypted Payload][Auth Tag (16 bytes)]
   ```

---

# 8. WebAssembly サンドボックスプラグインアーキテクチャ

ブラウザや非信頼サーバー環境において、サードパーティ製フックやカスタム Diff ドライバーを安全に隔離実行する WASM プラグイン仕様。

```typescript
export interface WasmPluginAdapter {
  /** WASM モジュールのインスタンス化およびサンドボックスメモリ初期化 */
  loadPlugin(wasmBytes: Uint8Array, config?: Record<string, unknown>): Promise<WasmInstance>;

  /** カスタムフック (pre-commit / pre-push) のサンドボックス実行 */
  executeHook(hookName: string, context: HookContext): Promise<HookResult>;
}
```

---

# 9. 鍵失効リスト (CRL) 検証および鍵ローテーション手順

`WebCryptoSigner` における電子署名検証時、対象の公開鍵 Fingerprint が `.sfvcs/crl` (失効リスト) 内に含まれる場合、検証ステータスを `REVOKED_KEY_ERROR` と判定して拒否する。失効した鍵は新しく生成した Ed25519 鍵ペアへアトミックにローテーション更新する。

---

# 10. KEK / DEK エンベロープ暗号鍵ローテーション仕様 (Envelope Key Rotation)

マスターパスフレーズの変更や暗号鍵の漏洩リスクに対処するため、暗号化済みオブジェクト Payload を再暗号化（全走査・再出力）することなく、高速 $O(1)$ に鍵を安全更新するローテーション仕様。

## 10.1 ローテーション手順
1. 新しいパスフレーズから新 KEK ($\text{KEK}_{\text{new}}$) を Argon2id により派生。
2. 古い KEK ($\text{KEK}_{\text{old}}$) で暗号化されていた DEK ($\text{DEK}_{\text{raw}}$) を複合展開。
3. 暗号化データキー（Encrypted DEK）領域のみを $\text{KEK}_{\text{new}}$ で再暗号化し保存メタデータを書き換え。
- **計算量**: ストレージ内の全 Chunk オブジェクトの再暗号化を回避し、$O(1)$ で鍵更新が完了する。

---

# 11. IndexedDB / OPFS Storage Quota Management & LRU Eviction

Web ブラウザ環境でストレージ容量制限（`QuotaExceededError`）が発生した際、Derived Index (Fingerprint DB, Diff Cache) を安全に解放し、コア Object の破損を防ぐアルゴリズム。

```typescript
export async function handleQuotaExceeded(storage: StorageAdapter): Promise<void> {
  // 1. derived index / diff cache を優先的に削除して空き容量を確保
  await storage.clearDerivedIndexCache();
  
  // 2. それでも容量不足の場合は LRU ルールに基づき古くなった未到達 loose オブジェクトを解放
  await storage.evictUnreachableLooseObjectsLRU();
}
```

---

# 12. マルチスレッド/Web Worker 並列実行時の IV 再利用防止暗号化 (AES-SIV / Synthetic IV & Counter State) 仕様

複数の Web Worker や並列 Worker スレッドが同一の暗号化鍵 (DEK) を用いて並行して Object チャンクを暗号化する際、IV / Nonce の衝突による破綻を防ぐ安全アルゴリズム。

## 12.1 AES-SIV (Synthetic IV: RFC 5297) 暗号化仕様
1. **Nonce 衝突問題の回避**:
   標準の AES-GCM では並列処理時に同一 IV が再利用された場合に暗号解読の脆弱性が発生する。これを防ぐため、暗号学的合成 IV 方式 **AES-SIV (RFC 5297)** を採用する。
2. **合成 IV ($S_{iv}$) 計算式**:
   平文データ $M$（Chunk オブジェクト）および Associated Data (AD: CID ドメインタグ文字列) に対し、S2V 判定関数により $S_{iv}$ を決定論的に算出する。
   $$S_{iv} = \text{S2V}(\text{DEK}, M || \text{AD})$$
3. **並列暗号化出力レイアウト**:
   `[Synthetic_IV (16 bytes)][Ciphertext Payload]`
   各 Worker スレッドは状態共有やカウンター同期を行わずに、完全独立かつ安全に暗号化処理を実行可能となる。

