# sfvcs 3-Way Structural Merge ・ CRDT 競合解決詳細仕様書

本書は `sfvcs` における 3-Way 構造マージ（3-Way Structural Merge）アルゴリズム、Prolly Tree / Sequence Tree の並行変更統合処理、CRDT 研究（Kleppmann et al., 2021 等）に基づく Tree Move 競合（閉環・重複移動）の自動解決仕様、コンフリクト表現形式、および Rebase / Cherry-Pick / Stash / Revert への応用手順を定義する詳細仕様書である。

---

# 1. 概要と基本原理

`sfvcs` におけるマージは、共通の祖先（Base）、現在のブランチ（Ours）、および合流対象のブランチ（Theirs）の 3 つのスナップショットを比較・統合する **3-Way Structural Merge** を基本とする。

## 1.1 階層的 Fast-Path スキップ
すべてのオブジェクトが Content ID (CID) を持つ不変構造であることを利用し、マージ処理は階層（Directory Node -> File Node -> Sequence Node -> Chunk）ごとに以下の $O(1)$ Fast-Path 判定を適用する。

| 条件 | 判定結果 | 適用アクション | 計算量 |
|---|---|---|---|
| `Ours_CID == Theirs_CID` | 一致 (No conflict) | `Ours_CID` (または `Theirs_CID`) をそのまま採用 | $O(1)$ |
| `Base_CID == Ours_CID` かつ `Base_CID != Theirs_CID` | Theirs のみ変更 | `Theirs_CID` を採用 | $O(1)$ |
| `Base_CID != Ours_CID` かつ `Base_CID == Theirs_CID` | Ours のみ変更 | `Ours_CID` を採用 | $O(1)$ |
| `Base_CID != Ours_CID` かつ `Base_CID != Theirs_CID` かつ `Ours_CID != Theirs_CID` | 双方が異なって変更 | 下位ノードへ解像度を下げて再帰降下比較 | 下位走査 |

---

# 2. Directory Tree & File Level 3-Way Merge

ディレクトリエントリ（ファイル・サブディレクトリ）レベルでの 3-way マージ処理フロー。

## 2.1 エントリ変更マトリクス

| Base エントリ | Ours エントリ | Theirs エントリ | 結果アクション | 競合 (Conflict) |
|---|---|---|---|---|
| A | A | B | B を採用 | なし |
| A | B | A | B を採用 | なし |
| A | B | B | B を採用 (同一変更) | なし |
| A | Deleted | A | 削除 (Deleted) | なし |
| A | A | Deleted | 削除 (Deleted) | なし |
| A | B (Modified) | Deleted | **Modify/Delete 競合** | あり |
| A | Deleted | C (Modified) | **Delete/Modify 競合** | あり |
| A | B (Modified) | C (Modified) | 3-way Sequence Merge 実行 | 内容次第 |
| None | B (Added) | None | B を追加 | なし |
| None | None | C (Added) | C を追加 | なし |
| None | B (Added) | C (Added, 同一パス) | **Add/Add 同一パス競合** | あり (`B.CID != C.CID`) |

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
  - Base: ディレクトリ `/A/B/` が存在。
  - Ours: `/A` を `/A/B/` の下に移動 ($\text{Move}(A \to B/A)$)。
  - Theirs: `/B` を `/A/` の下に移動 ($\text{Move}(B \to A/B)$)。
  - 単純マージすると `/A` が `/B` の子になり、`/B` が `/A` の子になる閉環（ルックアップ不可能状態）が生成される。
- **自動解決アルゴリズム (Deterministic Cycle Resolution)**:
  1. マージ処理中に祖先パス探索アルゴリズム (`detect_ancestor_cycle`) を実行。
  2. 閉環が検出された場合、決定論的タイブレーキングルールを適用：
     - **ルール**: コミットハッシュ値 (CID) が大きい側（またはタイムスタンプが最新の側）の Move 操作を優先採用し、他方の Move 操作を元の親位置（Base の親）に戻す（またはコンフリクトとして記録）。

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

## 5.2 構造コンフリクトファイル (Structural Conflict State)
マージ未完了状態の情報は `.sfvcs/state/MERGE_HEAD`, `.sfvcs/state/MERGE_RR`, および `.sfvcs/index` 内の Stage 情報 (Stage 1: Base, Stage 2: Ours, Stage 3: Theirs) として記録される。

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
