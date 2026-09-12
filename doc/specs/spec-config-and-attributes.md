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

## 2.2 評価優先順位 (高い順)
1. CLI の `--exclude` 引数
2. `.sfvcsignore` 内のネストされた最新パターン
3. ルート `.sfvcsignore`
4. `.gitignore` ( `inheritGitignore: true` の場合 )
5. グローバル環境無視ファイル (`~/.sfvcsignore`)

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

| 属性キー | 設定値の例 | 説明 |
|---|---|---|
| `binary` | `binary` (フラグ) | バイナリファイルとして扱い、行単位 Diff をスキップ |
| `text` | `text` (フラグ) | テキストファイルとして強制扱い |
| `diff` | `diff=off` / `diff=custom` | Diff 生成の無効化またはカスタム Diff ドライバーの指定 |
| `cdc` | `cdc=large` / `cdc=small` | FastCDC のチャンクサイズを大容量資産向け等に個別変更 |
