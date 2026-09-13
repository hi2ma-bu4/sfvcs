# sfvcs 基本設計書: sfvcs LFS (Large File Storage) & クライアントフック
(`bd-storage-lfs-hooks.md`)

---

# 1. 概要と目的

本設計書は、大容量バイナリ資産をリポジトリ本体から透過的に分離管理する sfvcs LFS (Large File Storage) のストレージ構造、ポインタファイル処理、およびクライアントフック (Client Hooks) システムの基本設計書である。

本書は `doc/specs/spec-storage.md` の第9節 (9.1), 第14節 (14.1) および `doc/01_architecture/sfvcs-design.md` の第171.3節の仕様を完全網羅し、カプセル化された Binary Asset & Hook Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Domain Service & Asset Management Layer** に属し、大容量ファイルのストレージ分離および外部フック実行を担当する。

```
+-----------------------------------------------------------------------+
|             CLI / Working Tree / Commit Service                       |
+-----------------------------------------------------------------------+
                                   | Filter File / Trigger Hook
                                   v
+-----------------------------------------------------------------------+
|        bd-storage-lfs-hooks (LFS & Hook Engine Service)               |
|  - TypeScript: Hook Execution Sandbox & LFS Pointer Parser            |
|  - Rust/WASM Core: SHA-256 Stream Hasher & Pointer Filter Engine      |
+-----------------------------------------------------------------------+
                                   | Store Media Bytes / Run Process
                                   v
+-----------------------------------------------------------------------+
|              .sfvcs/lfs/storage & Exec Process Layer                  |
+-----------------------------------------------------------------------+
```

---

# 3. sfvcs LFS (Large File Storage) ストレージ構造 & ポインタ仕様

`.sfvcsattributes` に `filter=lfs` が指定されたファイルは、オブジェクトデータベースへの格納時に以下のように処理される。

1. **LFS Pointer 変換**: 実バイナリデータを `.sfvcs/lfs/objects/` に移動し、ワーキングツリーまたはコミット対象のファイルノードには以下のテキストポインタを代行保存する。
   ```text
   version https://sfvcs.org/spec/v1/lfs
   oid sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
   size 104857600
   ```
2. **ストレージ構造**: `.sfvcs/lfs/objects/e3/b0/e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`
3. **オンデマンド同期**: ネットワーク `push` / `fetch` 時に、LFS ポインタのみを先行同期し、実バイナリデータは必要に応じてストリーミングダウンロードを行う。

---

# 4. クライアントフック (Client Hooks) システム

リポジトリ操作の各ライフサイクルにおいて、自動検証・フォーマット・通知を行うスクリプトフックを提供する。

### 4.1 フック実行タイミングと仕様
- `pre-commit`: コミット作成前に実行。非ゼロを返すとコミットを中断。
- `commit-msg`: コミットメッセージバリデーション。
- `post-commit`: コミット完了後の通知処理。
- `pre-push`: リモート送信前のリファレンス検証。
- `post-checkout`: チェックアウト完了後のワーキングツリーセットアップ。

---

# 5. TypeScript / WASM インターフェース

```typescript
export interface LfsPointerModel {
  version: string;
  oid: string;
  size: bigint;
}

export interface LfsManagerFacade {
  isLfsPointer(data: Uint8Array): boolean;
  parsePointer(data: Uint8Array): LfsPointerModel;
  cleanFile(filePath: string, inputStream: ReadableStream<Uint8Array>): Promise<Uint8Array>; // Returns Pointer text bytes
  smudgeFile(pointer: LfsPointerModel, outputStream: WritableStream<Uint8Array>): Promise<void>;
}

export interface HookRunnerFacade {
  executeHook(hookName: string, args: string[], env?: Record<string, string>): Promise<{ exitCode: number; stdout: string; stderr: string }>;
}
```
