# sfvcs アルゴリズム詳細仕様書

本書は `sfvcs` における核心アルゴリズムの具体的な処理フロー、パラメタ定義、計算式、擬似コード、Sequence Tree 差分から行番号 (Line Offset / Unified Diff) への変換、並びに Sparse Checkout (部分チェックアウト) および Shallow History (浅い履歴) の検証ロジックを明確にする仕様書である。

対象アルゴリズム：
1. **Content-Defined Chunking (FastCDC & Gear Hash)**
2. **Prolly Tree / Persistent Sequence Tree の構築アルゴリズム**
3. **Multi-resolution Structural Diff エンジン**
4. **Sequence Tree Chunk Diff から行番号 (Unified Diff) への変換アルゴリズム**
5. **Move / Rename / Copy 検出と Fingerprint 類似度検索（Winnowing）**
6. **Move / Rename / Case-only Rename 最適化アルゴリズム**
7. **Sparse Checkout (部分チェックアウト) 境界判定アルゴリズム**
8. **Shallow History (浅い履歴 clone) 境界判定および `fsck` 検証**

---

# 1. Content-Defined Chunking (FastCDC & Gear Hash)

`sfvcs` では、ファイルコンテンツの分割に **FastCDC** (Xia et al., 2016) アルゴリズムを採用する。

## 1.1 パラメータ仕様

| パラメータ名         | 変数名        | デフォルト値            | 設定可能範囲      | 説明                                                     |
| -------------------- | ------------- | ----------------------- | ----------------- | -------------------------------------------------------- |
| 最小チャンクサイズ   | `MIN_SIZE`    | **2,048 B (2 KiB)**     | 512 B 〜 8 KiB    | これ未満の境界検出をスキップ                             |
| ターゲットサイズ     | `AVG_SIZE`    | **8,192 B (8 KiB)**     | 2 KiB 〜 32 KiB   | 正規化境界判定の目標平均サイズ                           |
| 最大チャンクサイズ   | `MAX_SIZE`    | **65,536 B (64 KiB)**   | 16 KiB 〜 256 KiB | 強制切り出し閾値                                         |
| ローリングウィンドウ | `WINDOW_SIZE` | **48 バイト**           | 固定              | ハッシュ計算用ウィンドウ                                 |
| 正規化マスク1        | `MASK_S`      | `0x0000d90003510000ULL` | -                 | `MIN_SIZE` 〜 `AVG_SIZE` 用の厳格マスク (約13ビット立て) |
| 正規化マスク2        | `MASK_L`      | `0x0000d90000000000ULL` | -                 | `AVG_SIZE` 〜 `MAX_SIZE` 用の緩和マスク (約9ビット立て)  |

## 1.2 Gear Hash テーブル (256 x 64-bit uint)
Gear Hash は 256 個の 64 ビット擬似乱数表 `GEAR_TABLE` を用いて高速にローリングハッシュを計算する。

```python
# Gear Table (256個のランダム 64bit 整数)
GEAR_TABLE = [
    0x69b4a9283b320722, 0x0bc255140d34195b, 0x8484b80b7211e4f3, 0x6e545129653775db,
    # ... (256要素の固定ランダムテーブル)
]
```

## 1.3 FastCDC 境界検出擬似コード

```python
def find_chunk_boundary(buffer: bytes, offset: int, total_len: int) -> int:
    remaining = total_len - offset
    if remaining <= MIN_SIZE:
        return remaining

    max_len = min(remaining, MAX_SIZE)
    hash_val = 0
    
    # 1. MIN_SIZE までは境界検索を完全にスキップして高速化
    curr = offset + MIN_SIZE
    
    # 2. MIN_SIZE から AVG_SIZE までの探索 (厳格マスク MASK_S で小チャンク増大を抑制)
    avg_boundary = offset + AVG_SIZE
    if avg_boundary > offset + max_len:
        avg_boundary = offset + max_len

    while curr < avg_boundary:
        byte_val = buffer[curr]
        hash_val = ((hash_val << 1) + GEAR_TABLE[byte_val]) & 0xFFFFFFFFFFFFFFFF
        if (hash_val & MASK_S) == 0:
            return curr - offset + 1
        curr += 1

    # 3. AVG_SIZE から MAX_SIZE までの探索 (緩和マスク MASK_L で大チャンク長大化を抑制)
    max_boundary = offset + max_len
    while curr < max_boundary:
        byte_val = buffer[curr]
        hash_val = ((hash_val << 1) + GEAR_TABLE[byte_val]) & 0xFFFFFFFFFFFFFFFF
        if (hash_val & MASK_L) == 0:
            return curr - offset + 1
        curr += 1

    # 4. MAX_SIZE 到達時は強制分割
    return max_len
```

---

# 2. Sequence Tree (Prolly Tree) 構築アルゴリズム

Leaf Chunk の列から、Content-Defined な多重木（Sequence Tree）を再帰的に構築する。

## 2.1 内部ノードの境界判定ルール
内部ノード (Sequence Node) のグループ化境界は、子要素の CID ハッシュ値から計算する。

- 子ノードの目標ファンアウト: **64**
- 内部ノード判定マスク: `NODE_MASK = 0x3F` (下位6ビットがすべてゼロ: 1/64 の確率)

```python
def is_internal_node_boundary(child_cid_bytes: bytes) -> bool:
    # CID の最後の 4 バイトを uint32 ビット列として解釈
    val = read_uint32_be(child_cid_bytes, offset=29)
    return (val & 0x3F) == 0
```

## 2.2 ボトムアップストリーミング木構築擬似コード

```python
def build_sequence_tree(chunks: List[Chunk]) -> SequenceNode:
    if len(chunks) == 1:
        # 1チャンクのみの場合は単一のSequence Nodeで包む
        return create_sequence_node(children=[chunks[0]], is_leaf_children=True)

    current_level_children = chunks
    is_leaf = True

    while len(current_level_children) > 1:
        next_level_nodes = []
        group = []
        
        for child in current_level_children:
            group.append(child)
            # グループ境界判定 (または上限ファンアウト128超過時)
            if is_internal_node_boundary(child.cid) or len(group) >= 128:
                node = create_sequence_node(children=group, is_leaf_children=is_leaf)
                next_level_nodes.append(node)
                group = []

        if len(group) > 0:
            node = create_sequence_node(children=group, is_leaf_children=is_leaf)
            next_level_nodes.append(node)

        current_level_children = next_level_nodes
        is_leaf = False

    return current_level_children[0]
```

---

# 3. Multi-resolution Structural Diff エンジン

二つのスナップショット（またはツリーノード）を高速かつ高解像度に比較する。

## 3.1 解像度下降比較ステップ

1. **Exact CID Fast Path**:
   `Old_Node.CID == New_Node.CID` の場合、全サブツリーを **EQUAL** として比較処理を即座にスキップ ($O(1)$)。

2. **Node Type 判定**:
   - Directory vs Directory: エントリ名キーのソート済みディフ。
   - File vs File: メタデータ比較後、`Content Root CID` を比較。
   - Sequence vs Sequence: 子要素の Sequence Alignment 処理へ下降。
   - Chunk vs Chunk: バイトレベル Meyers Diff。

## 3.2 Sequence Alignment アルゴリズム

旧子要素列 $A = [a_1, a_2, \dots, a_m]$ と 新子要素列 $B = [b_1, b_2, \dots, b_n]$ の比較：

```python
def diff_sequence_nodes(node_A: SequenceNode, node_B: SequenceNode) -> DiffResult:
    if node_A.cid == node_B.cid:
        return DiffResult.Unchanged()

    # 1. Exact Child CID Match による アンカー（アンカーポイント）特定
    anchors = find_exact_cid_anchors(node_A.children_cids, node_B.children_cids)

    # 2. アンカー間のギャップ (非一致区間) のみを選択的に再帰比較
    diffs = []
    for gap in extract_gaps(anchors):
        if gap.is_exact_match:
            diffs.append(DiffResult.Unchanged(gap.cid))
        else:
            sub_diff = diff_unmatched_subtrees(gap.sub_A, gap.sub_B)
            diffs.append(sub_diff)

    return combine_diffs(diffs)
```

---

# 4. Sequence Tree 差分から行番号 (Unified Diff) への変換アルゴリズム

Sequence Tree の Chunk 単位の構造差分から、人間が可読な標準 Unified Diff 形式（`@@ -a,b +c,d @@`）と行番号 (Line Offset) を生成するアルゴリズム。

## 4.1 変換処理手順
1. **改行インデックス (Line Boundary Index) キャッシュ**:
   各 Chunk オブジェクト (`SFCK`) について、改行文字 `\n` (0x0A) の絶対バイトオフセット列を抽出・保持。
2. **Chunk 差分からバイトオフセット区間の確定**:
   Multi-resolution Diff によって特定された不一致 Chunk 区間の旧ファイル内バイト範囲 $[S_{\text{old}}, E_{\text{old}}]$ と新ファイル内バイト範囲 $[S_{\text{new}}, E_{\text{new}}]$ を決定。
3. **バイト範囲の行番号変換**:
   改行インデックステーブルを用いて、$S_{\text{old}}$ 直前の改行数から旧ファイル開始行番号 $a$、$E_{\text{old}}$ までの改行数から行数 $b$ を求める（新ファイル $c, d$ も同様）。
4. **Chunk 内 Meyers Line Diff 実行**:
   不一致 Chunk の該当バイト区間のみをメモリ上で改行分割し、標準的な Meyers Diff を実行して行レベルの `+` / `-` パッチ行を確定・整形出力する。

---

# 5. Move / Rename / Copy 検出と Fingerprint 類似度検索

## 5.1 Winnowing ベースの近似 Fingerprint 計算

ファイル・サブルーチンの類似度判定のため、**Winnowing** アルゴリズムによりスケッチ（Fingerprint）を生成する。

- **k-gram サイズ**: $k = 16$ バイト
- **ウィンドウサイズ**: $w = 32$ バイト

```python
def compute_winnowing_fingerprint(data: bytes) -> List[uint32]:
    # 境界値・小規模ファイルフォールバック処理
    if len(data) == 0:
        return []
    
    # データサイズが k-gram (16 bytes) 未満の場合はゼロパディングして1つのハッシュを生成
    if len(data) < 16:
        padded = data.ljust(16, b'\x00')
        return [gear_hash(padded)]

    hashes = [gear_hash(data[i:i+16]) for i in range(len(data) - 15)]
    
    # ハッシュ列の長さがウィンドウサイズ w (32) 未満の場合は全ハッシュ中の最小値を採択
    if len(hashes) < 32:
        return [min(hashes)]

    fingerprints = set()
    for i in range(len(hashes) - 31):
        window = hashes[i:i+32]
        min_val = min(window)
        fingerprints.add(min_val)

    return sorted(list(fingerprints))
```

## 5.2 小規模ファイル (Short Files) の Winnowing 計算フォールバック規約
Winnowing アルゴリズムは $k$-gram ($k=16$) およびウィンドウサイズ $w=32$（計約47バイト）以上のデータ長を想定している。47バイト未満の短小ファイルに対する例外・境界値処理ルールを以下のように定める。

1. **データ長 $N < 16$ バイトの場合**:
   末尾を `0x00` バイトで 16 バイトまでゼロパディングし、単一の Gear Hash を計算して要素数 1 の Fingerprint スケッチとする。
2. **データ長 $16 \le N < 47$ バイトの場合 ($k$-gram 数 $M < 32$)**:
   ウィンドウサイズを満たさないため、全 $k$-gram ハッシュ列の中から単一の最小ハッシュ値を選出して Fingerprint スケッチとする。
3. **Jaccard 類似度計算時のフォールバック**:
   比較対象の双方または一方が短小ファイルの場合、Winnowing Fingerprint の要素数が少なくなるため、内容の直接バイト比較（または CID 比較）へフォールバックして信頼度スコアを決定する。

---

# 6. Move / Rename / Case-only Rename 最適化アルゴリズム

## 6.1 $O(1)$ ディレクトリサブツリー一括リネーム (Directory Subtree Rename)

サブディレクトリ（例: `src/old_dir/` 以下の数千ファイル）が一括移動された場合、個別ファイル比較を一切行わず、$O(1)$ でディレクトリ単位のリネームとして検出する。

```python
def detect_directory_renames(old_tree: DirectoryNode, new_tree: DirectoryNode) -> List[RenameItem]:
    deleted_dirs = collect_deleted_directory_cids(old_tree, new_tree)
    added_dirs = collect_added_directory_cids(old_tree, new_tree)
    directory_renames = []
    
    for cid, old_dir_path in deleted_dirs.items():
        if cid in added_dirs:
            new_dir_path = added_dirs[cid]
            directory_renames.append(DirectoryRename(
                from_path=old_dir_path,
                to_path=new_dir_path,
                confidence=1.00,
                subtree_cid=cid
            ))
            skip_subtree_inspection(old_dir_path, new_dir_path)
            
    return directory_renames
```

## 6.2 大文字小文字のみの変更検出 (Case-only Rename Detection)

```python
def detect_case_only_renames(deleted_files: List[FileItem], added_files: List[FileItem]) -> List[RenameItem]:
    lowercase_map = { f.path.lower(): f for f in deleted_files }
    case_renames = []
    
    for added in added_files:
        added_lower = added.path.lower()
        if added_lower in lowercase_map:
            deleted = lowercase_map[added_lower]
            if deleted.path != added.path and deleted.content_cid == added.content_cid:
                case_renames.append(RenameItem(
                    type="CASE_ONLY_RENAME",
                    from_path=deleted.path,
                    to_path=added.path,
                    confidence=1.00
                ))
    return case_renames
```

## 6.3 複合信頼度スコア計算 (Composite Confidence Score)

一部内容が変更された移動 (`MOVE + MODIFY`) において、以下の 3 つの要素を重み付け加算して最終スコア $S \in [0.0, 1.0]$ を算出する。

$$S = w_1 \cdot \text{Similarity}_{\text{Content}} + w_2 \cdot \text{Similarity}_{\text{Path}} + w_3 \cdot \text{Context}_{\text{ParentDir}}$$

- **$w_1 = 0.60$**: Winnowing Fingerprint の Jaccard 類似度。
- **$w_2 = 0.25$**: ファイル名文字列の類似度 (Levenshtein 距離正規化値)。
- **$w_3 = 0.15$**: 親ディレクトリ構造・同層隣接ファイルの CID 一致率。

$S \ge \text{renameThreshold}$（デフォルト: **0.65**）の場合に `MOVE + MODIFY` 候補として採用する。

---

# 7. Sparse Checkout (部分チェックアウト) 境界判定アルゴリズム

リポジトリ全体の一部サブディレクトリのみを作業ツリーにチェックアウト・更新するアルゴリズム。

## 7.1 定義と判定ルール
- `.sfvcs/config` または `.sfvcs/sparse-checkout` 内にチェックアウト対象パスパターン（例: `src/core/*`）を定義。
- **Traversal Algorithm**:
  Directory Tree 下降時、ディレクトリエントリのパスが Sparse パターンにマッチしない場合、その `Directory Node` (`SFDR`) の配下走査および物理ディスクへの書き出しをスキップする。ただし、オブジェクトストア内には CID 参照を保持するため、コミット作成時に未チェックアウト領域のデータが破壊されることはない。

---

# 8. Shallow History (浅い履歴 clone) 境界判定および `fsck` 検証

コミット履歴の深さを指定（例: `--depth=1`）してクローン・取得するアルゴリズム。

## 8.1 Shallow Root (浅い境界) の記録
- 取得を中断した最古のコミット CID 群を `.sfvcs/shallow` ファイルに記録。

## 8.2 Shallow `fsck` 検証ルール
- `sfvcs fsck` 実行時、通常のコミットオブジェクト検証では全 Parent Commit CID の存在を必須とするが、`.sfvcs/shallow` に記載されたコミット CID については **「Parent Commit 非存在」を正常（Missing Parent Allowed）** として扱い、検証エラーを抑制する。

---

# 9. Bisect (二分探索バグ特定) アルゴリズム

有向非巡回グラフ (DAG) であるコミット履歴木から、問題を引き起こした最初のコミットを二分探索で効率的に特定するアルゴリズム。

## 9.1 中点 (Mid-point) 選定計算式
「Bad」と判定されたコミット $B$ と「Good」と判定されたコミット群 $G_1, G_2, \dots$ に対し、未検証の到達可能コミット集合 $U$ を抽出する。各コミット $c \in U$ について、その子孫数 $D(c)$ を計算し、以下の重み関数 $W(c)$ を最小化するコミット $c_{\text{mid}}$ を次の検証対象（Mid-point）として選定する。

$$W(c) = \left| D(c) - \frac{|U|}{2} \right|$$

```python
def find_bisect_midpoint(bad_commit: CID, good_commits: Set[CID], skipped_commits: Set[CID]) -> CID:
    # 1. Bad から到達可能で Good から到達不能な未検証コミット集合 U を抽出
    ancestors_bad = get_all_ancestors(bad_commit)
    ancestors_good = set()
    for g in good_commits:
        ancestors_good.update(get_all_ancestors(g))
    
    untested_candidates = ancestors_bad - ancestors_good - skipped_commits
    if not untested_candidates:
        return bad_commit

    total_count = len(untested_candidates)
    best_commit = None
    min_diff = float('inf')

    # 2. 各ノードの下位子孫数を計算し半数 (|U| / 2) に最も近いノードを選択
    for candidate in untested_candidates:
        descendant_count = count_descendants_in_set(candidate, untested_candidates)
        diff = abs(descendant_count - (total_count / 2))
        if diff < min_diff:
            min_diff = diff
            best_commit = candidate

    return best_commit
```

---

# 10. Blame (行単位履歴・著者追跡) アルゴリズム

ファイル内の全行に対し、該当行を最後に変更・追加したコミット CID、著者、タイムスタンプを算出するアルゴリズム。

## 10.1 Prolly Tree & Winnowing 指紋アライメントによる追跡
1. 対象コミット $C$ から親コミット $P$ へ逆方向に対向比較を実行。
2. `SFSQ` Sequence Tree アライメントにより不一致 Chunk 区間を特定。
3. ファイルがリネームまたは別ファイルへ移動している場合、第 5 節の Winnowing Fingerprint 類似度検索を実行して同一プロバナンス行として追跡を継続。
4. 親コミット $P$ に存在しない行をコミット $C$ の寄与として確定し、親 $P$ へ再帰下降。

---

# 11. Pathspec Trie ($O(K)$ プレフィックスツリー) 検索アルゴリズム

`.sfvcsignore` や `.sfvcsattributes` の大量のワイルドカードパターンに対し、走査パス $P$ を $O(K)$ ($K$ はパス文字列長) で高速マッチングする Prefix Trie アルゴリズム。

```python
class PathspecTrieNode:
    def __init__(self):
        self.children = {} # char -> PathspecTrieNode
        self.rules = []    # マッチするルール情報

def match_pathspec_trie(root: PathspecTrieNode, path: str) -> List[Rule]:
    current = root
    matched_rules = []
    
    for char in path:
        if char in current.children:
            current = current.children[char]
            if current.rules:
                matched_rules.extend(current.rules)
        else:
            break
            
    return matched_rules
```

---

# 12. サブモジュール (`ENTRY_SUBMODULE`) 再帰的操作アルゴリズム

親リポジトリとネストされた配下リポジトリ (`ENTRY_SUBMODULE`: `0x04`) 間の決定論的連携アルゴリズム。

1. **Recursive Fetch / Checkout**:
   親リポジトリの `Directory Node` (`SFDR`) を走査し、`ENTRY_SUBMODULE` エントリを検出した場合、配下の `.sfvcs/modules/<submodule_name>/` から対象 Commit CID を再帰的にチェックアウト。
2. **Submodule Status**:
   サブモジュールディレクトリの現在の `HEAD` CID と、親 Directory Node に記録されたターゲット CID を比較し、`SUBMODULE_DIRTY`（内部未コミット変更あり）または `SUBMODULE_MOVED`（参照コミット不一致）を検出。

---

# 13. Prolly Tree 病的入力フォールバック & 最大深度制限アルゴリズム

悪意ある入力（同じ文字の無制限な連続等）により CDC ハッシュが境界条件を満たさず、単一 Chunk が巨大化したり Sequence Tree が深くなりすぎる病的なケースに対するフォールバック保護仕様。

## 13.1 最大深度 `MAX_TREE_DEPTH = 32` および確定分割
- **ツリー最大深度制限**: Sequence Tree の再帰深度が `MAX_TREE_DEPTH = 32` に達した場合、境界判定 `is_internal_node_boundary()` の結果を無視し、ファンアウト上限 `MAX_FANOUT = 128` で強制的に固定グループ化・分割を行う。これによりスタックオーバーフローおよび極端な木構造の崩壊を数学的に防止する。

---

# 14. サブモジュール循環参照検出・防止アルゴリズム (Submodule Cyclic Dependency Detection)

サブモジュールが自らを親として参照したり、A -> B -> A の循環参照（Cyclic Submodule Tree）を持つ場合に、再帰走査時の無限ループ・スタックオーバーフローを防止する判定アルゴリズム。

```python
def check_submodule_cycles(visited_repo_paths: Set[str], current_submodule_url: str) -> None:
    canonical_url = normalize_repo_url(current_submodule_url)
    if canonical_url in visited_repo_paths:
        raise SubmoduleCycleError(f"Submodule cyclic dependency detected: {canonical_url}")
    visited_repo_paths.add(canonical_url)
```

---

# 15. 文字コード自動判定 & マルチバイト安全 Unified Diff 変換

Shift_JIS, EUC-JP, UTF-16, UTF-8 などの多種多様なエンコーディングを含むテキストファイルに対し、文字化けやマルチバイト文字の境界切断を発生させずに Unified Diff を安全生成するアルゴリズム。

1. **BOM & UTF-8 / Universal Chardet 判定**:
   ファイル先頭の BOM (Byte Order Mark) およびバイト頻度解析によりエンコーディングを決定。
2. **文字境界アライン**:
   Chunk 差分からの行分割時、UTF-8 コードポイントの途中バイト（例: 3バイト文字の2バイト目）で切断しないよう、前後の改行バイト `\n` または UTF-8 リーダーバイト位置へオフセットを文字境界補正する。

---

# 16. Prolly Tree 病的ハッシュ攻撃・偏り保護のための二重マスク (Dual-Mask Normalization) 制御

入力データが連続する同一バイト列（例: `0x00` の大量連続）やハッシュ攻撃的な入力を含む場合、単一の正規化マスク判定では Chunk サイズが極端に巨大化・細分化する危険がある。

## 16.1 Dual-Mask Normalization アルゴリズム
`FastCDC` 境界判定において、`MIN_SIZE` (2 KiB) から `AVG_SIZE` (8 KiB) までは厳格なマスク `MASK_S` (13 bits: 確率 $1/8192$) を使用し、`AVG_SIZE` (8 KiB) から `MAX_SIZE` (64 KiB) までは緩和されたマスク `MASK_L` (9 bits: 確率 $1/512$) を使用する。

さらに、Gear Hash テーブルのハッシュ空間の偏りを補正するため、ローリングハッシュ値に二次ハッシュ関数（Salted SHA-256 由来のビット置換: `hash ^ 0x9E3779B97F4A7C15ULL`）を合成し、病的入力であっても決定論的かつ安定した Chunk 分割結果を保証する。

```python
def dual_mask_chunk_boundary(buffer: bytes, offset: int, total_len: int) -> int:
    remaining = total_len - offset
    if remaining <= MIN_SIZE:
        return remaining

    max_len = min(remaining, MAX_SIZE)
    hash_val = 0
    SALT = 0x9E3779B97F4A7C15
    
    curr = offset + MIN_SIZE
    avg_boundary = min(offset + AVG_SIZE, offset + max_len)

    while curr < avg_boundary:
        byte_val = buffer[curr]
        hash_val = ((hash_val << 1) + GEAR_TABLE[byte_val]) & 0xFFFFFFFFFFFFFFFF
        if ((hash_val ^ SALT) & MASK_S) == 0:
            return curr - offset + 1
        curr += 1

    max_boundary = offset + max_len
    while curr < max_boundary:
        byte_val = buffer[curr]
        hash_val = ((hash_val << 1) + GEAR_TABLE[byte_val]) & 0xFFFFFFFFFFFFFFFF
        if ((hash_val ^ SALT) & MASK_L) == 0:
            return curr - offset + 1
        curr += 1

    return max_len
```

---

# 17. MinHash / SuperMinHash スケッチを用いた Packfile 差分基底 (Thin Delta Base) 高速選定アルゴリズム

パックファイル生成時、全オブジェクト間のペア比較を行わずに、最も差分圧縮率（Thin Delta）が高くなる類似ベースオブジェクト $O_{\text{base}}$ を $O(1)$ で高速選定するアルゴリズム。

## 17.1 MinHash スケッチ計算と Jaccard 類似度判定
1. 各 Leaf Chunk またはファイルオブジェクトに対し、$K = 64$ 個の独立したハッシュ関数 $h_1, h_2, \dots, h_K$ による MinHash スケッチ $V(O) = [\min_{x \in O} h_1(x), \dots, \min_{x \in O} h_K(x)]$ を生成する。
2. 2 つのオブジェクト $A, B$ の Jaccard 類似度 $\hat{J}(A, B)$ を以下で推定する：
   $$\hat{J}(A, B) = \frac{1}{K} \sum_{i=1}^K \mathbb{I}(V(A)[i] == V(B)[i])$$
3. 類似度 $\hat{J}(A, B) \ge 0.50$ のオブジェクト候補の中で、最もバイトサイズが近くかつ生成タイミングが近いものを優先して VCDIFF Thin Delta のベースオブジェクト $O_{\text{base}}$ として選択する。

---

# 18. サブ行レベル (Sub-line / Token-level) Meyers Alignment を統合した高精度 Blame / Provenance 追跡

1 行の中に複数の識別子や変数が変更された場合（例: `let x = 1;` -> `let x = 2;`）、行単位の Blame では行全体の著者が上書きされる。サブ行レベル（トークン単位）で変化を追跡し、より精密な変更履歴 provenance を特定する。

## 18.1 トークン境界 Meyers Diff アライメント
1. 行レベルの差分から変更行ペア $(L_{\text{old}}, L_{\text{new}})$ を特定。
2. 該当行をプログラミング言語・汎用トークナイザ（識別子、演算子、リテラル、空白）によりトークン列 $T_{\text{old}}, T_{\text{new}}$ に分割。
3. トークン列に対して Meyers Diff を再帰適用し、未変更のトークン列（例: `let`, `x`, `=`）の provenance（元コミット・著者）を保持したまま、変更されたトークン（例: `2`）のみに新コミット・著者を紐付ける。

---

# 19. 複数マージベース (Criss-Cross Merge) における仮想マージベース (Virtual Merge Base) 自動生成アルゴリズム

有向非巡回グラフ (DAG) 内に複数の共通最浅祖先（Lowest Common Ancestors: LCA）が存在する Criss-Cross マージシナリオ（例: Ours と Theirs が相互に交差マージを繰り返した場合）において、確定的な 3-Way Structural Merge を実現するアルゴリズム。

## 19.1 仮想マージベース生成手順
1. コミットグラフから Ours コミット $C_{\text{ours}}$ と Theirs コミット $C_{\text{theirs}}$ の全共通祖先集合 $\text{LCA}(C_{\text{ours}}, C_{\text{theirs}}) = \{B_1, B_2, \dots, B_m\}$ を検出。
2. $|\text{LCA}| = 1$ の場合は、その単一コミットをそのまま Merge Base とする。
3. $|\text{LCA}| > 1$ (Criss-Cross シナリオ) の場合：
   - 祖先コミット群 $B_1, B_2$ に対し、再帰的に 3-Way Structural Merge を適用。
   - メモリ上に仮のマージスナップショット（Virtual Merge Base: $V_{\text{base}}$）を一時構築する。
4. この合成された仮想ツリー $V_{\text{base}}$ を `Base` とみなし、$C_{\text{ours}}$ と $C_{\text{theirs}}$ の最終 3-Way Structural Merge を実行する。これにより Criss-Cross 発生時でも競合の誤検出をゼロにする。

```python
def find_or_create_virtual_merge_base(commit_ours: Commit, commit_theirs: Commit) -> DirectoryNode:
    lcas = get_lowest_common_ancestors(commit_ours, commit_theirs)
    if len(lcas) == 1:
        return load_commit_root_tree(lcas[0])

    # Criss-Cross シナリオ: 複数の LCA から再帰的に Virtual Merge Base を合成
    base_tree = load_commit_root_tree(lcas[0])
    for i in range(1, len(lcas)):
        next_tree = load_commit_root_tree(lcas[i])
        # 再帰的 3-way merge により中立な Virtual Merge Base を生成
        base_tree = merge_trees(base=base_tree, ours=base_tree, theirs=next_tree).result_tree

    return base_tree
```

---

# 20. 巨大バイナリ用 VCDIFF メモリフットプリント制限付きストリーミングエンコード/デコードアルゴリズム

数ギガバイト級のバイナリファイルに対して VCDIFF 差分エンコード/デコードを行う際、メモリ使用量を一定枠（例: **64 MiB 以下**）に制限しながらストリーミング処理を行うアルゴリズム。

## 20.1 チャンクドスライディングウィンドウ VCDIFF 仕様
1. 入力ストリームを固定サイズのターゲットウィンドウ $W_T$ (デフォルト: **16 MiB**) ごとに区切る。
2. 対応するベースオブジェクトの参照範囲 $W_B$ をメモリ上にマップし、VCDIFF 命令（`ADD`, `RUN`, `COPY`）を生成・消費する。
3. デコーダ側は `COPY` 命令実行時、ベースストリームへのランダムアクセスを $W_B$ バッファ内に限定することで、RAM 消費量を一定閾値以内に保ったままストリーミング処理を完結させる。
