# sfvcs 基本設計書: プラガブル・アダプタ構造 & 抽象 Storage/Crypto/Compression インターフェース
(`bd-virtual-adapters.md`)

---

# 1. 概要と目的

本設計書は、物理ファイルシステムへの直接依存を完全に排除し、インメモリ、Web ブラウザ（IndexedDB, OPFS）、Node.js、WebAssembly 環境で同一の VCS コアロジックをシームレスに動作させるプラガブル・アダプタ構造 (Pluggable Adapters) および 仮想ワークツリー (Virtual Working Tree) 仕様の基本設計書である。

本書は `doc/02_specs/spec-virtual-vcs.md` の第1節 (1.1, 1.2), 第2節, 第3節 (3.1, 3.2, 3.3), 第4節 (4.1, 4.2), 第6節および `doc/01_architecture/sfvcs-design.md` の仕様を完全網羅し、カプセル化された Virtual VCS Layer モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Virtual VCS Interface & Adapter Layer** に属し、上位の全 VCS サービスに対して抽象アダプタ群を提供する。

```
+-----------------------------------------------------------------------+
|              VCS Core Services & Domain Business Layer                |
+-----------------------------------------------------------------------+
                                   | Read/Write, Crypto, Compress
                                   v
+-----------------------------------------------------------------------+
|        bd-virtual-adapters (Pluggable Adapter Layer)                  |
|  - StorageAdapter Interface (NodeFS / Memory / OPFS / IndexedDB)      |
|  - CryptoAdapter Interface (NodeCrypto / WebCrypto / WASM)            |
|  - CompressionAdapter Interface (ZstdWasm / Native Deflate)           |
+-----------------------------------------------------------------------+
                                   | Concrete Storage & Drivers
                                   v
+-----------------------------------------------------------------------+
|         IndexedDB / OPFS / Memory / Node.js fs / Crypto Drivers       |
+-----------------------------------------------------------------------+
```

---

# 3. 仮想ワークツリーデータ構造 (`VirtualWorkingTree`) & スナップショットフロー

Web ブラウザやメモリ環境において、物理ディスクなしでファイル編集・ステージング・コミットを行うデータ構造と処理フロー。

### 3.1 `VirtualWorkingTree` データ構造
```typescript
export interface VirtualWorkingTreeEntry {
  path: string; // NFC 正規化済み相対パス
  content: Uint8Array;
  mode: number;
  mtimeMs: number;
  isDirty: boolean;
}

export class VirtualWorkingTree {
  private entries: Map<string, VirtualWorkingTreeEntry> = new Map();

  public setFile(path: string, content: Uint8Array, mode = 0o644): void {
    const normalizedPath = normalizeNfcPath(path);
    this.entries.set(normalizedPath, {
      path: normalizedPath,
      content,
      mode,
      mtimeMs: Date.now(),
      isDirty: true,
    });
  }

  public getFile(path: string): VirtualWorkingTreeEntry | undefined {
    return this.entries.get(normalizeNfcPath(path));
  }

  public removeFile(path: string): boolean {
    return this.entries.delete(normalizeNfcPath(path));
  }

  public listFiles(): VirtualWorkingTreeEntry[] {
    return Array.from(this.entries.values());
  }
}
```

### 3.2 インメモリ・スナップショット作成フロー
1. `VirtualWorkingTree.listFiles()` で Dirty 状態のエントリを抽出。
2. チャンキングエンジン (FastCDC) により Chunk CID (`SFCK`) を生成。
3. Prolly Tree Builder によりルート Sequence Node CID (`SFSQ`) を構築しコミット。

---

# 4. 抽象アダプタインターフェース仕様

### 4.1 `StorageAdapter` インターフェース
```typescript
export interface StorageAdapter {
  readFile(path: string): Promise<Uint8Array>;
  writeFile(path: string, data: Uint8Array): Promise<void>;
  deleteFile(path: string): Promise<void>;
  exists(path: string): Promise<boolean>;
  readDir(path: string): Promise<string[]>;
  stat(path: string): Promise<{ size: bigint; mtimeMs: number; isDirectory: boolean }>;
  lock(path: string, ttlMs: number): Promise<Disposable>;
}
```

- **ビルトイン Storage ドライバ一覧**:
  1. `NodeFsAdapter`: Node.js 環境でのローカルディスク I/O (`fs/promises`)。
  2. `MemoryAdapter`: 完全インメモリハッシュマップストレージ（高速ユニットテスト用）。
  3. `OPFSAdapter`: Web ブラウザ向け Origin Private File System (`FileSystemSyncAccessHandle`) アダプタ。
  4. `IndexedDBAdapter`: ブラウザ IndexedDB アダプタ。

### 4.2 `CryptoAdapter` インターフェース
```typescript
export interface CryptoAdapter {
  blake3(data: Uint8Array): Promise<Uint8Array>;
  sha256(data: Uint8Array): Promise<Uint8Array>;
  randomBytes(length: number): Uint8Array;
  sign(privateKey: Uint8Array, message: Uint8Array): Promise<Uint8Array>;
  verify(publicKey: Uint8Array, message: Uint8Array, signature: Uint8Array): Promise<boolean>;
}
```

### 4.3 `CompressionAdapter` インターフェース
```typescript
export interface CompressionAdapter {
  compress(data: Uint8Array): Promise<Uint8Array>;
  decompress(data: Uint8Array): Promise<Uint8Array>;
}
```

---

# 5. モジュール分離と依存関係ルール

1. **VCS Core の純粋性**: `src/core` 内のモジュールは Node.js の `fs`, `path`, `crypto` モジュールを直接 `import` してはならない。必ず依存注入（Dependency Injection）された `StorageAdapter` および `CryptoAdapter` を経由する。
2. **アダプタ切り替えの透明性**: 設定パラメータ `vcs.adapter = "opfs" | "memory" | "node"` により、アプリケーションコードを変更せずに実行ストレージ環境を動的切り替え可能とする。
