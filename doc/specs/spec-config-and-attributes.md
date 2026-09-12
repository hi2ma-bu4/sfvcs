# sfvcs 設定ファイル・Ignore・属性詳細仕様書 (`.sfvcsconfig`, `.sfvcsignore`, `.sfvcsattributes`)

本書は `sfvcs` におけるリポジトリ設定ファイル（`.sfvcsconfig`）、無視パターンファイル（`.sfvcsignore`）、およびファイル属性定義（`.sfvcsattributes`）のフォーマットと挙動詳細を定義する仕様書である。

---

# 1. リポジトリ設定ファイル (`.sfvcsconfig`)

リポジトリおよびユーザー単位の動作パラメタを定義する設定ファイル。
フォーマットは **JSON5** または **INI 形式** をサポートする（ルート `.sfvcsconfig` ファイル）。

## 1.1 設定項目一覧

```json
{
  "core": {
    "caseSensitivity": "auto",       // "auto" | "sensitive" | "insensitive"
    "inheritGitignore": true,        // .gitignore パターンを自動的に継承して読み込む
    "racyFreeInspection": true,      // Racy Git 回避 (mtime 同値時の再ハッシュ検証) を有効化
    "defaultBranch": "main",
    "inlineThreshold": 1024          // インライン格納の閾値バイト数
  },
  "diff": {
    "renameThreshold": 0.65,         // Move/Rename 判定の最小信頼度スコア (0.0 - 1.0)
    "detectRenames": true,           // Move/Rename 候補の検出を有効化
    "detectDirectoryRenames": true,  // ディレクトリ一括移動 O(1) 検出の有効化
    "caseOnlyRename": true           // 大文字小文字のみのリネーム検出の有効化
  },
  "cdc": {
    "minSize": 2048,                 // FastCDC 最小チャンクサイズ (バイト)
    "avgSize": 8192,                 // FastCDC 平均ターゲットサイズ
    "maxSize": 65536                 // FastCDC 最大チャンクサイズ
  }
}
```

## 1.2 Git 設定の継承 (`inheritGitignore`)
`core.inheritGitignore: true`（デフォルト: `true`）が有効な場合、`sfvcs` は同一ディレクトリの `.gitignore` を自動的に読み込み、`.sfvcsignore` ルールとマージして適用する。

---

# 2. 無視パターンファイル (`.sfvcsignore`)

バージョン管理から除外するファイル・ディレクトリパターンを指定する。

## 2.1 パターン指定構文
構文は標準的な `.gitignore` 互換ルールに従う。

```gitignore
# コメント行
node_modules/
*.log
dist/
.env*

# 否定パターン (除外の例外)
!important.log
```

## 2.2 評価優先順位行列 (Precedence Evaluation Matrix)
ファイルシステム走査時、特定のパス $P$ が無視対象かどうかの判定は、ディレクトリ階層を降下しながら以下の **評価優先順位（1 が最高優先度）** に基づいて行われる。同一優先度内では、ファイルの下行に記述されたパターンが上行のパターンを上書きする。

| 優先度 | 設定ソース                                                         | 適用スコープ               | 補足説明                                     |
| ------ | ------------------------------------------------------------------ | -------------------------- | -------------------------------------------- |
| 1      | CLI コマンドライン引数 (`--exclude <pattern>`)                     | プロセス実行セッションのみ | 最優先で最上書き                             |
| 2      | ネストされた配下 `.sfvcsignore`                                    | 当該サブディレクトリ以下   | 最も対象ファイルに近いディレクトリのパターン |
| 3      | ネストされた配下 `.gitignore`                                      | 当該サブディレクトリ以下   | `inheritGitignore: true` 時のみ有効          |
| 4      | ルート `.sfvcsignore`                                              | リポジトリ全体             | リポジトリ標準の sfvcs 無視設定              |
| 5      | ルート `.gitignore`                                                | リポジトリ全体             | `inheritGitignore: true` 時のみ有効          |
| 6      | ローカル非共有ファイル (`.sfvcs/info/exclude`)                     | ローカルリポジトリのみ     | チーム共有されない個人用無視設定             |
| 7      | ユーザーグローバル設定 (`~/.sfvcsignore` / `~/.config/git/ignore`) | ユーザー環境全体           | 最底層デフォルト設定                         |

## 2.3 否定パターン (`!`) および親ディレクトリ除外評価アルゴリズム
否定パターン（例外許可: `!important.log`）の評価においては、パフォーマンスと正当性を両立させるため以下の親ディレクトリ除外ルールを厳格に適用する。

1. **親ディレクトリ除外時の無効化原則**:
   親ディレクトリ（例: `logs/`）自体が無視パターン（`logs/`）にマッチして無視された場合、その配下のサブファイルに対する否定パターン（`!logs/important.log`）は **効果を持たない**。性能向上のため、無視された親ディレクトリ配下のディスク走査自体がスキップされるためである。
2. **親ディレクトリ除外を解除する場合の手順**:
   親ディレクトリ内の特定のファイルのみを例外許可したい場合、親ディレクトリ自体を `logs/*` のようにワイルドカード指定で内容のみ無視し、否定パターンを指定しなければならない。
   ```gitignore
   # 正しい例外許可パターン
   logs/*
   !logs/important.log
   ```

---

# 3. ファイル属性ファイル (`.sfvcsattributes`)

パスごとの特殊な扱い（バイナリ/テキスト判定、差分ドライバー、CDC パラメタ上書き等）を制御する。

## 3.1 属性定義形式
```gitignore
# パターン                属性設定
*.png                    binary
*.pdf                    binary
*.min.js                 cdc=large
src/generated/**         diff=off
docs/*.md                text
```

## 3.2 定義可能な属性一覧

| 属性キー | 設定値の例                 | 説明                                                  |
| -------- | -------------------------- | ----------------------------------------------------- |
| `binary` | `binary` (フラグ)          | バイナリファイルとして扱い、行単位 Diff をスキップ    |
| `text`   | `text` (フラグ)            | テキストファイルとして強制扱い                        |
| `diff`   | `diff=off` / `diff=custom` | Diff 生成の無効化またはカスタム Diff ドライバーの指定 |
| `cdc`    | `cdc=large` / `cdc=small`  | FastCDC のチャンクサイズを大容量資産向け等に個別変更  |
