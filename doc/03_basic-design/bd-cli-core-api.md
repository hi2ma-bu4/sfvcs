# sfvcs 基本設計書: CLI サブコマンド & Core API & エラー構造仕様
(`bd-cli-core-api.md`)

---

# 1. 概要と目的

本設計書は、ユーザーがターミナルから操作する CLI コマンドインターフェース、TypeScript / Node.js 用 Core API (Facade)、および一貫した構造化エラー（Error Taxonomy）の基本設計書である。

本書は `doc/specs/spec-cli-and-api.md` の全セクション（第1, 2, 3節）および `doc/01_architecture/sfvcs-design.md` の全コマンド要求仕様を完全網羅し、カプセル化された Controller レイヤーとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **CLI & Core API Controller Layer** に位置し、ビジネスロジック (Domain Service) のコントローラおよびユーザー I/O ファサードとして機能する。

```
+-----------------------------------------------------------------------+
|             User Terminal CLI / External Node.js Application          |
+-----------------------------------------------------------------------+
                                   | Invoke Command / API Method
                                   v
+-----------------------------------------------------------------------+
|       bd-cli-core-api (CLI & API Controller Layer)                    |
|  - CLI Parser (Commander/Yargs) & Terminal Formatter                  |
|  - Core API Facade Class (SfvcsRepository)                            |
|  - Structured Error Taxonomy & Error Handler                          |
+-----------------------------------------------------------------------+
                                   | Delegate Business Operation
                                   v
+-----------------------------------------------------------------------+
|            Domain Service Layer (Commit, Merge, Diff, Sync, etc.)     |
+-----------------------------------------------------------------------+
```

---

# 3. CLI サブコマンド詳細仕様

全 15 個の主要サブコマンドとオプション定義:

1. **`sfvcs init [--bare] [directory]`**: リポジトリを初期化。
2. **`sfvcs status [--short]`**: ワーキングツリーとインデックスの変更状態を表示。
3. **`sfvcs commit -m <message> [--author=<author>]`**: コミットを作成。
4. **`sfvcs diff [<commit1>] [<commit2>]`**: 構造差分を表示。
5. **`sfvcs merge <branch>`**: 3-Way Structural Merge を実行。
6. **`sfvcs rebase <upstream>`**: リビジョン系列の付け替え。
7. **`sfvcs cherry-pick <commit>`**: 個別コミットの適用。
8. **`sfvcs stash [save|pop|list]`**: 作業の退避・復元。
9. **`sfvcs revert <commit>`**: コミット打ち消し。
10. **`sfvcs clone / fetch / push`**: リモート同期。
11. **`sfvcs bisect [start|good|bad|reset]`**: バグ探査。
12. **`sfvcs blame <file>`**: 行・トークン追跡。
13. **`sfvcs submodule [add|update|status]`**: サブモジュール操作。
14. **`sfvcs fsck [--full]`**: 整合性検証。
15. **`sfvcs repack / gc`**: リポジトリ最適化・GC。

---

# 4. Core API (TypeScript / Node.js API Facade)

外部 TypeScript プログラムから sfvcs をライブラリとして組み込むための ファサードクラス `SfvcsRepository` を提供する。

```typescript
export class SfvcsRepository {
  public static async init(path: string, options?: InitOptionsModel): Promise<SfvcsRepository>;
  public static async open(path: string, options?: OpenOptionsModel): Promise<SfvcsRepository>;

  public async getStatus(): Promise<StatusResultModel>;
  public async commit(message: string, options?: CommitOptionsModel): Promise<Uint8Array>;
  public async merge(targetBranch: string): Promise<MergeResultModel>;
  public async diff(oldTreeCid?: Uint8Array, newTreeCid?: Uint8Array): Promise<DiffResultModel>;
}
```

---

# 5. エラー構造と エラータイプ仕様 (Error Taxonomy)

すべての sfvcs エラーは共通の基底クラス `SfvcsError` を継承し、固有のエラーコード (`code`) と機械読み取り可能なメタデータを持つ。

```typescript
export class SfvcsError extends Error {
  public readonly code: string;
  public readonly details?: Record<string, unknown>;

  constructor(code: string, message: string, details?: Record<string, unknown>) {
    super(message);
    this.code = code;
    this.details = details;
  }
}
```

| エラーコード           | 名称                     | 説明                              |
| ---------------------- | ------------------------ | --------------------------------- |
| `ERR_NOT_A_REPOSITORY` | 非リポジトリエラー       | `.sfvcs` ディレクトリが存在しない |
| `ERR_CORRUPTED_OBJECT` | オブジェクト破損エラー   | BLAKE3 ハッシュが不一致           |
| `ERR_MERGE_CONFLICT`   | マージコンフリクト       | 自動解決不能な競合が発生          |
| `ERR_REF_LOCKED`       | 参照ロックエラー         | CAS または Lock 獲得失敗          |
| `ERR_SUBMODULE_CYCLE`  | サブモジュール循環エラー | サブモジュール参照が循環          |
