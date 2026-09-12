# sfvcs CLI コマンド・Core API・エラー処理詳細仕様書

本書は `sfvcs` における CLI（Command Line Interface）のサブコマンド仕様、コマンドライン引数、オプション、機械可読な JSON 出力フォーマット、Core JavaScript/TypeScript API、およびエラー分類・処理手順を定義する仕様書である。

---

# 1. CLI サブコマンド仕様

すべてのコマンドは、リポジトリルートまたは配下のサブディレクトリで実行可能とする。

## 1.1 `sfvcs init`
新しい sfvcs リポジトリを初期化する。

- **構文**: `sfvcs init [directory]`
- **オプション**:
  - `--default-branch <name>`: 初期ブランチ名指定（デフォルト: `main`）
  - `--hash <algo>`: CID ハッシュアルゴリズム指定 (`blake3` | `sha256`, デフォルト: `blake3`)
- **動作**:
  - 対象ディレクトリ配下に `.sfvcs/` ディレクトリ構造を作成。
  - 初期化完了メッセージを出力。

---

## 1.2 `sfvcs status`
ワークツリーおよびステージング状態（または直近スナップショットとの差分）を表示する。

- **構文**: `sfvcs status [options]`
- **オプション**:
  - `--json`: 機械可読な JSON フォーマットで出力。
- **JSON 出力フォーマット (`sfvcs status --json`)**:
```json
{
  "branch": "main",
  "commit": "023a4f8901234567890123456789012345678901234567890123456789012345",
  "untracked": ["newfile.txt"],
  "modified": ["src/index.ts"],
  "deleted": ["old.txt"],
  "renamed": [
    { "from": "docs/readme.txt", "to": "README.md", "confidence": 1.0 }
  ]
}
```

---

## 1.3 `sfvcs commit`
ワークツリーの現在の状態から新しいスナップショットとコミットを作成する。

- **構文**: `sfvcs commit -m <message> [options]`
- **オプション**:
  - `-m, --message <msg>`: コミットメッセージ（必須）
  - `--author <name <email>>`: 作成者情報の上書き
  - `--sign`: コミットへの暗号学的電子署名（Ed25519/SSH/GPG）を付与。
- **動作**:
  1. ワークツリーをスキャンし、変更ファイルを FastCDC / Sequence Tree 化。
  2. 新しい Directory Root を構築。
  3. Commit オブジェクトを作成し、アトミックに `refs/heads/<current_branch>` を更新。

---

## 1.4 `sfvcs diff`
二つのコミット、またはワークツリーとコミット間の Multi-resolution 構造差分を表示する。

- **構文**: `sfvcs diff [<old-cid-or-ref>] [<new-cid-or-ref>] [options]`
- **オプション**:
  - `--json`: 内部構造差分ツリーを JSON 形式で出力。
  - `--stat`: ファイル変更統計（挿入/削除バイト数）のみ表示。
  - `--no-renames`: Move / Rename / Copy 候補の自動抽出を無効化。
- **JSON 出力フォーマット (`sfvcs diff --json`)**:
```json
{
  "old_root": "02a1b2...",
  "new_root": "02c3d4...",
  "changes": [
    {
      "type": "MOVE",
      "old_path": "src/utils.ts",
      "new_path": "src/common/utils.ts",
      "confidence": 1.0,
      "subtree_cid": "02e5f6..."
    },
    {
      "type": "MODIFY",
      "path": "src/main.ts",
      "node_diff": {
        "changed_chunks": 2,
        "total_chunks": 10,
        "structural_similarity": 0.82
      }
    }
  ]
}
```

---

## 1.5 `sfvcs merge`
指定ブランチまたはコミットを現在のブランチへ 3-Way Structural Merge 合流する。

- **構文**: `sfvcs merge <commit-or-branch> [options]`
- **オプション**:
  - `--no-ff`: Fast-forward 可能な場合でも必ずマージコミットを作成。
  - `--abort`: 進行中の競合マージ処理を中断し元の状態に復元。

---

## 1.6 `sfvcs rebase`
現在のブランチのコミット列を指定上流コミットの上へ再配置する。

- **構文**: `sfvcs rebase <upstream> [options]`
- **オプション**:
  - `--continue`: 競合解消後に rebase 処理を再開。
  - `--abort`: rebase 処理を中断し元の状態に復元。

---

## 1.7 `sfvcs cherry-pick`
指定コミットの変更を現在の HEAD へ適用する。

- **構文**: `sfvcs cherry-pick <commit>`

---

## 1.8 `sfvcs stash`
未コミットの変更作業を一時退避・復元する。

- **構文**: `sfvcs stash [push|pop|apply|list|drop]`

---

## 1.9 `sfvcs revert`
指定コミットの変更を打ち消す新しいコミットを作成する。

- **構文**: `sfvcs revert <commit>`

---

## 1.10 `sfvcs clone` / `sfvcs fetch` / `sfvcs push`
リモートリポジトリとの同期を行う。

- **構文**:
  - `sfvcs clone <url> [directory] [--depth=<n>] [--sparse]`
  - `sfvcs fetch [<remote>]`
  - `sfvcs push [<remote>] [<branch>]`

---

## 1.11 `sfvcs bisect`
二分探索によってバグの発生コミットを自動特定する。

- **構文**: `sfvcs bisect [start|bad|good|skip|reset|run]`
- **サブコマンド**:
  - `sfvcs bisect start [<bad> [<good>...]]`: bisect セッションの開始。
  - `sfvcs bisect bad [<commit>]`: 指定コミットを問題あり（Bad）とマーク。
  - `sfvcs bisect good [<commit>]`: 指定コミットを問題なし（Good）とマーク。
  - `sfvcs bisect skip [<commit>]`: テスト不可能等のコミットをスキップ。
  - `sfvcs bisect reset`: bisect を終了し元の HEAD ブランチへ復帰。
  - `sfvcs bisect run <cmd> [args...]`: テストスクリプト（終了コード 0: Good, 1-127: Bad）を自動実行して判定。

---

## 1.12 `sfvcs blame`
ファイルの各行に対する最終変更コミット、著者、タイムスタンプを表示する。

- **構文**: `sfvcs blame <file> [options]`
- **オプション**:
  - `-L <start>,<end>`: 指定行範囲（例: `-L 10,25`）のみ追跡。
  - `-M`: 同一ファイル内の移動・コピー行を追跡。
  - `-C`: 他ファイルからの移動・コピー行（Winnowing Fingerprint アライメント）を追跡。
  - `--json`: 機械可読な JSON 形式でプロバナンス情報を出力。

---

## 1.13 `sfvcs submodule`
ネストされた外部/内部リポジトリ (`ENTRY_SUBMODULE`: `0x04`) を管理する。

- **構文**: `sfvcs submodule [add|status|init|update|sync]`
- **サブコマンド・オプション**:
  - `add <url> [<path>]`: サブモジュールを追加。
  - `status`: 各サブモジュールのコミット不一致・dirty 状態を表示。
  - `update [--recursive]`: 親リポジトリに記録された CID へサブモジュールを同期更新。

---

## 1.14 `sfvcs fsck`
リポジトリ内の全オブジェクトの暗号学的整合性および参照構造を完全検証する。

- **構文**: `sfvcs fsck [options]`
- **動作**:
  - すべてのオブジェクトバイナリをロードし、`Actual CID == Expected CID` を計算・検証。
  - 全 Directory Node, Sequence Node, Commit の参照先 CID の存在確認。
  - 破損が検出された場合、エラーリストと検出 CID を出力し終了コード `1` を返す。

---

## 1.15 `sfvcs repack` / `sfvcs gc`
ルーズオブジェクトをパックファイルに集約し、不要オブジェクトを掃除する。

- **構文**: `sfvcs gc [options]`
- **オプション**:
  - `--prune=<period>`: 指定期間以前の未到達オブジェクトを完全削除（デフォルト: `24h`）。

---

# 2. Core API (TypeScript / Node.js)

`packages/core` としてライブラリ利用可能な API インターフェース仕様。

```typescript
export interface RepositoryOptions {
  repoPath: string;
}

export class Repository {
  static async init(path: string, options?: InitOptions): Promise<Repository>;
  static async open(path: string): Promise<Repository>;

  /** オブジェクトの低レイヤ読み書き */
  async readObject(cid: Uint8Array): Promise<SfvcsObject>;
  async writeObject(obj: SfvcsObject): Promise<Uint8Array>; // returns CID

  /** コミット・スナップショット作成 */
  async createCommit(options: CommitOptions): Promise<Uint8Array>;

  /** 3-Way Structural Merge */
  async merge(targetCommit: Uint8Array, options?: MergeOptions): Promise<MergeResult>;

  /** 履歴再構築操作 */
  async rebase(upstreamCommit: Uint8Array): Promise<RebaseResult>;
  async cherryPick(commitCid: Uint8Array): Promise<CherryPickResult>;
  async stash(action: 'push' | 'pop' | 'apply'): Promise<StashResult>;
  async revert(commitCid: Uint8Array): Promise<RevertResult>;

  /** リモート同期 */
  async fetch(remoteName: string): Promise<FetchResult>;
  async push(remoteName: string, branchName: string): Promise<PushResult>;

  /** Multi-resolution Diff 実行 */
  async diff(oldCommitOrTree: Uint8Array, newCommitOrTree: Uint8Array, options?: DiffOptions): Promise<DiffResult>;

  /** Bisect 自動バグ特定 */
  async bisect(action: 'start' | 'bad' | 'good' | 'skip' | 'reset' | 'run', options?: BisectOptions): Promise<BisectResult>;

  /** Blame 行単位追跡 */
  async blame(filePath: string, options?: BlameOptions): Promise<BlameResult>;

  /** サブモジュール操作 */
  async submodule(action: 'add' | 'status' | 'update' | 'sync', options?: SubmoduleOptions): Promise<SubmoduleResult>;

  /** 整合性検査 */
  async fsck(): Promise<FsckReport>;
}
```

---

# 3. エラー構造とエラータイプ仕様

すべてのシステムエラーは、以下の共通基底クラス `SfvcsError` を継承し、固有の `code` を付与する。

| エラーコード (code)            | クラス名                | 発生条件                                                  | 対処方法                                       |
| ------------------------------ | ----------------------- | --------------------------------------------------------- | ---------------------------------------------- |
| `ERR_MISSING_OBJECT`           | `MissingObjectError`    | 指定された CID が objects (loose/pack) 内に存在しない     | `fsck` で影響範囲確認、リモート等から再取得    |
| `ERR_CORRUPT_OBJECT`           | `CorruptObjectError`    | オブジェクトの計算 CID が記載 CID と不一致                | バックアップからの復元、またはオブジェクト破棄 |
| `ERR_INVALID_CANONICAL_FORMAT` | `InvalidFormatError`    | ディレクトリエントリ未ソート、または未知の Format Version | 正しい仕様でエンコードされたオブジェクトへ更新 |
| `ERR_MERGE_CONFLICT`           | `MergeConflictError`    | 自動合流不可能なファイル競合を検出                        | コンフリクトマーカーの修正後 `commit`          |
| `ERR_CYCLE_DETECTED`           | `CycleDetectedError`    | Tree Move による循環参照移動を検出                        | 自動決定論解決またはユーザー手動指定           |
| `ERR_CONCURRENT_UPDATE`        | `ConcurrentUpdateError` | CAS 参照更新時に他プロセスとの書き込み競合を検出          | リトライ処理の実行                             |
| `ERR_REPOSITORY_LOCKED`        | `RepositoryLockedError` | `.sfvcs/locks/` 内にロックファイルが存在                  | 他プロセスの終了待機、または不要ロック削除     |
| `ERR_SUBMODULE_CYCLE`          | `SubmoduleCycleError`   | サブモジュール間の循環参照依存関係を検出                  | 設定ファイル内の URL / 階層構造修正            |
| `ERR_QUOTA_EXCEEDED`           | `QuotaExceededError`    | ブラウザストレージ (IndexedDB/OPFS) の容量超過            | Derived Cache のクリアまたはストレージ拡張     |
