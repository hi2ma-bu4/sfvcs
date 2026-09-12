# sfvcs アルゴリズム詳細仕様書

本書は `sfvcs` における核心アルゴリズムの具体的な処理フロー、パラメタ定義、計算式、および擬似コードを明確にする仕様書である。

対象アルゴリズム：
1. **Content-Defined Chunking (FastCDC & Gear Hash)**
2. **Prolly Tree / Persistent Sequence Tree の構築アルゴリズム**
3. **Multi-resolution Structural Diff エンジン**
4. **Move / Rename / Copy 検出と Fingerprint 類似度検索（Winnowing）**
5. **Move / Rename / Case-only Rename 最適化アルゴリズム**

---

# 1. Content-Defined Chunking (FastCDC & Gear Hash)

`sfvcs` では、ファイルコンテンツの分割に **FastCDC** (Xia et al., 2016) アルゴリズムを採用する。

## 1.1 パラメータ仕様

| パラメータ名 | 変数名 | デフォルト値 | 設定可能範囲 | 説明 |
|---|---|---|---|---|
| 最小チャンクサイズ | `MIN_SIZE` | **2,048 B (2 KiB)** | 512 B 〜 8 KiB | これ未満の境界検出をスキップ |
| ターゲットサイズ | `AVG_SIZE` | **8,192 B (8 KiB)** | 2 KiB 〜 32 KiB | 正規化境界判定の目標平均サイズ |
| 最大チャンクサイズ | `MAX_SIZE` | **65,536 B (64 KiB)**| 16 KiB 〜 256 KiB | 強制切り出し閾値 |
| ローリングウィンドウ | `WINDOW_SIZE` | **48 バイト** | 固定 | ハッシュ計算用ウィンドウ |
| 正規化マスク1 | `MASK_S` | `0x0000d90003510000ULL` | - | `MIN_SIZE` 〜 `AVG_SIZE` 用の厳格マスク (約13ビット立て) |
| 正規化マスク2 | `MASK_L` | `0x0000d90000000000ULL` | - | `AVG_SIZE` 〜 `MAX_SIZE` 用の緩和マスク (約9ビット立て) |

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
    # 子CIDのLCS (Longest Common Subsequence) または Unique Anchor 検出
    anchors = find_exact_cid_anchors(node_A.children_cids, node_B.children_cids)

    # 2. アンカー間のギャップ (非一致区間) のみを選択的に再帰比較
    diffs = []
    for gap in extract_gaps(anchors):
        if gap.is_exact_match:
            diffs.append(DiffResult.Unchanged(gap.cid))
        else:
            # ギャップ部分のみ解像度を上げて解剖
            sub_diff = diff_unmatched_subtrees(gap.sub_A, gap.sub_B)
            diffs.append(sub_diff)

    return combine_diffs(diffs)
```

---

# 4. Move / Rename / Copy 検出と Fingerprint 類似度検索

## 4.1 Winnowing ベースの近似 Fingerprint 計算

ファイル・サブルーチンの類似度判定のため、**Winnowing** アルゴリズムによりスケッチ（Fingerprint）を生成する。

- **k-gram サイズ**: $k = 16$ バイト
- **ウィンドウサイズ**: $w = 32$ バイト

```python
def compute_winnowing_fingerprint(data: bytes) -> List[uint32]:
    # 1. 全 k-gram のローリングハッシュ計算
    hashes = [gear_hash(data[i:i+16]) for i in range(len(data) - 15)]
    
    # 2. ウィンドウ w 内の最小ハッシュ値を抽出
    fingerprints = set()
    min_pos = -1
    
    for i in range(len(hashes) - 31):
        window = hashes[i:i+32]
        # ウィンドウ内の最小値のインデックスを選択
        min_val = min(window)
        fingerprints.add(min_val)

    return sorted(list(fingerprints))
```

---

# 5. Move / Rename / Case-only Rename 最適化アルゴリズム

ファイル移動・リネーム・大文字小文字の変更・ディレクトリ一括移動を高精度かつ高速に判定するための最適化アルゴリズム群。

## 5.1 $O(1)$ ディレクトリサブツリー一括リネーム (Directory Subtree Rename)

サブディレクトリ（例: `src/old_dir/` 以下の数千ファイル）が一括移動された場合、個別ファイル比較を一切行わず、$O(1)$ でディレクトリ単位のリネームとして検出する。

```python
def detect_directory_renames(old_tree: DirectoryNode, new_tree: DirectoryNode) -> List[RenameItem]:
    # 1. 旧ツリーから削除された全 Directory Node の (CID -> path) マップを作成
    deleted_dirs = collect_deleted_directory_cids(old_tree, new_tree)
    # 2. 新ツリーに追加された全 Directory Node の (CID -> path) マップを作成
    added_dirs = collect_added_directory_cids(old_tree, new_tree)
    
    directory_renames = []
    
    # 3. Directory CID の完全一致を比較 ($O(1)$)
    for cid, old_dir_path in deleted_dirs.items():
        if cid in added_dirs:
            new_dir_path = added_dirs[cid]
            directory_renames.append(DirectoryRename(
                from_path=old_dir_path,
                to_path=new_dir_path,
                confidence=1.00,
                subtree_cid=cid
            ))
            # このサブディレクトリ配下の個々のファイル走査を Fast-Path スキップ
            skip_subtree_inspection(old_dir_path, new_dir_path)
            
    return directory_renames
```

## 5.2 大文字小文字のみの変更検出 (Case-only Rename Detection)

Windows (NTFS) や macOS (APFS デフォルト) などのケースインセンシティブな環境において、`foo.ts` から `Foo.ts` へのリネームを競合なく検知する。

### 判定手順
1. 旧スナップショットのパス集合 `OldPaths` から小文字化マップ `Map<lowercase_path, original_path>` を作成。
2. 新スナップショットのパス `NewPath` について：
   - `NewPath` が `OldPaths` に直接存在せず（ファイル削除と新規追加に見える）、かつ `NewPath.toLowerCase()` が小文字化マップにヒットする場合。
   - かつ `Content_CID` が一致、または類似度が閾値以上である場合、`CASE_ONLY_RENAME`（大文字小文字変更）として判定。

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

## 5.3 複合信頼度スコア計算 (Composite Confidence Score)

一部内容が変更された移動 (`MOVE + MODIFY`) において、以下の 3 つの要素を重み付け加算して最終スコア $S \in [0.0, 1.0]$ を算出する。

$$S = w_1 \cdot \text{Similarity}_{\text{Content}} + w_2 \cdot \text{Similarity}_{\text{Path}} + w_3 \cdot \text{Context}_{\text{ParentDir}}$$

- **$w_1 = 0.60$**: Winnowing Fingerprint の Jaccard 類似度。
- **$w_2 = 0.25$**: ファイル名文字列の類似度 (Levenshtein 距離正規化値)。
- **$w_3 = 0.15$**: 親ディレクトリ構造・同層隣接ファイルの CID 一致率。

$S \ge \text{renameThreshold}$（デフォルト: **0.65**）の場合に `MOVE + MODIFY` 候補として採用する。
