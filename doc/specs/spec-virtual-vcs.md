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
ハッシュ計算（SHA-256 / FastCDC 用 Gear Hash）を抽象化する。

```typescript
export interface CryptoAdapter {
  /** SHA-256 ハッシュ計算 (32バイト) */
  sha256(data: Uint8Array): Promise<Uint8Array>;

  /** Gear Hash または WebAssembly (WASM) 加速ハッシュの実行 */
  fastCdcHash?(data: Uint8Array, offset: number, length: number): number;
}
```

- **Node.js**: `node:crypto` を使用。
- **ブラウザ**: Web Crypto API (`crypto.subtle.digest('SHA-256', ...)`) を使用。
- **WASM 加速**: C/Rust からコンパイルされた WebAssembly モジュールを組み込むことで、大量ハッシュ計算および FastCDC 処理を物理ネイティブに近い性能に引き上げる。

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

# 5. モジュール分離と依存関係ルール

仮想 VCS 機能の導入に伴い、コードベースおよび設定の依存関係は以下のように厳格に分類・分離する。

1. **`@sfvcs/core`**:
   - 外部依存ゼロ（Pure TypeScript）。
   - ブラウザ・Node.js 双方で同一のバイナリが動作。
2. **`@sfvcs/wasm`**:
   - FastCDC、Gear Hash、および Winnowing Fingerprint を高速化するための WebAssembly モジュール（任意導入）。
3. **`@sfvcs/adapter-browser`**:
   - `IndexedDB`, `OPFS`, `WebCrypto`, `CompressionStream` アダプタ集。
4. **`@sfvcs/adapter-node`**:
   - `node:fs`, `node:crypto`, `node:zlib` アダプタ集。
