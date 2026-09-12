# sfvcs 3-Way Structural Merge ・ CRDT 競合解決詳細仕様書

本書は `sfvcs` における 3-Way 構造マージ（3-Way Structural Merge）アルゴリズム、Prolly Tree / Sequence Tree の並行変更統合処理、CRDT 研究（Kleppmann et al., 2021 等）に基づく Tree Move 競合（閉環・重複移動）の自動解決仕様、コンフリクト表現形式、および Rebase / Cherry-Pick / Stash / Revert への応用手順を定義する詳細仕様書である。

---

# 1. 概要と基本原理

`sfvcs` におけるマージは、共通の祖先（Base）、現在のブランチ（Ours）、および合流対象のブランチ（Theirs）の 3 つのスナップショットを比較・統合する **3-Way Structural Merge** を基本とする。

## 1.1 階層的 Fast-Path スキップ
すべてのオブジェクトが Content ID (CID) を持つ不変構造であることを利用し、マージ処理は階層（Directory Node -> File Node -> Sequence Node -> Chunk）ごとに以下の $O(1)$ Fast-Path 判定を適用する。

| 条件                                                                               | 判定結果           | 適用アクション                                  | 計算量   |
| ---------------------------------------------------------------------------------- | ------------------ | ----------------------------------------------- | -------- |
| `Ours_CID == Theirs_CID`                                                           | 一致 (No conflict) | `Ours_CID` (または `Theirs_CID`) をそのまま採用 | $O(1)$   |
| `Base_CID == Ours_CID` かつ `Base_CID != Theirs_CID`                               | Theirs のみ変更    | `Theirs_CID` を採用                             | $O(1)$   |
| `Base_CID != Ours_CID` かつ `Base_CID == Theirs_CID`                               | Ours のみ変更      | `Ours_CID` を採用                               | $O(1)$   |
| `Base_CID != Ours_CID` かつ `Base_CID != Theirs_CID` かつ `Ours_CID != Theirs_CID` | 双方が異なって変更 | 下位ノードへ解像度を下げて再帰降下比較          | 下位走査 |

---

# 2. Directory Tree & File Level 3-Way Merge

ディレクトリエントリ（ファイル・サブディレクトリ）レベルでの 3-way マージ処理フロー。

## 2.1 エントリ変更マトリクス

| Base エントリ | Ours エントリ | Theirs エントリ     | 結果アクション            | 競合 (Conflict)         |
| ------------- | ------------- | ------------------- | ------------------------- | ----------------------- |
| A             | A             | B                   | B を採用                  | なし                    |
| A             | B             | A                   | B を採用                  | なし                    |
| A             | B             | B                   | B を採用 (同一変更)       | なし                    |
| A             | Deleted       | A                   | 削除 (Deleted)            | なし                    |
| A             | A             | Deleted             | 削除 (Deleted)            | なし                    |
| A             | B (Modified)  | Deleted             | **Modify/Delete 競合**    | あり                    |
| A             | Deleted       | C (Modified)        | **Delete/Modify 競合**    | あり                    |
| A             | B (Modified)  | C (Modified)        | 3-way Sequence Merge 実行 | 内容次第                |
| None          | B (Added)     | None                | B を追加                  | なし                    |
| None          | None          | C (Added)           | C を追加                  | なし                    |
| None          | B (Added)     | C (Added, 同一パス) | **Add/Add 同一パス競合**  | あり (`B.CID != C.CID`) |

---

# 3. Prolly Tree / Sequence Tree 3-Way Structural Merge

ファイル内容を表現する Sequence Tree (Prolly Tree) の 3-way マージアルゴリズム。

## 3.1 アルゴリズム概要
1. Base, Ours, Theirs の 3 つの Sequence Node を用意。
2. Exact Anchor (3 者に共通して存在する子ノード CID) を境界としてシーケンスを区画（Chunk Region）に分割。
3. 非一致区間（Gap）について、以下のルールで統合処理を行う：
   - Base から Ours のみが変更された区間 $\rightarrow$ Ours の変更を採用。
   - Base から Theirs のみが変更された区間 $\rightarrow$ Theirs の変更を採用。
   - Base から Ours と Theirs の双方が異なる変更を加えた区間 $\rightarrow$ バイト/行レベルの 3-way 比較を行い、重複のない連続した変更であれば自動結合。編集範囲が重なる場合は **Text Conflict Marker** を生成。

---

# 4. Tree Move 競合と CRDT アルゴリズム (Kleppmann et al.)

ツリー構造におけるディレクトリ・ファイルの移動（Move/Rename）処理が並行して発生した場合、循環参照（閉環: Cycle）や孤立ノードが発生する危険がある。`sfvcs` では Kleppmann et al. (2021) 「*A highly-available move operation for replicated trees*」の研究モデルを応用して確定的に判定・解決する。

## 4.1 閉環競合 (Cycle Conflict) と解決
- **発生シナリオ**:
  - Base: ディレクトリ `/A/` および `/B/` が存在。
  - Ours: ディレクトリ `/A` を `/B/` の下に移動 ($\text{Move}(A \to B/A)$)。
  - Theirs: ディレクトリ `/B` を `/A/` の下に移動 ($\text{Move}(B \to A/B)$)。
  - 単純マージすると `/A` が `/B` の子になり、`/B` が `/A` の子になる閉環（ルックアップ不能・無限ループ状態）が生成される。
- **自動解決アルゴリズム (Deterministic Cycle Resolution: Kleppmann et al., 2021)**:
  1. **グラフ構築と祖先パスチェック**:
     ディレクトリツリーの再構築時、提案された全 Move 操作による親ノード対 `(Child, Parent)` から移動移動グラフを維持。移動先ノード $P$ が移動対象ノード $C$ の子孫構造内に含まれていないか（$P \in \text{Descendants}(C)$）を祖先パス探索関数 `is_ancestor(C, P)` により全数チェック。
  2. **決定論的タイブレーキング (Deterministic Tie-Breaking)**:
     閉環が検出された場合、レプリカ間の一貫性を保証するため、決定論的優先度関数 $\text{Priority}(\text{Op})$ を用いて衝突した Move 操作の優先度を評価する。
     $$\text{Priority}(\text{Op}) = \text{Commit\_Timestamp} || \text{Commit\_CID\_Bytes}$$
     - **勝者 (Winner)**: $\text{Priority}$ の値が大きい（最新のタイムスタンプ、または辞書順で大きい CID）側の Move 操作をそのまま採用。
     - **敗者 (Losing Fallback)**: 優先度の低い側の Move 操作は自動的にキャンセルし、該当ディレクトリを **Base 状態の元の親ディレクトリ配下（または安全なフォールバック位置）へ安全に復帰** させる。
  3. **コンフリクト通知**:
     フォールバックが発生した場合は自動的に警告メッセージを `.sfvcs/MERGE_MSG` に記録し、`sfvcs status` でユーザーに非破壊的な補正が適用されたことを明示する。

## 4.2 重複移動競合 (Double Move Conflict)
- **発生シナリオ**:
  - Base: ファイル `file.txt` が `/dir1/` に存在。
  - Ours: `file.txt` を `/dir2/` に移動。
  - Theirs: `file.txt` を `/dir3/` に移動。
- **解決ルール**:
  - 内容が同一である場合、最後書き込み優先（LWW: Last-Write-Wins based on Commit Timestamp/CID）により移動先を確定するか、`/dir2/file.txt` と `/dir3/file.txt` の双方は配置せずユーザーに競合選択を促す。

---

# 5. コンフリクト表現形式

## 5.1 テキストコンフリクトマーカー (In-file Conflict Markers)
テキストファイル内容の衝突時、ファイルデータ内に以下の標準マーカーを挿入して保持する。

```
<<<<<<< OURS (main)
const timeout = 5000;
=======
const timeout = 10000;
>>>>>>> THEIRS (feature/timeout)
```

## 5.2 構造コンフリクトファイルおよび状態永続化ファイル (State Machine Files)
マージ、リベース、チェリーピック、チェンジ退避等の中断・コンフリクト未完了状態の永続化およびステートマシン制御は、`.sfvcs/` 配下の以下の専用状態ファイルによって管理される。

| ファイルパス              | 役割・格納データ                                                                     | 削除・クリーンアップタイミング                      |
| ------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------- |
| `.sfvcs/MERGE_HEAD`       | マージ対象の Theirs Commit CID (33bytes)                                             | `sfvcs commit` または `sfvcs merge --abort` 完了時  |
| `.sfvcs/MERGE_MSG`        | マージコミット用メッセージ案（コンフリクト一覧および Tree Move 解決ログ含む）        | マージコミット完了時                                |
| `.sfvcs/REBASE_HEAD`      | Rebase 中の元の HEAD Commit CID                                                      | `sfvcs rebase --continue` 完了時または `--abort` 時 |
| `.sfvcs/rebase-merge/`    | Rebase の進行状態データ（適用コミットキュー、現在のパッチ index、`onto` Commit CID） | Rebase シーケンス完了時                             |
| `.sfvcs/CHERRY_PICK_HEAD` | 現在 Cherry-Pick 実行中の Commit CID                                                 | Cherry-Pick 完了時または `--abort` 時               |
| `.sfvcs/REVERT_HEAD`      | 現在 Revert 実行中の Commit CID                                                      | Revert 完了時または `--abort` 時                    |
| `.sfvcs/STASH_DIR`        | Stash スタックオブジェクトのデータディレクトリ                                       | `sfvcs stash drop` または `clear` 実行時            |

---

# 6. Rebase, Cherry-Pick, Stash, Revert への 3-Way Merge 応用

歴史再構築および退避コマンド群の内部アルゴリズム仕様。

## 6.1 Rebase (`sfvcs rebase <upstream>`)
1. 現在のブランチ独自のコミット列 $C_1, C_2, \dots, C_k$ を特定（$C_1$ の親は Base $B$）。
2. 作業ツリーを `<upstream>` コミット $U_0$ へチェックアウト。
3. $i = 1 \dots k$ について順次パッチ適用：
   - 3-Way Merge 実行: `Base = Parent(C_i)`, `Ours = Head`, `Theirs = C_i`
   - コンフリクトなければ新規コミット $C_i'$ を作成し `Head = C_i'` と更新。

## 6.2 Cherry-Pick (`sfvcs cherry-pick <commit>`)
1. 対象コミット $C$ とその親コミット $P = \text{Parent}(C)$ を特定。
2. 3-Way Merge 実行: `Base = P`, `Ours = HEAD`, `Theirs = C`。
3. マージ結果を作業ツリーに適用し、新しいコミットを作成。

## 6.3 Stash (`sfvcs stash [save|pop]`)
1. `save`: 現在の `HEAD` コミット、インデックス状態、および作業ツリー未コミット変更から一時コミットオブジェクト（Stash Commit）を非表示参照 `refs/stash` に作成し、作業ツリーをクリーン化。
2. `pop`: `refs/stash` のコミットから 3-Way Merge 実行（`Base = Stash_Parent`, `Ours = Current_HEAD`, `Theirs = Stash_Commit`）により適用。

## 6.4 Revert (`sfvcs revert <commit>`)
1. 打消し対象コミット $C$ とその親コミット $P = \text{Parent}(C)$ を特定。
2. 逆向き 3-Way Merge 実行: `Base = C`, `Ours = HEAD`, `Theirs = P`。
3. マージ結果を新コミット（Revert Commit）として記録。

---

# 7. 汎用言語非依存セマンティック 3-Way Structural Merge (JSON/YAML/Lockfile/TOML)

設定ファイルや依存関係ロックファイル (`package-lock.json`, `Cargo.lock` 等) の行単位差分による競合を防止するため、プログラミング言語に依存しない抽象構造ツリー（Abstract Structure Tree: Key-Value Map / Sequence List / Scalar）に対するセマンティック 3-Way マージアルゴリズムを定義する。

## 7.1 キーバリューマップ (Key-Value Map) マージルール
Base, Ours, Theirs のマップ構造 $M_{\text{base}}, M_{\text{ours}}, M_{\text{theirs}}$ の各キー $k$ について：
- $k \in M_{\text{ours}}$ かつ $k \notin M_{\text{theirs}}$（Ours 追加・Theirs なし） $\rightarrow$ $M_{\text{result}}[k] = M_{\text{ours}}[k]$
- $k \notin M_{\text{ours}}$ かつ $k \in M_{\text{theirs}}$（Theirs 追加・Ours なし） $\rightarrow$ $M_{\text{result}}[k] = M_{\text{theirs}}[k]$
- $M_{\text{ours}}[k] == M_{\text{base}}[k]$ かつ $M_{\text{theirs}}[k] != M_{\text{base}}[k]$ $\rightarrow$ $M_{\text{result}}[k] = M_{\text{theirs}}[k]$
- $M_{\text{ours}}[k] != M_{\text{base}}[k]$ かつ $M_{\text{theirs}}[k] == M_{\text{base}}[k]$ $\rightarrow$ $M_{\text{result}}[k] = M_{\text{ours}}[k]$
- $M_{\text{ours}}[k] != M_{\text{theirs}}[k]$（異なって変更） $\rightarrow$ 下位再帰 3-Way マージ実行（値がスカラの場合は Structural Conflict）

---

# 8. Fugue Sequence CRDT リアルタイム協調編集アライメント

スナップショット生成前のリアルタイム並行編集（Web IDE 協調等）において、Prolly Tree Sequence Node の子要素列と Fugue CRDT の可逆状態ベクトルを統合し、非インタラクティブな確定アライメントを実現する。

---

# 9. Concurrent Move / Delete 競合解法 (移動と削除の平行衝突)

一方のブランチでファイル/ディレクトリが移動 ($\text{Move}(A \to B)$) され、他方のブランチで同一ファイル/ディレクトリが削除 ($\text{Delete}(A)$) された場合の決定論的マージ競合解決仕様。

## 9.1 優先ルールと対話型/自動フォールバックメカニズム
1. **決定論的デフォルト方針 (Move-Preserved Fallback)**:
   データ消失（Data Loss）の危険を最小化するため、非対話型マージ（自動 CI / ロボットマージ）においては **移動 ($\text{Move}$) 側の操作を優先** し、移動先パス $B$ にオブジェクトを維持保存する。
2. **コンフリクトステータス記録**:
   `.sfvcs/MERGE_MSG` に `CONFLICT (rename/delete): A moved to B in Ours, deleted in Theirs.` の警告を記録し、`sfvcs status` でコンフリクト状態としてユーザーによる明示的確認または `--theirs` (削除採用) オプションでの上書きを許容する。
