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
  "commit": "013a4f8901234567890123456789012345678901234567890123456789012345",
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
  "old_root": "01a1b2...",
  "new_root": "01c3d4...",
  "changes": [
    {
      "type": "MOVE",
      "old_path": "src/utils.ts",
      "new_path": "src/common/utils.ts",
      "confidence": 1.0,
      "subtree_cid": "01e5f6..."
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

## 1.5 `sfvcs fsck`
リポジトリ内の全オブジェクトの暗号学的整合性および参照構造を完全検証する。

- **構文**: `sfvcs fsck [options]`
- **動作**:
  - すべてのオブジェクトバイナリをロードし、`Actual CID == Expected CID` を計算・検証。
  - 全 Directory Node, Sequence Node, Commit の参照先 CID の存在確認。
  - 破損が検出された場合、エラーリストと検出 CID を出力し終了コード `1` を返す。

---

## 1.6 `sfvcs repack` / `sfvcs gc`
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

  /** Multi-resolution Diff 実行 */
  async diff(oldCommitOrTree: Uint8Array, newCommitOrTree: Uint8Array, options?: DiffOptions): Promise<DiffResult>;

  /** 整合性検査 */
  async fsck(): Promise<FsckReport>;
}
```

---

# 3. エラー構造とエラータイプ仕様

すべてのシステムエラーは、以下の共通基底クラス `SfvcsError` を継承し、固有の `code` を付与する。

| エラーコード (code) | クラス名 | 発生条件 | 対処方法 |
|---|---|---|---|
| `ERR_MISSING_OBJECT` | `MissingObjectError` | 指定された CID が objects (loose/pack) 内に存在しない | `fsck` で影響範囲確認、リモート等から再取得 |
| `ERR_CORRUPT_OBJECT` | `CorruptObjectError` | オブジェクトの計算 CID が記載 CID と不一致 | バックアップからの復元、またはオブジェクト破棄 |
| `ERR_INVALID_CANONICAL_FORMAT` | `InvalidFormatError` | ディレクトリエントリ未ソート、または未知の Format Version | 正しい仕様でエンコードされたオブジェクトへ更新 |
| `ERR_CONCURRENT_UPDATE` | `ConcurrentUpdateError` | CAS 参照更新時に他プロセスとの書き込み競合を検出 | リトライ処理の実行 |
| `ERR_REPOSITORY_LOCKED` | `RepositoryLockedError` | `.sfvcs/locks/` 内にロックファイルが存在 | 他プロセスの終了待機、または不要ロック削除 |
