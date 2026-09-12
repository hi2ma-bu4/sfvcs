# sfvcs 仮設計書

- プロジェクト名: `sfvcs`
- 管理ディレクトリ: `.sfvcs`
- 想定実装環境: Node.js + TypeScript
- 現段階: 仮設計 / 要件定義・アーキテクチャ検討段階
- 目的: Gitとは異なる、独自仕様の高効率なコンテンツ指向バージョン管理システムを構築する

---

# 1. 文書の目的

本書は `sfvcs` の基本要件、アーキテクチャ設計、データモデル、ストレージ構造、差分計算、実装方針、および関連詳細仕様書へのインデックスを記録するための基本設計書である。

詳細なバイナリレイアウト、具体アルゴリズムの擬似コード、パックファイル物理フォーマット、および CLI/API インターフェースについては、以下の専用詳細仕様書を参照すること。

- **[バイナリフォーマット・カノニカルシリアライズ詳細仕様書](../specs/spec-binary-format.md)** (`doc/specs/spec-binary-format.md`)
- **[アルゴリズム詳細仕様書 (FastCDC, Prolly Tree, Diff, Line Offset, Move/Rename最適化, Bisect, Blame, Submodule, Pathspec Trie)](../specs/spec-algorithms.md)** (`doc/specs/spec-algorithms.md`)
- **[ストレージ構造・パックファイル・Index・Commit Graph・LFS・VCDIFF・Tri-color GC詳細仕様書](../specs/spec-storage.md)** (`doc/specs/spec-storage.md`)
- **[3-Way Structural Merge ・ CRDT 競合解決 ・ 汎用構造化マージ詳細仕様書](../specs/spec-merge.md)** (`doc/specs/spec-merge.md`)
- **[リモート同期 ・ Wire Protocol ・ LFS 転送詳細仕様書](../specs/spec-network.md)** (`doc/specs/spec-network.md`)
- **[仮想VCS (インメモリ/ブラウザ/OPFS/IndexedDB/WebCrypto/暗号化/CRDT/WASMプラグイン) 詳細仕様書](../specs/spec-virtual-vcs.md)** (`doc/specs/spec-virtual-vcs.md`)
- **[設定ファイル・Ignore・属性詳細仕様書 (.sfvcsconfig, .sfvcsignore, .sfvcsattributes)](../specs/spec-config-and-attributes.md)** (`doc/specs/spec-config-and-attributes.md`)
- **[CLI コマンド・Submodule・Bisect・Blame・Core API・エラー処理詳細仕様書](../specs/spec-cli-and-api.md)** (`doc/specs/spec-cli-and-api.md`)
- **[ドキュメント命名・構造・記述ルール](../rules/doc-naming-rules.md)** (`doc/rules/doc-naming-rules.md`)

本書では、現時点で確定した設計と、研究・実験によって今後変更する可能性がある設計を明確に区別する。

特に以下について、後から設計意図を失わないことを目的とする。

- なぜ通常のGit型オブジェクトモデルをそのまま採用しないのか
- なぜファイルを単一blobとして保存しないのか
- なぜファイル内容を複数解像度の木構造として保持するのか
- なぜ行単位をcanonicalなデータ構造にしないのか
- なぜASTをcanonicalなデータ構造にしないのか
- なぜMOVEを保存形式として持たないのか
- なぜdiff indexをcanonical storageから分離するのか
- なぜcontent-defined chunkingを採用するのか
- なぜ木構造を固定階層ではなくrecursive / adaptiveな構造にするのか
- 各種パラメータを論文・既存実装から初期化し、ベンチマークで調整する理由

---

# 2. プロジェクト概要

`sfvcs` はGit互換を目的としない独自仕様のバージョン管理システムである。

主な設計思想は以下の通り。

1. データをcontent-addressed objectとして扱う (標準CID: **BLAKE3 [0x02]**, 互換: SHA-256 [0x01])
2. immutableなデータ構造を基本とする
3. 過去のsnapshot間で共通する構造を可能な限り共有する
4. ファイル内容を単一blobではなくrecursiveな sequence tree (Prolly Tree) として保持する
5. content-defined chunkingによって変更位置に依存しにくいchunk境界を作る
6. 上位Nodeもcontent identityを持つ
7. 同一内容のNodeは複数のsnapshotから共有する
8. diffはrootから必要な部分だけ解像度を上げながら比較する (Multi-resolution Diff)
9. MOVE / COPY / RENAMEなどはcanonical storageに記録せず、snapshot間の比較結果から導出する
10. diff用の近似検索indexはcanonical storageとは分離し、再構築可能なderived dataとする
11. 3-Way Structural Merge により Prolly Tree 構造および CRDT 研究に基づく確定的な合流・競合解決を行う
12. Prolly Tree ルート CID 比較による対数時間 $O(\log N)$ 最小差分交渉 Wire Protocol を備える

最終的には、

    「変更履歴を保存する」

というより、

    「各snapshotを、過去snapshotとの構造共有が最大になるように保存し、
     snapshot間の変更内容を効率的に復元する」

ことを目的とする。

---

# 3. Gitとの関係

`sfvcs` はGitの内部実装を再現するプロジェクトではない。

Gitの以下の概念は参考にする。

- commit
- branch
- immutable object
- content address
- parent commit
- reachable object
- garbage collection

しかし、データ保存方式については独自方式を採用する。

特に、Gitで一般的な

    file -> blob

という単純なファイル内容保存ではなく、

    file
      -> recursive content tree
        -> chunk
          -> bytes

という構造を基本とする。

また、Gitのdelta compressionを主要な共有機構とはしない。

`sfvcs` では、delta compressionより先に

    content-addressed structural sharing

を利用する。

---

# 4. 用語

## 4.1 Repository

`.sfvcs`をルートとして管理されるバージョン管理対象。

---

## 4.2 Object

content-addressedなimmutableデータ。

例:

- Commit
- Directory Node
- File Node
- Sequence Node
- Chunk
- metadata object

---

## 4.3 CID

Content ID。

Objectのcanonical serializationから計算されるcontent identity。

デフォルトは **BLAKE3 (32バイト)** とする。

互換・検証用に **SHA-256** もサポートする。

CIDは以下を目的とする。

- object identity
- deduplication
- equality判定
- subtree equality判定
- diff高速化
- integrity verification

---

## 4.4 Snapshot

ある時点のファイルシステム状態。

Commitはsnapshotを直接内包するのではなく、root directory CIDなどを参照する。

---

## 4.5 Sequence Tree

ファイル内容を順序付き要素として表現するrecursive tree。

例:

    Root
    ├── Node
    │   ├── Node
    │   │   ├── Chunk
    │   │   └── Chunk
    │   └── Chunk
    └── Node
        ├── Chunk
        └── Chunk

各Nodeはordered childrenを持つ。

---

## 4.6 Chunk

ファイルbyte列のleaf object。

Chunk境界は固定byte数ではなく、原則としてcontent-defined chunkingによって決定する。

---

## 4.7 Derived Index

canonical storageから再構築可能な補助データ。

例:

- CID -> occurrence index
- fingerprint -> candidate index
- line boundary index
- diff cache
- path rename candidate index

Derived Indexが失われてもrepositoryの正当性は失われない。

---

## 4.8 Occurrence

Objectのidentityとは別に、そのobjectがsequence上のどこに出現しているかを表す。

例えば、

    [A, A, B]

では、

    CID(A)

は1つのidentityだが、Aは2つのoccurrenceを持つ。

Identityとpositionを混同しない。

---

# 5. 基本設計原則

## 5.1 IdentityとPositionを分離する

最重要原則の一つ。

同一CIDが複数箇所に存在できる。

例えば、

    A B C

を

    C A B

に移動した場合、

Cそのもののidentityは変わらない。

変わるのはsequence上のpositionだけ。

したがって、object identityにpositionを含めない。

---

## 5.2 Canonical storageとdiff semanticsを分離する

canonical storageには、

- INSERT
- DELETE
- MOVE
- COPY
- RENAME

などの操作履歴を保存しない。

保存するのは変更後のimmutable snapshot。

例えば、

    Old:
    A B C D

    New:
    C D A B

の場合、New側には単純に

    C D A B

というsequence treeを保存する。

diff時にOldとNewを比較して、

    C,Dが位置を変更した

ことをMOVE candidateとして導出する。

---

## 5.3 Exact identityとsimilarityを分離する

CID一致はexact equality。

fingerprint一致・類似度は候補探索。

したがって、

    CID == CID

は完全一致として扱う。

一方、

    fingerprint(A) ≈ fingerprint(B)

は、

    AとBは似ている可能性がある

というだけであり、identityを同一視してはいけない。

---

## 5.4 Derived indexは壊れてもよい

Diff indexはrepositoryの正本ではない。

以下の状態を許容する。

- indexが存在しない
- indexが古い
- indexが破損している
- indexを削除した
- 別のアルゴリズムで再構築した

その場合でもcanonical objectsから再構築できる。

---

## 5.5 行はcanonical structureにしない

テキストファイルであっても、canonical structureを行単位にはしない。

理由:

- 1行追加で以降のpositionが変化する
- JSON/minified textに弱い
- binaryに対応できない
- line ending差の影響
- 長大な1行を扱いにくい
- 行という概念自体が言語依存・フォーマット依存

行境界はdiff表示・検索用のderived informationとして利用する。

---

## 5.6 ASTはcanonical structureにしない

ASTは将来的なderived indexとしては検討可能だが、core storageには使用しない。

理由:

- 言語依存
- parser依存
- parser version依存
- binary fileを扱えない
- 未対応言語が存在する
- source code以外では意味を持たない

`sfvcs`のcore storageはbyte sequenceとして言語非依存にする。

---

# 6. 全体アーキテクチャ

概念上は以下の構造とする。

    Repository
        │
        ├── References
        │      ├── branches
        │      └── tags
        │
        ├── Commits
        │
        ├── Directory Tree
        │
        ├── File Metadata
        │
        └── Content Tree
                │
                ├── Internal Sequence Node
                │
                └── Chunk
                        │
                        └── Raw Bytes

これとは別に、

    Derived Data
        │
        ├── CID Index
        ├── Fingerprint Index
        ├── Position Index
        ├── Line Index
        └── Diff Cache

を持つ。

Derived Dataはcanonical objectsとは分離する。

---

# 7. `.sfvcs` ディレクトリ

Gitの`.git`に相当する管理ディレクトリとして、

    .sfvcs/

を使用する。

初期案:

    .sfvcs/
    ├── config
    ├── HEAD
    ├── refs/
    │   ├── heads/
    │   └── tags/
    ├── objects/
    ├── index/
    ├── logs/
    ├── packs/
    └── state/

ただし、実際のディレクトリ構成は実装時に変更可能。

---

# 8. References

branchはmutable referenceとして扱う。

例えば、

    .sfvcs/refs/heads/main

にはcommit CIDを保存する。

概念的には、

    main -> Commit C

となる。

Commit自体はimmutable。

branchだけが変更される。

---

# 9. Commit

Commitは少なくとも以下を持つ。

    Commit {
        version
        parents[]
        root
        author
        timestamp
        message
        signature
    }

`root`はsnapshotのroot directory CID。

Commitはcontent-addressed objectとする。

Commitのtimestamp等を含める場合、同じsnapshotでもcommit objectは異なる。

そのため、

    Snapshot identity

と

    Commit identity

は区別する。

---

# 10. Snapshot

Snapshotはdirectory treeのroot CIDによって表現される。

概念:

    Snapshot
        |
        +-- root Directory CID

同じfilesystem状態なら同じroot CIDになる。

ただしcommit metadataが異なる場合、commit CIDは異なる。

---

# 11. Directory Tree

Directoryはrecursive treeとする。

概念:

    Directory
    ├── file.txt -> File Node
    ├── image.png -> File Node
    ├── sub_repo -> Submodule (Commit CID)
    └── src -> Directory Node
              ├── main.ts
              └── util.ts

directory entryには、

- name
- type (FILE: 0x01, DIRECTORY: 0x02, SYMLINK: 0x03, SUBMODULE: 0x04)
- child CID

持たせる。

また、`File Node` 内の拡張属性 (`xattr`) については、シリアライズ順序による CID 決定性のブレを防ぐため、キー文字列の UTF-8 バイト昇順でのソートを必須とする。
サブモジュールエントリ (`ENTRY_SUBMODULE`) は、ネストされた別 sfvcs リポジトリの該当コミット CID (`SFCM`) を参照し、マルチリポジトリの統合管理を可能にする。

---

# 12. Directory Nodeのidentity

Directory NodeのCIDは、canonicalizedされたentry集合から決定する。

例えば、

    name (UTF-8, Unicode NFC)
    type
    child CID
    mode等の必要なmetadata

をcanonical serializationする。

entryの順序をidentityに含めるか、name sortをcanonical ruleにするかは要検討。

第一候補は、

    directory entriesはname順でcanonicalize

する方式。

これにより同じdirectory状態は同じCIDになる。

---

# 13. Directoryとrename

例えば、

    old.txt -> new.txt

というrenameが発生した場合、

file content CIDが同じなら、

    old.txt
        -> File CID A

から

    new.txt
        -> File CID A

となる。

File Node自体をコピーする必要はない。

diff側では、

    old.txt DELETE
    new.txt ADD

ではなく、

    RENAME old.txt -> new.txt

という意味を導出できる。

ただし、同一contentのファイルが複数存在する場合は一意にrenameと判定できない。

そのためrenameは確定操作ではなく、

    rename candidate

として扱う場合がある。

---

# 14. File Node

File Nodeは少なくとも、

    File {
        contentRoot
        metadata
    }

を持つ。

metadata候補:

- executable
- mode
- symlink information
- file type
- extended attributes (xattr)
- 必要に応じた追加metadata

mtimeは原則としてcontent identityに含めない。

理由:

同じファイル内容でもアクセス・コピーなどでmtimeが変化するため。

ただし「mtimeそのものをversion管理したい」という要件が追加された場合は別途検討する。

---

# 15. ファイル内容のcanonical representation

ファイル内容は、

    bytes

を基本とする。

テキスト・画像・音声・動画・バイナリ等を同じ仕組みで扱う。

canonical content:

    File
      ↓
    Sequence Tree
      ↓
    Chunk
      ↓
    bytes

---

# 16. Sequence Tree

Sequence Treeは順序付きpersistent tree。

例えば、

    A B C D E F G H

というchunk sequenceを、

    Root
    ├── Node(A B C)
    └── Node(D E F G H)

のように表現する。

重要なのは、Nodeがchild CIDを順序付きで持つこと。

例えば概念的には、

    SequenceNode {
        children: [
            CID-A,
            CID-B,
            CID-C
        ],
        lengths: [
            ...
        ]
    }

となる。

---

# 17. positionをkeyにしない

sequence treeでは、

    0
    1
    2
    3

のような絶対indexをobject identityに使用しない。

例えば先頭にchunkを1つ追加すると全positionがずれてしまうため。

代わりに、

    ordered children

そのものをsequenceとして扱う。

offset lookupが必要な場合は、各childのlogical lengthまたはsubtree lengthを利用して探索する。

---

# 18. Logical Length

各Sequence Nodeは、

    logicalLength

を持つ。

これはそのsubtreeが表すbyte数。

例えば、

    Chunk A = 4 KiB
    Chunk B = 8 KiB

なら、

    parent.logicalLength = 12 KiB

となる。

これによりbyte offsetからtreeを下降できる。

---

# 19. Sequence Nodeのcanonical identity

Sequence NodeのCIDは、

    node type
    format version
    ordered child CIDs
    child logical lengths
    必要なcanonical metadata

から決定する。

positionは含めない。

同じchild列を持つNodeは同じCIDになる。

---

# 20. Leaf Chunk

Leafはraw bytesを保持する。

概念:

    Chunk {
        type
        formatVersion
        byteLength
        data
    }

CIDはcanonical dataから計算する。

---

# 21. Chunking

固定サイズchunkではなく、content-defined chunkingを基本とする。

目的:

    編集によるbyte offset変化が、
    以降のchunk全体を作り直す原因になることを防ぐ。

例えば、

    Old:
    [A][B][C][D][E]

    New:
    [A][B][X][D][E]

なら、

    A
    B
    D
    E

を可能な限り再利用する。

---

# 22. CDCの候補

第一候補:

    FastCDC系

検討対象:

- Rabin fingerprint
- Gear hash
- FastCDC
- FastCDC normalization
- Dolt/Prolly Tree系boundary algorithm

現時点でFastCDCを採用確定とはしない。

実装前にベンチマークする。

---

# 23. Leaf chunkの初期パラメータ

研究・既存実装から得られた初期候補:

    target: 8 KiB
    min:    2 KiB
    max:    64 KiB

これは確定値ではない。

比較対象として少なくとも、

    4 KiB
    8 KiB
    16 KiB
    32 KiB

程度をベンチマークする。

また、

    min / target / max

の比率も調整対象とする。

---

# 24. Nomsの参考値

Noms/Prolly Treeでは代表的な設定として、

    average chunk size = 4 KiB
    rolling hash window = 64 bytes
    boundary probability = 1 / 4096

が使われている。

この方式はrecursiveに適用される。

これを`sfvcs`の設計の重要な先行例として扱う。

ただし、Nomsの値をそのまま最適値とはみなさない。

---

# 25. FastCDCの参考値

FastCDC系では、

    min = 8 KiB
    avg = 16 KiB
    max = 32 KiB

のような実装例が存在する。

また別実装では、

    min = 16 KiB
    target = 64 KiB
    max = 256 KiB

なども存在する。

つまり、chunk sizeは用途依存である。

`sfvcs`では研究値を初期値として採用し、実際のrepository workloadで測定する。

---

# 26. Chunk size distribution

単純なCDCではchunk sizeが幾何分布になり、小chunkが多くなる問題がある。

Dolt/Prolly Tree系ではchunk size distributionを制御し、target size周辺に集中させる手法が検討されている。

したがって、

    average = target

だけではなく、

    size distribution

そのものを評価する。

評価項目:

- mean
- median
- p50
- p90
- p99
- min
- max
- chunk count
- bytes per chunk
- boundary stability

---

# 27. Boundary Stability

非常に重要な評価指標。

小さな変更を行った場合に、変更箇所から遠く離れたchunk境界まで変更される割合を測定する。

例:

    Old:
    [A][B][C][D][E][F]

    New:
    [A][B][X][D][E][F]

理想:

    [A] [B] [D] [E] [F]

をそのまま再利用できる。

悪い例:

    [A][B][C][D][E][F]

が、

    [A][B][XDE][F...]

のように大きく再分割される。

評価では、

    changed input bytes
    vs
    changed stored bytes

を比較する。

---

# 28. Internal Node

Leafより上のNodeはchild CIDを順序付きで保持する。

内部Nodeのboundaryについては、固定child countよりもcontent-dependentな方式を第一候補とする。

候補:

1. child CID hashによるboundary
2. serialized child entryのCDC
3. child count + target serialized size
4. Prolly Tree型boundary

---

# 29. Internal Nodeのサイズ基準

固定fanoutだけでなく、

    serialized node size

を重要なパラメータとする。

例えば、

    target node size = 数KiB〜数十KiB

程度を候補としてベンチマークする。

正確な値は未確定。

理由:

- CIDサイズ
- child count
- length metadata
- object store overhead

などによって最適値が変わるため。

---

# 30. Adaptive Depth

tree depthは固定しない。

概念:

    小さいfile
        -> shallow tree

    大きいfile
        -> deep tree

つまり、

    L0
    L1
    L2
    L3

という固定レベル名をcanonical modelとして持たない。

Nodeは単に、

    leaf
    internal node

としてrecursiveに定義する。

実際のdepthはデータ量によって決まる。

---

# 31. Recursive Treeの重要な性質

例えば、

    root
      -> A
      -> B
      -> C

のBだけが変更された場合、

    root
      -> A
      -> B'
      -> C

となる。

AとCは既存CIDを再利用できる。

rootとBのpathだけが新しくなる。

これをpersistent structural sharingとして利用する。

---

# 32. MOVEを保存しない理由

Old:

    A B C D

New:

    C D A B

という状態だけから、

    C,Dをmoveした

ことは確定できない。

実際には、

- copy
- delete + insert
- move
- generatorによる再生成

など複数の操作履歴が同じ最終状態を作れる。

したがってcanonical storageにMOVEを保存すると、履歴の意味を誤って固定する可能性がある。

`sfvcs`では、

    state

を保存し、

    operation semantics

をdiffで導出する。

---

# 33. Multi-resolution Diff

diffはrootから開始する。

基本アルゴリズム:

    compare(oldRoot, newRoot)

1. CIDが同じ
   -> Equal
2. leaf
   -> byte-level diff
3. internal node
   -> child sequenceを比較
4. exact CID matchをanchorとして利用
5. 同じsubtreeが別位置にある場合MOVE candidate
6. unmatched nodeについてfingerprintを比較
7. similarityが高ければrecursive descent
8. similarityが低ければinsert/delete/replace
9. 最終的にhuman-readable diffへ変換

---

# 34. CID Equal Fast Path

最重要の最適化。

    oldCID === newCID

なら、

    subtree全体を比較不要

とする。

これは巨大なunchanged file / directoryに対して非常に強い。

例えば1GB fileでも、

    oldRootCID === newRootCID

ならbyte比較を一切行わない。

---

# 35. Multi-resolutionの考え方

比較対象:

    Root
      ↓
    large subtree
      ↓
    medium subtree
      ↓
    small subtree
      ↓
    chunk
      ↓
    bytes

と必要な場所だけ降りる。

変更のない領域は上位CID一致によってskipする。

変更領域だけ解像度を上げる。

---

# 36. Diff Index

Diff Indexはcanonical storageとは別物。

候補:

    CID -> occurrence list

    CID -> parent information

    fingerprint -> candidate CID list

    CID -> approximate locations

    commit -> subtree occurrences

など。

目的:

- move candidate探索
- cross-file reuse探索
- rename candidate探索
- partial move探索
- approximate matching

---

# 37. Exact CID Index

最も基本的なindex。

概念:

    CID
      -> [
           occurrence,
           occurrence,
           ...
         ]

occurrenceは、

    commit
    file
    tree path
    offset
    length
    parent context

などを持つ。

ただし全repositoryに対して常時完全indexを構築するかは未決定。

---

# 38. Approximate Fingerprint Index

CIDが違っても内容が似ている場合がある。

例:

    Old:
    A B C D

    New:
    A B X D

BとDなどは同一CIDだが、CとXは異なる。

より細かいsimilarity探索では、

- rolling hash
- SimHash
- MinHash
- winnowing
- local fingerprints

などを候補として検討する。

ただし、fingerprintはidentityではない。

---

# 39. Fingerprintの用途

fingerprintは、

    "このNodeと似ているNodeはどれか？"

を高速に探すために使用する。

使ってはいけない用途:

    "このNodeはこのNodeと同一である"

の判定。

同一性判定はCIDを使う。

---

# 40. Fingerprint IndexはDerived

fingerprint indexはcanonical objectに含めない。

理由:

- algorithm変更可能
- hash変更可能
- threshold変更可能
- 再構築可能
- repository portabilityを保てる

例えば将来、

    fingerprint algorithm v1

から

    fingerprint algorithm v2

へ変更しても、object graph自体は変更不要。

---

# 41. Child Sequence Alignment

Internal Node同士を比較する場合、

    old.children
    new.children

をsequenceとして比較する。

候補アルゴリズム:

- LCS
- Myers
- Patience Diff
- Histogram Diff
- unique CID anchor
- rolling hash based matching

ただし、通常のbyte/string diffをそのまま使用する必要はない。

CID自体が強力なanchorになる。

---

# 42. Unique CID Anchor

例えば、

    Old:
    A B C D E F

    New:
    A B X D E F

なら、

    A
    B
    D
    E
    F

がexact anchorになる。

その間にある、

    C -> X

だけをrecursiveに比較する。

---

# 43. MOVE Candidate

Old:

    A B C D E

New:

    D E A B C

の場合、

    A B C
    D E

が両方に存在する。

順序だけが変化している。

この場合、

    MOVE [D,E] old position -> new position

というcandidateを生成できる。

---

# 44. MOVE + MODIFY

より難しいケース:

    Old:
    A B C D E

    New:
    A D X B C

Dが移動し、Eが削除され、Xが挿入されたケース。

さらに、

    D -> X

のように一部変更される場合もある。

このため、

    exact subtree match

だけでMOVE判定を終わらせない。

必要に応じて、

    parent context
    size
    fingerprint
    neighboring anchors

を利用する。

---

# 45. MOVEの曖昧性

同一CIDが複数存在する場合、

    A B A C

から

    A A B C

への変化について、どのAが移動したかは一意ではない。

そのため内部的には、

    MOVE candidate

とし、

最終diff生成時に、

- confidence
- context
- size
- neighboring match
- occurrence uniqueness

などを考慮する。

---

# 46. MOVE detectionの方針

最初の実装では、完全な最小edit scriptを目指さない。

目的は、

    高速
    + 十分に人間に分かりやすい
    + 大規模変更でも破綻しにくい

こと。

MOVE検出はheuristicとする。

---

# 47. Structural Diff研究との関係

Structural diffの研究では、treeのsharingを利用しながら差分を求めることで効率化する方向が存在する。

`sfvcs`ではこれを、

    content-addressed subtree

と組み合わせる。

CIDが一致するsubtreeは、構造的に同一であることを暗号学的hashによって高速に判定できる。

---

# 48. Line Diff

最終表示時にはline diffが必要になる場合がある。

ただしline情報は、

    canonical storage

ではなく、

    diff presentation / derived index

として生成する。

例えばleaf chunkをbyteとして保持したまま、

    newline offsets

をcacheする。

---

# 49. Binary Diff

binary fileでもcanonical structureは同じ。

例えば画像:

    image
      -> sequence tree
         -> chunks

となる。

diff表示は、

    binary changed

でもよい。

必要ならbyte-level change statisticsを提供する。

---

# 50. Text / Binaryの判定

core storageでは特別扱いしない。

UI / diff renderer側で、

- MIME
- filename
- encoding
- binary detection

などを使って表示方法を決める。

---

# 51. File Rename

File root CIDが同一ならrename候補として強く扱える。

例:

    old/a.txt -> FileCID X

    new/b.txt -> FileCID X

の場合、

    RENAME old/a.txt -> new/b.txt

を候補とする。

---

# 52. Directory Rename

DirectoryそのものもCIDを持つため、

    old/src
      -> Directory CID X

が、

    new/src
      -> Directory CID X

となれば、directory subtree全体のrename候補を作れる。

消去法でfile単位でrenameを検出するより効率的。

---

# 53. Cross-file Move

ファイルAからファイルBへ大きなblockが移動した場合でも、chunk CIDを利用できる。

例:

    A:
    [X][Y][Z]

    B:
    [P][Q]

から、

    A:
    [X][Z]

    B:
    [P][Y][Q]

となった場合、YのCIDが両者で一致する。

これを利用して、

    cross-file move candidate

を生成できる。

ただしcopyとの区別は最終状態だけでは不可能。

---

# 54. File Split / Merge

同じchunkが複数Fileに現れる場合、

    file split
    file merge

も候補として検出できる。

ただし最初のバージョンでは、これらを特別なsemantic operationとして表示せず、

    DELETE
    ADD
    MOVE candidate

程度に留めることも可能。

---

# 55. Storage Layer

canonical objectを通常の1object=1fileで保存すると、大規模repositoryではfilesystem inode数が問題になる可能性がある。

そのため将来的にはpack storageを第一候補とする。

概念:

    objects/
    ├── loose/
    └── packs/
         ├── pack-XXXX.pack
         └── pack-XXXX.idx

---

# 56. Loose Object

開発初期はloose objectを許容する。

例えば、

    .sfvcs/objects/<prefix>/<cid>

のように保存する。

メリット:

- 実装が簡単
- debuggingしやすい
- atomic writeしやすい

デメリット:

- object数増加でfilesystem overhead
- 大規模repositoryでlookupが遅くなる

---

# 57. Pack Object

本番向けには複数objectをpackする。

packには、

    compressed object bytes

をappendする。

indexには、

    CID -> offset
    CID -> size

などを保持する。

---

# 58. Compression

CIDは圧縮前のcanonical bytesから計算する。

つまり、

    canonical bytes
        ↓
    CID

とし、

    canonical bytes
        ↓
    compression
        ↓
    storage

とする。

圧縮形式を変更してもCIDは変化しない。

---

# 59. Compression候補

Node.js標準APIを優先する。

候補:

- Brotli
- gzip/deflate

zstd等は性能面では魅力的だが、外部依存を増やすため初期実装では慎重に扱う。

ユーザー環境にNode.jsがあることを前提とする。

---

# 60. Atomic Commit

commit処理は、

1. 新しいobjectを作る
2. objectをstorageへ保存
3. root treeを保存
4. commit objectを保存
5. refをatomicに更新

の順とする。

途中で失敗した場合、まだrefから到達できないobjectが残る。

それらはGC対象とする。

---

# 61. Reference更新

branch更新はatomicで行う。

可能なら、

    expected old CID
    new CID

によるCAS方式を採用する。

これにより並列processによるref上書きを検出できる。

---

# 62. Garbage Collection

immutable object storeでは、不要objectが増加する。

GCは、

    refs
      ↓
    reachable commits
      ↓
    reachable roots
      ↓
    reachable tree objects
      ↓
    reachable chunks

をmarkし、到達不能objectをsweepする。

---

# 63. Derived IndexとGC

Derived Indexはcanonical objectではない。

したがって、

    index rebuild

できる設計にする。

GC時にはindex側のstale entriesも削除する。

またはGC後にindexを再構築する。

---

# 64. Repository Integrity

repository integrity checkでは、

- CIDが内容と一致するか
- child CIDが存在するか
- logicalLengthが正しいか
- Directory entryが正しいか
- Commit parentが存在するか
- refが有効なcommitを指しているか

などを検査する。

---

# 65. Canonical Serialization

CIDを安定させるため、object serializationはcanonicalでなければならない。

禁止:

- property order依存
- platform依存
- locale依存
- JSON.stringifyの偶然の表現依存
- undefinedの曖昧な扱い
- timestamp formattingの揺れ

canonical encodingを明示的に定義する。

候補:

- 独自binary encoding
- canonical CBOR
- length-prefixed binary format

初期実装では依存を減らすため独自binary encodingも検討する。

---

# 66. Object Format Version

object serializationにはformat versionを含める。

例えば、

    objectType
    formatVersion
    payload

とする。

format version変更時にはCIDも変化する可能性がある。

これは意図した仕様とする。

---

# 67. Hash Algorithm

標準デフォルト:

    BLAKE3 (32バイト, algo_id = 0x02)

互換・検証用:

    SHA-256 (32バイト, algo_id = 0x01)

理由:

- 高速な並行ハッシュ計算可能
- cryptographic identityとして扱える

---

# 68. Object Type

概念的には、

    commit
    directory
    file
    sequence
    chunk

などを区別する。

CIDのcanonical inputにはobject typeを含める。

例えば、

    hash("chunk" + payload)

と

    hash("sequence" + payload)

が偶然同じpayloadでも同一identityにならないようにする。

実際にはlength-prefix等によってdomain separationを行う。

---

# 69. Small File Optimization

非常に小さいfileに対して、

    Chunk
    Sequence Node
    Root Node

を大量に作るのは非効率。

そのため、

    small file

は直接inline可能な設計を検討する。

例えばFile Nodeに、

    inline bytes

または

    contentRoot

のどちらかを持たせる。

thresholdはベンチマークで決定する。

---

# 70. MetadataとContentの分離

File Nodeのmetadataとcontent bytesは分離する。

これにより、

    permission変更

だけで巨大file contentを再生成しなくて済む。

逆にcontentが変わった場合でもmetadataは再利用できる可能性がある。

---

# 71. Directory metadata

Directoryについても必要に応じてmetadataを持つ。

ただしmtimeなど頻繁に変化する情報をidentityに入れるとstructural sharingを破壊するため慎重に扱う。

---

# 72. Working Tree

最初のCLIでは、

    working tree
        ↓
    scan
        ↓
    snapshot construction
        ↓
    commit

という流れを基本とする。

Gitのindexに完全相当するものを最初から作る必要はない。

---

# 73. Working Tree Scan

filesystemを走査し、

    path
    type
    metadata
    content

を取得する。

既存snapshotとの比較で、

- unchanged file
- modified file
- added file
- deleted file
- rename candidate

を効率的に判定する。

---

# 74. Unchanged File Detection

前回snapshotのFile Nodeを参照できる場合、

    metadata + size + optional quick fingerprint

などで変更なし候補を作る。

最終的にはcontent CIDを検証する。

必要以上に巨大fileを毎回hashしないよう、

    filesystem metadata cache

などを検討する。

ただしmtime/sizeだけでidentityを確定してはいけない。

---

# 75. Content Construction

変更されたfileについて、

1. file bytesを読む
2. leaf CDC
3. Chunk object生成
4. internal node構築
5. root CID取得
6. File Node生成

を行う。

unchanged chunkはCIDから再利用する。

---

# 76. Incremental Construction

可能であればfile全体を一度にmemoryへ読み込まない。

streaming constructionを基本とする。

目標:

    file size >> available RAM

でも処理できること。

---

# 77. Streaming CDC

入力:

    Readable stream

から順次byteを処理し、

    current chunk

を構築する。

boundaryが成立したらChunk objectをflushする。

---

# 78. Streaming Tree Construction

leaf chunkを逐次生成しながらinternal nodeを構築する。

イメージ:

    chunk
      ↓
    group
      ↓
    node
      ↓
    group
      ↓
    parent node
      ↓
    root

これにより巨大fileでも全treeをmemoryに展開する必要をなくす。

---

# 79. Tree ConstructionとDiffの関係

tree constructionはdiff性能を意識して設計する。

重要なのは、

    small edit
        -> small structural change

となること。

そのため、

- CDC
- boundary stability
- internal grouping
- node target size

を独立ではなく、総合的に評価する。

---

# 80. Multi-resolutionの最終形

理想形:

    Root
      │
      ├── large unchanged subtree
      │       └── CID equal -> SKIP
      │
      └── changed subtree
              │
              ├── unchanged subtree
              │       └── CID equal -> SKIP
              │
              └── changed subtree
                      │
                      └── ...
                              ↓
                           Chunk
                              ↓
                           Bytes

diff workは変更量に近づける。

---

# 81. Diff Complexityの目標

厳密な計算量保証よりも、

    O(number of changed / inspected nodes)

に近い挙動を目標とする。

完全一致subtreeはO(1)でskipできる。

理想的には、

    total repository size

ではなく、

    changed structural regions

に依存する。

---

# 82. Large Repository

大規模repositoryでは、

- object lookup
- CID index
- pack index
- tree traversal
- diff candidate search
- memory consumption

がボトルネックになる。

そのためベンチマークでは小規模だけでなく、

    10 MB
    100 MB
    1 GB
    10 GB+

などを対象とする。

---

# 83. Performance Metrics

最低限測定する。

## Storage

- total stored bytes
- object count
- unique chunk count
- deduplication ratio
- pack overhead

## Construction

- snapshot creation time
- bytes/sec
- CPU time
- peak RSS
- allocation count

## Diff

- wall time
- CPU time
- peak RSS
- number of visited nodes
- number of CID comparisons
- number of fingerprint comparisons
- number of byte comparisons

## Sharing

- reused bytes
- newly written bytes
- reused node count
- newly created node count

---

# 84. Boundary Stability Benchmark

例えば以下の変更を大量に生成する。

- 先頭1byte insertion
- 中央1byte insertion
- 末尾1byte insertion
- 1byte deletion
- 1 KiB insertion
- 1 KiB deletion
- random replacement
- repeated block insertion
- large block move

各ケースで、

    changed input bytes
    changed chunk count
    changed stored bytes
    changed tree nodes

を測定する。

---

# 85. Move Benchmark

最低限、

    ABCDE -> CDEAB
    ABCDE -> DEABC
    ABCDE -> ADEC B
    A B C D E -> A D X B C

などをテストする。

さらに、

- duplicate blocks
- nested moves
- move + modify
- move + delete
- move + insert
- cross-file move
- rename + modify

を試す。

---

# 86. Real-world Benchmark

synthetic testだけではなく、実際のsource repositoryを使う。

候補:

- TypeScript repository
- JavaScript repository
- Python repository
- JSON-heavy repository
- generated files
- binary assets
- documentation
- large text files

変更履歴を実際に再生し、

    commit N
      -> commit N+1

のdiff性能を測定する。

---

# 87. Parameter Sweep

各パラメータを固定せず比較する。

例:

    leaf target:
        4 KiB
        8 KiB
        16 KiB
        32 KiB

    node target:
        4 KiB
        8 KiB
        16 KiB
        32 KiB

など。

全組み合わせが多すぎる場合は段階的に探索する。

---

# 88. Parameter Preset

最終的には、

    Research Default

    Balanced

    Large File

    Small File

などのpresetを用意する可能性がある。

ただし、repository formatそのものとparameter presetを混同しない。

---

# 89. Format Compatibility

tree construction parameterを変更すると、

    同じbytes

でも

    別tree

になる可能性がある。

そのためrepository formatには、

    tree algorithm version

などを含める。

---

# 90. Algorithm Version

例えば、

    chunking algorithm v1
    node grouping algorithm v1
    fingerprint algorithm v1

などを明示する。

これにより、

    old repository

を新しいアルゴリズムで再構築する場合も互換性を管理できる。

---

# 91. Repack / Rebuild

canonical formatを変更せずに、

- pack compression
- diff index
- fingerprint index

だけを再構築できるようにする。

tree algorithmを変更する場合は、

    repository migration

として扱う。

---

# 92. CLI

初期CLI候補:

    sfvcs init
    sfvcs status
    sfvcs add
    sfvcs commit
    sfvcs log
    sfvcs diff
    sfvcs show
    sfvcs branch
    sfvcs checkout
    sfvcs restore
    sfvcs gc
    sfvcs fsck
    sfvcs repack

ただし、最初から全部実装しない。

---

# 93. 初期実装優先順位

## Phase 1

object storage

- CID
- canonical serialization
- loose objects
- read/write
- fsck

## Phase 2

content tree

- Chunk
- Sequence Node
- recursive tree
- CDC
- streaming construction

## Phase 3

directory / snapshot

- Directory
- File
- Commit
- refs

## Phase 4

basic diff

- CID equality
- recursive comparison
- insert/delete/modify

## Phase 5

multi-resolution diff

- child alignment
- exact anchors
- subtree skip

## Phase 6

move detection

- occurrence index
- CID matching
- move candidates

## Phase 7

approximate matching

- fingerprint
- candidate index
- similarity matching

## Phase 8

performance

- pack
- cache
- index optimization
- GC

---

# 94. 最初から実装しないもの

初期版では以下を後回しにする。

- AST
- CRDT
- distributed merge
- remote protocol
- server
- Git compatibility
- perfect minimum edit script
- sophisticated semantic source diff
- advanced compression
- complex query language

まず、

    efficient immutable content tree
    +
    efficient structural diff

を完成させる。

---

# 95. Merge

将来的にmergeを実装する。

基本:

    base
      / \
    ours theirs

の3-way merge。

CID equalityを最大限利用する。

例えば、

    base subtree == ours subtree

ならours側では変更されていない。

同様に、

    base subtree == theirs subtree

ならtheirs側では変更されていない。

これにより巨大なunchanged subtreeを高速に処理できる。

---

# 96. MergeとMOVE

MOVEは3-way diffから導出する。

canonical objectにMOVE operationを保存しない。

必要に応じて、

    move-aware 3-way merge

を実装する。

CRDTのtree move研究は将来の高度なmerge設計の参考資料とする。

---

# 97. Conflict

ASTを使わないため、source codeのsemantic conflictは検出しない。

基本的には、

- byte range
- chunk
- line
- structural sequence

の競合として扱う。

将来的にlanguage-specific merge pluginを追加する可能性はあるがcoreには含めない。

---

# 98. Object Sharing

同一CIDはrepository全体で共有する。

つまり、

    file A -> Chunk X

    file B -> Chunk X

ならChunk Xを2回保存しない。

さらに、

    commit 1 -> subtree X
    commit 2 -> subtree X

でも同一objectを共有する。

---

# 99. Global Deduplication

deduplicationの単位:

- file
- chunk
- internal node
- directory
- metadata

すべてcontent-addressedにすることで自然にdeduplicateされる。

---

# 100. Copy vs Move

canonical storageでは区別しない。

例えば、

    old:
    A B

    new:
    A B
    C B

の場合、Bがcopyされたように見える。

しかし、

    move

か

    copy

かは最終状態だけでは確定できない。

そのためdiff rendererでは、

    copied
    moved
    reused

などをconfidence付きで表現する可能性がある。

---

# 101. Diff Confidence

将来的にdiff resultにconfidenceを持たせる可能性がある。

例:

    exact CID match:
        1.00

    unique subtree match:
        1.00

    fingerprint + context match:
        0.90

    approximate similarity:
        0.70

など。

数値は未確定。

これは内部アルゴリズムの確信度をUI表示に使うためのもの。

---

# 102. Duplicate Handling

CIDが重複する場合、

    CID -> [occurrence...]

となる。

単純なhash map lookupだけではMOVEを一意に決定できない。

候補の選択には、

- occurrence count
- parent context
- neighboring CID
- offset
- subtree size
- path
- fingerprint

などを利用する。

---

# 103. Context Matching

例えば、

    Old:
    A B C D E

    New:
    A D E B C

の場合、D/Eの前後contextを見る。

単純CID一致だけではなく、

    surrounding CID sequence

を利用することでambiguityを減らす。

---

# 104. Hierarchical Fingerprint

fingerprintも単一解像度ではなく、Node単位で持つ可能性がある。

例えば、

    large Node fingerprint
    medium Node fingerprint
    small Node fingerprint

とする。

ただしNodeそのものにapproximate fingerprintをidentityとして含めない。

必要ならderived metadataとして保存する。

---

# 105. Fingerprint IndexのLazy Construction

repository全体に対して最初からfingerprint indexを作る必要はない。

diff要求時に、

    requested subtree

についてlazyに生成する方式を第一候補とする。

頻繁に使用するindexのみcacheする。

---

# 106. Diff Cache

同じcommit pairについて何度もdiffする場合、

    diff(commitA, commitB)

結果をcacheできる。

ただしcacheはcanonical objectではない。

tree / algorithm versionが変わった場合は無効化する。

---

# 107. Memory Strategy

巨大repositoryを扱うため、全CID indexを常にmemoryに載せることは避ける。

候補:

- LRU cache
- memory mapped index
- sorted index
- pack index
- lazy traversal

Node.jsでは特にheap使用量に注意する。

---

# 108. Streamingを優先する処理

可能な限り、

- file hashing
- chunking
- pack writing
- tree construction
- large diff output

をstreaming可能にする。

巨大fileを一度にBuffer化しない。

---

# 109. Node.js / TypeScript

実装言語:

    TypeScript

runtime:

    Node.js

を第一候補とする。

理由:

- filesystem API
- stream
- crypto
- worker_threads
- CLI
- cross-platform support

を利用できる。

---

# 110. 依存関係

初期実装ではNode.js標準APIを最大限利用する。

優先:

    node:fs
    node:path
    node:crypto
    node:stream
    node:zlib
    node:buffer
    node:worker_threads

外部依存は性能・機能上明確な理由がある場合のみ追加する。

---

# 111. Parallelism

CPU-heavy処理:

- hashing
- CDC
- fingerprint
- diff

についてworker_threadsを将来的に検討する。

ただし初期実装では、

    single-thread correctness

を優先する。

parallelizationはbenchmarkで必要性を確認してから導入する。

---

# 112. Hashing

BLAKE3 は WASM / ネイティブモジュール、SHA-256 は Node.js crypto を利用する。

大量hashでCPUがボトルネックになる場合、

- streaming hash
- batching
- worker_threads
- BLAKE3 (並列ハッシュ計算)

を検討する。

---

# 113. Security

CIDは単なるcache keyではなくobject integrityに利用する。

そのためcryptographic hashを使用する。

repository objectを読み込んだとき、

    actual CID
    vs
    expected CID

を検証可能にする。

---

# 114. Path Security

filesystemからrepositoryへpathを保存するとき、

- `..`
- absolute path
- NUL
- platform-specific separator
- case sensitivity

などをcanonicalizeする。

repository内pathはplatform-independent representationを検討する。

---

# 115. Windows / Linux / macOS

core object formatはplatform-independentとする。

filesystem metadataについてはplatform差を吸収する。

path separatorはrepository内部では `/` を第一候補とする。

---

# 116. Symlink

symlinkは通常fileとは別typeとして扱う。

例えば、

    Symlink {
        target
    }

のようなobjectを持つ。

symlink targetを通常のfile contentとして扱わない。

---

# 117. Executable Bit

Unix executable bitなどはFile metadataとして扱う。

content identityとは分離する。

---

# 118. Case Sensitivity

repository内部ではcase-sensitiveなpathを基本とする。

Windowsなどcase-insensitive filesystem上では、

    Foo
    foo

が同時に存在できない問題をstatus時に検出する。

---

# 119. Object Graph

最終的な概念図:

    Ref
     │
     ▼
    Commit
     │
     ▼
    Directory Root
     │
     ├── Directory
     │    ├── File
     │    │    └── Sequence Root
     │    │         └── Sequence Node
     │    │              └── Chunk
     │    │                   └── bytes
     │    │
     │    └── Directory
     │
     └── ...

すべてのNodeはCIDで参照される。

---

# 120. Diff Engine

概念:

    Commit A
       │
       ▼
    Root A
       │
       │ compare
       ▼
    Root B
       ▲
       │
    Commit B

compare:

    if CID equal:
        equal

    else:
        compare node type

        if directory:
            compare entries

        if file:
            compare metadata
            compare content root

        if sequence:
            align children

        if chunk:
            byte diff

---

# 121. Directory Diff

Directory diffでは、

1. 同名entryを比較
2. CID一致ならskip
3. CID不一致ならrecursive
4. old-only entryをdelete
5. new-only entryをadd
6. content CID一致の異なるpathをrename candidate

とする。

---

# 122. File Diff

File diff:

    metadata compare
    +
    content tree compare

metadataのみ変更ならcontent treeには降りない。

content rootが同一ならcontent diff不要。

---

# 123. Sequence Diff

Sequence diff:

    CID equal
        -> skip

    children align
        -> exact anchors

    reordered anchors
        -> move candidates

    unmatched:
        -> fingerprint candidates

    similar:
        -> recurse

    leaf:
        -> byte diff

---

# 124. Diff Renderer

内部diff resultと表示形式を分離する。

内部:

    Equal
    Insert
    Delete
    Replace
    MoveCandidate
    RenameCandidate
    CopyCandidate

表示:

    text diff
    binary summary
    file diff
    directory diff
    machine-readable JSON

など。

---

# 125. Machine-readable Diff

将来的に、

    sfvcs diff --json

を提供する。

内部representationをそのまま外部APIにしない。

versioned schemaにする。

---

# 126. API Design

core libraryとCLIを分離する。

概念:

    packages/core
    packages/cli

など。

ただしmonorepo構成は実装時に決定する。

---

# 127. Core API候補

例えば概念上、

    Repository.open()
    Repository.init()
    repository.readObject()
    repository.writeObject()
    repository.createSnapshot()
    repository.commit()
    repository.diff()
    repository.gc()

など。

実際のAPI名は実装時に確定する。

---

# 128. Error Handling

repository破損と通常の変更を区別する。

例:

    MissingObjectError
    InvalidObjectError
    HashMismatchError
    InvalidReferenceError
    UnsupportedFormatError
    ConcurrentUpdateError

など。

---

# 129. Transaction / Recovery

object write途中でprocessがkillされても、

    refが古い状態

ならrepositoryは基本的に利用可能であるべき。

未到達objectはGCで回収する。

---

# 130. Repository Lock

同時commitやGCでは競合する。

必要に応じて、

    .sfvcs/lock

などのrepository lockを利用する。

ただしref updateは可能ならCASで処理する。

---

# 131. Crash Safety

最低限、

    write object
      ↓
    fsync必要箇所
      ↓
    update ref

という順序を守る。

どこまでfsyncするかは性能とのトレードオフとして検討する。

---

# 132. GC安全性

GCとcommitが同時に動くと、

    commit中のobject

を誤って削除する危険がある。

そのためGC中はcommitをlockするか、

    generation / transaction marker

などを利用して安全性を確保する。

---

# 133. Research References

設計時に参考にする主要研究・実装。

## Prolly Tree / Noms

NomsのProlly Tree説明。

重要な知見:

- content-defined chunking
- recursive chunking
- immutable snapshots
- structural sharing
- 約4 KiB平均chunk
- 64 byte rolling window
- `1 / 4096` boundary probability

---

## FastCDC

「FastCDC: a Fast and Efficient Content-Defined Chunking Approach for Data Deduplication」

重要な知見:

- CDCの高速化
- min-size skip
- normalization
- chunk size distribution制御

---

## Dolt / Prolly Tree

DoltのProlly Tree設計・解説。

重要な知見:

- Noms型Prolly Treeの実用化
- chunk size distributionの改善
- boundary probabilityのnormalization
- content-addressed tree
- efficient diff / merge

---

## Persistent Data Structures

Driscoll et al.

「Making Data Structures Persistent」

重要な知見:

- immutable versioned data structures
- path copying
- persistent tree

`sfvcs`のsnapshot / structural sharingの理論的背景として参考にする。

---

## Structural Diff

「An Efficient Algorithm for Type-Safe Structural Diffing」

重要な知見:

- tree structural diff
- sharing
- source/target tree比較
- linear-time structural diffの考え方

---

## Fine-grained Source Code Differencing

「Fine-grained and Accurate Source Code Differencing」

重要な知見:

- tree-based diff
- move action
- developer intentに近いedit script

ただし`sfvcs`ではASTをcanonical structureには使用しない。

---

## Move-aware Replicated Trees

Kleppmann et al.

「A highly-available move operation for replicated trees」

重要な知見:

- tree move
- concurrent move
- tree structure semantics
- moveを扱う際のcycleなどの問題

主に将来のmerge / CRDT研究の参考資料とする。

---

# 134. 設計上の重要な判断

現時点で以下を基本方針として固定する。

### 採用

- content-addressed objects
- immutable snapshots
- persistent structural sharing
- recursive sequence tree
- content-defined chunking
- adaptive depth
- CID-based equality
- multi-resolution diff
- derived diff index
- derived fingerprint index
- MOVEのdiff時導出
- streaming construction
- binary / text共通core

### 採用しない

- ASTをcore storageに使用
- lineをcore storageに使用
- absolute positionをobject identityに使用
- MOVE operationをcanonical storageに保存
- approximate fingerprintをidentityとして使用
- 完全最小edit scriptを初期目標にする
- Git互換object format
- Git互換repository

---

# 135. 確定パラメータ仕様

本設計書および詳細仕様書に基づき確定された標準パラメータ値を以下に示す。

    Hash アルゴリズム:
        BLAKE3 (33バイト固定長 CID: [0x02][32バイトHash])
        (SHA-256 [0x01] も同等にサポート)

    パス正規化ルール:
        Unicode NFC (Normalization Form C) 統一 / POSIX スラッシュ区切り

    Leaf CDC (FastCDC):
        MIN_SIZE    = 2 KiB (2,048 B)
        AVG_SIZE    = 8 KiB (8,192 B)
        MAX_SIZE    = 64 KiB (65,536 B)
        Window Size = 48 bytes
        Hash        = Gear Hash (256 x 64bit uint)

    Internal Sequence Node:
        目標ファンアウト = 64
        グループ境界条件 = (Child_CID_uint32 & 0x3F) == 0
        最大ファンアウト = 128

    Small File Inline 閾値:
        1,024 バイト以下 (SFFL 内に直列格納)

    Winnowing Fingerprint:
        k-gram = 16 bytes
        window = 32 bytes
        Jaccard 類似度閾値 = 0.65

    Packfile:
        Magic = "SFPK"
        圧縮   = Thin Delta (ofs-delta / ref-delta) + Deflate

---

# 136. 今後の開発・ベンチマーク検証計画

基本仕様および詳細アルゴリズムの定義が完了したため、次のフェーズでは各パラメータの実際のワークロード下でのベンチマーク検証およびチューニングを行う。

1. **FastCDC パラメータ測定 (MIN/AVG/MAX 組み合わせ比較)**
2. **Prolly Tree ファンアウト密度とツリー深さの最適化**
3. **大規模リポジトリにおける Multi-resolution Diff の処理時間およびノード探索数の計測**
4. **Winnowing Fingerprint による Move 検出の適合率・再現率評価**

---

# 137. 実験計画

まず以下の独立実験を作る。

## Experiment A: CDC

入力:

    real repository files

比較:

    4 KiB
    8 KiB
    16 KiB
    32 KiB

出力:

    chunk count
    distribution
    boundary stability
    storage size

---

## Experiment B: Internal Tree

同一leaf sequenceに対して、

    fixed fanout
    size based
    CID hash based
    recursive CDC

を比較。

測定:

    tree depth
    node count
    average node size
    changed nodes after edit
    diff traversal count

---

## Experiment C: Diff

変更:

- insertion
- deletion
- replacement
- move
- move + modify
- cross-file move
- rename

測定:

    visited nodes
    CID comparisons
    fingerprint comparisons
    runtime
    memory

---

## Experiment D: Fingerprint

候補:

    rolling hash
    SimHash
    MinHash
    winnowing

について、

    candidate recall
    false positive
    CPU cost
    memory cost

を測定。

---

# 138. 「最適」の定義

`sfvcs`での最適化は単純なCPU時間最小化ではない。

総合的に、

    storage efficiency
    +
    snapshot construction speed
    +
    diff speed
    +
    memory usage
    +
    boundary stability
    +
    implementation complexity

を評価する。

特に、

    storageを10%削減するために
    diffを10倍遅くする

ような変更は採用しない。

---

# 139. 設計の中心となる目標

最終的な理想形は、

    小さな変更
        ↓
    小さな構造変更
        ↓
    小さな保存コスト
        ↓
    小さなdiff探索範囲

である。

逆に、

    1 byte変更
        ↓
    巨大なchunk再分割
        ↓
    巨大なtree変更
        ↓
    巨大なdiff

となる設計は避ける。

---

# 140. 大規模moveへの対応

以下を重要なユースケースとする。

    [A][B][C][D][E][F]
             ↓
    [D][E][F][A][B][C]

canonical storageでは、

    A B C D E F

と

    D E F A B C

という2つのsequence treeを生成するだけ。

CIDは再利用される。

diffでは、

    D E F

と

    A B C

のposition変更を検出する。

これによりMOVEの保存コストを特別に必要としない。

---

# 141. 部分変更への対応

例えば、

    Old:
    A B C D E

    New:
    A B X D E

なら、

    A
    B
    D
    E

のCIDを可能な限り再利用する。

CとXだけが異なる。

---

# 142. 大きなblock + 部分変更

例えば、

    Old:
    [巨大Block A]
    [巨大Block B]
    [巨大Block C]

    New:
    [巨大Block A']
    [巨大Block C]
    [巨大Block B]

の場合、

- Aはmodified subtree
- Bはmove
- Cはunchanged

として扱える可能性がある。

multi-resolution treeの重要なユースケース。

---

# 143. File rename + content modification

例えば、

    old/foo.ts

が

    new/bar.ts

にrenameされ、同時に一部変更された場合、

File root CIDは異なる。

そのため単純なroot CID一致だけではrenameを検出できない。

この場合、

    subtree overlap
    fingerprint
    neighboring context
    size
    path similarity

などを利用してrename candidateを生成する。

---

# 144. 重要な制約

「最終状態から過去の操作履歴を完全復元する」ことは目標にしない。

diffは、

    plausible structural explanation

を生成する。

したがって、

    MOVE

    RENAME

    COPY

などは「状態差分を人間に説明するための意味付け」である。

---

# 145. Repository formatとdiff format

これらを明確に分離する。

Repository:

    immutable state

Diff:

    interpretation of two states

これによりdiffアルゴリズムを変更してもrepository formatを変更する必要がない。

---

# 146. 将来的なdiff algorithm交換

同じobject graphに対して、

    diff-v1
    diff-v2
    diff-move-aware
    diff-fast
    diff-human

など複数のdiff strategyを実装可能にする。

これは`sfvcs`の重要な利点とする。

---

# 147. 互換性

object format:

    versioned

tree algorithm:

    versioned

diff algorithm:

    independent

とする。

特に、

    diff algorithmの変更

ではrepository objectをmigrationしない。

---

# 148. 監査可能性

`sfvcs fsck`で、

    ref
      -> commit
      -> directory
      -> file
      -> sequence
      -> chunk

の全参照を辿れるようにする。

CID mismatchを検出する。

---

# 149. Debugging

開発時には、

    sfvcs debug object <cid>

    sfvcs debug tree <cid>

    sfvcs debug diff <old> <new>

などを検討する。

特にtree可視化はアルゴリズム開発に重要。

---

# 150. Tree Debug Output

例えば、

    Root CID
    ├─ Node CID ... [length=...]
    │  ├─ Chunk CID ... [8.2 KiB]
    │  ├─ Chunk CID ... [7.4 KiB]
    │  └─ Chunk CID ... [9.1 KiB]
    └─ Node CID ...

のように表示できるようにする。

---

# 151. Benchmark Tool

プロジェクト内部に、

    benchmark/

を用意する。

例:

    benchmark/chunking.ts
    benchmark/tree.ts
    benchmark/diff.ts
    benchmark/move.ts
    benchmark/repository.ts

研究値を比較できるようにする。

---

# 152. Regression Benchmark

一度速くなった処理が将来遅くならないよう、

    benchmark baseline

を保存する。

特に、

- hashing
- chunking
- tree construction
- diff

を継続的に測定する。

---

# 153. 正しさのテスト

Property-based testingを検討する。

例えば、

    encode(decode(object)) == canonical object

    CID(object) == CID(object)

    decode(encode(bytes)) == bytes

    snapshot(root) == expected files

など。

---

# 154. Tree Property

任意のsequenceについて、

    flatten(build(bytes)) == bytes

が成立しなければならない。

また、

    CID(build(bytes))

が同じ入力に対して常に同じであることを保証する。

---

# 155. Structural Sharing Property

同じchunkが複数snapshotに存在する場合、

    CID(oldChunk) == CID(newChunk)

であること。

不要なobject再生成が発生していないかをbenchmarkで検査する。

---

# 156. Diff Correctness

Old/Newをdiffした結果を適用した場合、

    applyDiff(old, diff(old,new)) == new

となることを基本propertyとする。

ただしMOVE/COPY/RENAMEはsemantic ambiguityがあるため、内部diff representationとhuman rendererを分離する。

---

# 157. Canonical Storageの最終イメージ

    bytes
      │
      ▼
    CDC
      │
      ▼
    Chunk objects
      │
      ▼
    Sequence nodes
      │
      ▼
    Sequence root
      │
      ▼
    File node
      │
      ▼
    Directory tree
      │
      ▼
    Snapshot
      │
      ▼
    Commit
      │
      ▼
    Ref

すべてimmutable object。

---

# 158. Diffの最終イメージ

    old root
        │
        ├── CID equal ──────────────┐
        │                           │
        └── CID different           │
                │                   │
          child alignment           │
                │                   │
          exact anchors             │
                │                   │
          move candidates           │
                │                   │
          fingerprint candidates    │
                │                   │
          recursive descent         │
                │                   │
             leaf diff              │
                │                   │
                └───────────────> diff result

---

# 159. 最重要設計思想

`sfvcs`では、

    「変更を保存する」

のではなく、

    「変更後の状態を構造的に保存し、
     変更前との関係を高速に発見する」

ことを基本思想とする。

このため、

    storage

と

    diff

を意図的に分離する。

---

# 160. 現時点での暫定結論

現段階では以下を第一案とする。

    Repository
      ↓
    Commit
      ↓
    Persistent Directory Tree
      ↓
    File Node
      ↓
    Recursive Content-defined Sequence Tree
      ↓
    Content-defined Chunk
      ↓
    Raw Bytes

そして、

    Multi-resolution Diff Engine
      ↓
    CID equality
      ↓
    structural alignment
      ↓
    fingerprint candidate search
      ↓
    move / rename / copy candidate
      ↓
    leaf diff

を別レイヤーとして実装する。

---

# 161. 今後の研究順序

次の順序で調査・実験する。

1. FastCDC / CDC
2. Noms Prolly Tree
3. Dolt Prolly Tree
4. internal node boundary
5. fanout / node size
6. persistent sequence tree
7. structural diff
8. sequence alignment
9. move-aware diff
10. fingerprint algorithms
11. approximate matching
12. pack storage
13. merge

特に最初の5項目を固めてからdiff algorithmを実装する。

理由は、

    tree structure

がdiff性能の上限を決めるため。

---

# 162. 未確定事項の扱い

本書に記載された以下の項目は、現時点では仕様確定ではない。

- chunk size
- chunk distribution
- CDC algorithm
- internal boundary algorithm
- internal node size
- fingerprint algorithm
- diff alignment algorithm
- move threshold
- pack format
- compression
- small file threshold

これらは、

    Research
      ↓
    Prototype
      ↓
    Benchmark
      ↓
    Decision

の順に決定する。

---

# 163. 設計変更の原則

ベンチマーク結果によって本書の数値・アルゴリズムが変更されることを許容する。

ただし変更時には、

    なぜ変更したか
    何を比較したか
    何が改善したか
    何が悪化したか

を記録する。

「なんとなく8 KiB」ではなく、

    8 KiB was selected because ...

という形で設計判断を追跡可能にする。

---

# 164. 最終目標

`sfvcs`が目指すものは、

- Git互換ではない
- 単なるGit cloneでもない
- 単なるdeduplicating filesystemでもない
- 単なるdiff toolでもない

独自の、

    content-addressed
    persistent
    structurally shared
    multi-resolution
    move-aware

なversion control systemである。

最終的には、

    大きなrepository
    +
    小さな変更
    +
    大きなblock移動
    +
    file rename
    +
    cross-file reuse

に対して、

    保存量
    diff時間
    memory使用量

を可能な限り変更量に近づけることを目標とする。

---

# 165. 現時点での一言まとめ

sfvcsのcoreは、

    「content-defined recursive persistent tree」

である。

diffのcoreは、

    「multi-resolution structural alignment」

である。

move / rename / copyは、

    「canonical dataではなくdiffから導出されるsemantic information」

である。

fingerprint indexは、

    「canonical dataではなく再構築可能なsearch acceleration layer」

である。

この4つを混ぜないことを、設計上の最重要原則とする。

---

# 166. 参考文献・参考資料

本設計は、以下の研究論文・実装・技術資料を参考としている。

本節に掲載する資料は、必ずしも `sfvcs` がその設計・アルゴリズムをそのまま採用することを意味しない。

特に数値については、

    既存研究・実装の値
        ↓
    sfvcsの初期値
        ↓
    実データによるベンチマーク
        ↓
    最終値

という手順で決定する。

---

## 166.1 Persistent Data Structures

### Making Data Structures Persistent

James R. Driscoll, Neil Sarnak, Daniel D. Sleator, Robert E. Tarjan.

Journal of Computer and System Sciences, Volume 38, Issue 1, 1989, Pages 86-124.

DOI:

    https://doi.org/10.1016/0022-0000(89)90034-2

Publisher / publication information:

    https://www.sciencedirect.com/science/article/pii/0022000089900342

Author / institutional information:

    https://collaborate.princeton.edu/en/publications/making-data-structures-persistent/

Open PDF:

    https://www.cs.cmu.edu/~sleator/papers/making-data-structures-persistent.pdf

関連する設計:

- persistent data structure
- immutable version
- structural sharing
- path copying
- 複数versionの保持
- snapshot設計

`sfvcs`では、過去snapshotをimmutable object graphとして保持し、
変更部分だけ新しいNodeを作成して既存Nodeを共有するという設計思想の基礎資料とする。

---

## 166.2 Prolly Tree / Noms

### Noms

Attic Labs.

Nomsはversioned / forkable / syncable databaseであり、
Prolly Treeを利用したcontent-addressedなデータ構造を実装していた。

Repository:

    https://github.com/attic-labs/noms

Prolly Treeを含む設計説明:

    https://github.com/attic-labs/noms/blob/master/doc/intro.md

関連する設計:

- Prolly Tree
- probabilistic B-Tree
- content-defined chunking
- recursive chunking
- history independence
- content addressing
- structural sharing
- efficient diff
- efficient merge
- efficient sync

特に以下の値・性質を参考にする。

    average chunk size ≈ 4 KiB
    rolling hash window = 64 bytes
    boundary probability ≈ 1 / 4096

Nomsの説明では、64-byte windowと約4 KiBの平均chunk sizeから、
1 bitの変更によってchunk boundaryが移動する確率を約1.6%としている。

また、chunkingをrecursiveに適用することでmulti-level treeを構築する。

4-level treeの例として、

    4096^4 ≈ 281 TB

という容量が示されている。

---

## 166.3 Noms Prolly Treeに関する追加検討

### Some notes and questions on Prolly Trees

Noms GitHub Issue #3878.

    https://github.com/attic-labs/noms/issues/3878

Prolly Treeのboundary生成についての議論。

特に、

- rolling hashをprimary boundary mechanismとして使用することの問題
- deterministic hash-linked B-Tree
- boundary stability
- chunking方式

について参考にする。

`sfvcs`では、leafのbyte-level CDCとinternal nodeのboundary生成を
必ずしも同じアルゴリズムにする必要はない、という判断の根拠として参照する。

---

## 166.4 FastCDC

### FastCDC: A Fast and Efficient Content-Defined Chunking Approach for Data Deduplication

Wen Xia, Yukun Zhou, Hong Jiang, Dan Feng, Yu Hua, Yuchong Hu,
Yucheng Zhang, Qing Liu.

2016 USENIX Annual Technical Conference (USENIX ATC '16),
Pages 101-114.

USENIX:

    https://www.usenix.org/conference/atc16/technical-sessions/presentation/xia

Paper PDF:

    https://www.usenix.org/system/files/conference/atc16/atc16-paper-xia.pdf

DBLP:

    https://dblp.org/rec/conf/usenix/XiaZJFHHLZ16.html

関連する設計:

- Content-Defined Chunking
- Gear-based hashing
- minimum chunk size
- skipping sub-minimum boundaries
- chunk-size normalization
- CDC performance
- deduplication

FastCDCでは、

- hash判定の簡略化
- sub-minimum chunk boundaryのskip
- chunk-size distribution normalization

を組み合わせることでCDCの高速化を行っている。

`sfvcs`ではleaf chunking algorithmの第一候補としてFastCDC系を検討する。

---

## 166.5 Dolt / Prolly Tree

### Prolly Trees

DoltHub.

    https://www.dolthub.com/blog/2024-03-03-prolly-trees/

関連する設計:

- Noms Prolly Treeの実用上の問題
- chunk-size distribution
- probabilistic tree
- structural sharing
- content addressing
- diff
- merge

特に重要なのは、Noms型の単純なrolling hashによるchunkingでは、

    average ≈ 4 KiB

でもchunk sizeが幾何分布となり、

    小さいchunkが多数
    大きいchunkが少数

という分布になりやすい問題を実際の実装経験から指摘している点。

`sfvcs`では、

    average chunk size

だけではなく、

    median
    p90
    p99
    min
    max
    boundary stability

などを評価する方針の根拠とする。

---

## 166.6 Current Prolly Tree Implementation

### prolly

Content-addressed ordered map based on Prolly Trees.

Repository:

    https://github.com/crabbuild/prolly

関連する設計:

- content-addressed ordered map
- immutable snapshots
- structural sharing
- efficient diff
- efficient merge
- tree statistics
- fanout
- serialized node size
- persistent updates

特に、

    tree shape
    fill factor
    fanout
    serialized size

などを実装上の評価対象としている点を参考にする。

---

## 166.7 lakeFS Versioning Internals

### Versioning Internals

lakeFS Documentation.

    https://docs.lakefs.io/v1.61/understand/how/versioning-internals/

Current internals overview:

    https://docs.lakefs.io/concepts/internals/

関連する設計:

- content-addressed storage
- immutable objects
- Merkle tree
- Range
- Meta Range
- structural sharing
- commit効率
- diff効率
- recursive hierarchy

lakeFSでは、

    Range
        ↓
    Meta Range

という2層Merkle treeを利用している。

また、

    Meta Range
        ↓
    Meta Range
        ↓
    Meta Range

とrecursiveにすることで任意の深さに拡張可能であることが説明されている。

`sfvcs`における、

    fixed L0/L1/L2/L3

ではなく、

    recursive / adaptive tree

を採用する設計の参考資料とする。

---

## 166.8 Structural Diff

### An Efficient Algorithm for Type-Safe Structural Diffing

Victor Cacciari Miraldo, Wouter Swierstra.

Proceedings of the ACM on Programming Languages,
Volume 3, ICFP, 2019.

DOI:

    https://doi.org/10.1145/3341717

Utrecht University:

    https://research-portal.uu.nl/en/publications/an-efficient-algorithm-for-type-safe-structural-diffing/

Full text repository:

    https://dspace.library.uu.nl/handle/1874/395202

関連する設計:

- structural diff
- tree diff
- sharing
- source / target tree comparison
- linear-time structural diff
- diff granularity

この研究では、UNIX diffのようなline-based diffとは異なり、
データ構造そのものを利用してdiffする方法を扱っている。

`sfvcs`の、

    root
      ↓
    subtree
      ↓
    child
      ↓
    leaf

と必要な部分だけ解像度を下げていく
multi-resolution diffの考え方に関連する。

ASTを利用した例が中心だが、
`sfvcs`ではASTそのものをcanonical storageに採用しない。

---

## 166.9 Fine-grained / Move-aware Source Diff

### Fine-grained and accurate source code differencing

Jean-Rémy Falleri, Floréal Morandat, Xavier Blanc,
Matias Martinez, Martin Monperrus.

29th ACM/IEEE International Conference on Automated Software Engineering,
ASE 2014, Pages 313-324.

DOI:

    https://doi.org/10.1145/2642937.2642982

ACM:

    https://doi.org/10.1145/2642937.2642982

DBLP:

    https://dblp.org/rec/conf/kbse/FalleriMBMM14.html

関連する設計:

- fine-grained diff
- tree matching
- edit script
- move action
- developer intent
- moved code detection

特に、

    MOVE

を差分の意味として扱う必要性を検討する際の参考資料。

`sfvcs`ではASTをcanonical structureとして利用しないため、
アルゴリズムをそのまま採用するものではない。

---

## 166.10 Diff Algorithm Comparison

### How different are different diff algorithms in Git?

Yusuf Sulistyo Nugroho, Hideaki Hata, Kenichi Matsumoto.

Empirical Software Engineering,
Volume 25, 2020, Pages 790-823.

DOI:

    https://doi.org/10.1007/s10664-019-09772-z

Springer:

    https://link.springer.com/article/10.1007/s10664-019-09772-z

arXiv:

    https://arxiv.org/abs/1902.02467

DBLP:

    https://dblp.org/rec/journals/ese/NugrohoHM20

関連する設計:

- Myers
- Minimal
- Patience
- Histogram
- diff結果の違い
- diff quality
- source code change extraction

異なるdiff algorithmが同じ2ファイルに対して異なるdiffを生成し得ることを実証している。

`sfvcs`では、

    「唯一絶対のdiff」

を前提とせず、

    diff algorithm
    +
    semantic interpretation

を分離する設計の参考とする。

---

## 166.11 Move-aware Replicated Trees

### A highly-available move operation for replicated trees

Martin Kleppmann, Dominic P. Mulligan,
Victor B. F. Gomes, Alastair R. Beresford.

IEEE Transactions on Parallel and Distributed Systems,
Volume 33, Issue 7, 2021, Pages 1711-1724.

DOI:

    https://doi.org/10.1109/TPDS.2021.3118603

Author page:

    https://martin.kleppmann.com/2021/10/07/crdt-tree-move-operation.html

Cambridge repository:

    https://www.repository.cam.ac.uk/items/70401d54-e309-420a-a7d1-f597e6ae8523

関連実装:

    https://github.com/trvedata/move-op

関連する設計:

- tree move
- subtree move
- concurrent move
- tree consistency
- cycle prevention
- move semantics

`sfvcs`ではdistributed CRDTを初期実装に採用しないが、
将来的なmergeやtree move semanticsの研究資料として利用する。

---

## 166.12 Version Control as a General Persistent Data Model

### Version Control Is for Your Data Too

Gowtham Kaki, KC Sivaramakrishnan, Suresh Jagannathan.

3rd Summit on Advances in Programming Languages (SNAPL 2019),
LIPIcs Volume 136, Article 8, 2019.

DOI:

    https://doi.org/10.4230/LIPIcs.SNAPL.2019.8

Dagstuhl:

    https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.SNAPL.2019.8

PDF:

    https://drops.dagstuhl.de/opus/volltexte/2019/10551/pdf/LIPIcs-SNAPL-2019-8.pdf

関連する設計:

- versioned data
- persistent versions
- provenance
- 3-way merge
- version control as a general data model

CARMOTというversion-control-based distributed programming modelを提案している。

`sfvcs`において、

    version
    ancestry
    snapshot
    merge

を分離して考える際の参考資料とする。

---

## 166.13 SirixDB

### SirixDB

SirixDB project.

Repository:

    https://github.com/sirixdb/sirix

Architecture:

    https://sirix.io/docs/architecture.html

Project site:

    https://sirix.io/

関連する設計:

- immutable snapshots
- persistent tree
- structural sharing
- copy-on-write
- append-only storage
- revision management
- page-level versioning
- variable-sized fragments

`sfvcs`における、

    immutable snapshot
    +
    structural sharing
    +
    append-oriented storage

の実装上の比較対象として利用する。

---

# 167. 参考資料から得られた主要な設計知見

上記資料から、`sfvcs`の設計に特に重要な知見を以下に整理する。

## 167.1 Persistent Data Structure

過去versionをimmutableに保持し、
変更部分だけ新しい構造を作ることで、
過去versionと現在versionの構造を共有できる。

参考:

    Driscoll et al. (1989)

---

## 167.2 Content Addressing

内容からidentityを作ることで、

    同一内容
        =
    同一CID

とできる。

これによりsnapshot間・file間・subtree間で自然なdeduplicationが可能になる。

参考:

    Noms
    Dolt
    lakeFS

---

## 167.3 Content-defined Chunking

固定byte位置でchunkを切るのではなく、
内容からboundaryを決定することで、
挿入・削除による後続chunkの全崩壊を防ぎやすくする。

参考:

    Noms
    FastCDC
    Dolt

---

## 167.4 Recursive Chunking

leaf chunkだけでなく、
chunk列をさらにchunkingすることでrecursive treeを構築できる。

参考:

    Noms Prolly Tree

---

## 167.5 Boundary Stability

CDCでは平均chunk sizeだけでは不十分。

小さな変更によって、

    どれだけ遠くまでboundaryが変化するか

を評価する必要がある。

参考:

    Noms
    Dolt
    FastCDC

---

## 167.6 Chunk Size Distribution

単純なrolling hashでは、
target size付近に均等に分布するとは限らない。

そのため、

    chunk size distribution

を独立した設計パラメータとして扱う。

参考:

    Dolt
    FastCDC

---

## 167.7 Structural Diff

treeの構造を利用することで、
変更されていないsubtreeを丸ごとskipできる。

参考:

    Cacciari Miraldo & Swierstra (2019)

---

## 167.8 Move-aware Diff

MOVEは単なるDELETE + INSERTとは異なる意味を持ち得る。

一方、最終状態だけでは、

    move
    copy
    delete + insert

を常に一意に判定できない。

したがって、

    canonical state

と

    diff semantics

を分離する。

参考:

    Falleri et al. (2014)
    Kleppmann et al. (2021)

---

## 167.9 Diff Algorithm is Not Unique

同じ2つの状態でも、
diff algorithmによって異なる変更結果が得られる。

したがってrepository formatとdiff algorithmを分離する。

参考:

    Nugroho et al. (2020)

---

## 167.10 Recursive Merkle Structure

repository全体を単一の巨大objectとして扱わず、
複数のcontent-addressed subtreeに分割することで、
変更された部分だけ新しいobjectを作れる。

参考:

    Noms
    lakeFS
    Dolt
    SirixDB

---

# 168. 参考資料の位置付け

参考資料は以下のカテゴリに分類する。

| 資料                         | 主な用途                               |
| ---------------------------- | -------------------------------------- |
| Driscoll et al.              | Persistent Data Structure              |
| Noms                         | Prolly Tree / recursive chunking       |
| FastCDC                      | Leaf CDC                               |
| Dolt                         | Chunk distribution / Prolly Tree実用化 |
| prolly                       | 現行Prolly Tree実装比較                |
| lakeFS                       | Recursive Merkle / snapshot sharing    |
| Cacciari Miraldo & Swierstra | Structural Diff                        |
| Falleri et al.               | Move-aware Diff                        |
| Nugroho et al.               | Diff algorithm comparison              |
| Kleppmann et al.             | Tree Move semantics                    |
| CARMOT                       | Versioned state / 3-way merge          |
| SirixDB                      | Persistent tree / append-only storage  |

---

# 169. 参考資料に関する注意

参考資料に記載された数値やアルゴリズムは、
そのまま `sfvcs` の仕様値とはしない。

特に、

    chunk size
    fanout
    node size
    fingerprint threshold
    move threshold
    diff algorithm

については、各資料の目的・データ構造・workloadが異なる。

したがって、

    論文値を採用
        ↓
    sfvcs prototype
        ↓
    real-world benchmark
        ↓
    parameter optimization

という手順を取る。

---

# 170. 今後追加する参考資料

今後の調査で以下の分野について有力な資料が見つかった場合、
本節へ追加する。

- Content-Defined Chunking
- FastCDC variants
- Gear hash
- Rabin fingerprint
- chunk boundary stability
- Prolly Tree
- Merkle Search Tree
- persistent sequence
- sequence CRDT
- structural diff
- sequence alignment
- move detection
- copy detection
- approximate matching
- fingerprint index
- MinHash
- SimHash
- winnowing
- LCS
- Myers
- Patience
- Histogram
- tree diff
- content-addressed storage
- persistent storage
- append-only storage
- pack files
- deduplication
- snapshot storage
- garbage collection
- three-way merge

---

# 171. 拡張アーキテクチャ要件および高度設計仕様

本節では、大規模リポジトリ、ブラウザ仮想環境、大容量資産、および高効率コミットグラフ走査における高度な機能拡張アーキテクチャ方針を定義する。

## 171.1 Bisect（二分探索バグ特定）および Blame（行単位追跡）
- **Bisect**: 有向非巡回コミットグラフ（DAG）におけるトポロジカルソートと親コミット比率計算に基づき、二分探索で回帰バグ発生コミットを $O(\log C)$ で特定。
- **Blame**: Prolly Tree の Sequence Node アライメントおよび Winnowing Fingerprint を用いて、ファイル跨ぎの MOVE/RENAME を追跡しながら行単位の著者・コミット provenance を算出。

## 171.2 Commit Graph インデックス構造
- `.sfvcs/objects/info/commit-graph` バイナリ形式により、コミットの世代番号（Generation Numbers / Corrected Commit Date）およびパス変更 Reachability Bloom Filter を保持。コミット履歴探索および Merge Base 計算を高速化。

## 171.3 大容量バイナリ資産管理 (sfvcs LFS)
- ギガバイト級のメディア・データセット等に対して、`File Node` 内に `CONTENT_LFS_POINTER` 構造を採用し、ローカルオブジェクトストアの肥大化を防止。外部ストレージ連携および Range Request ストリーミング転送をサポート。

## 171.4 ゼロ知識・リポジトリ暗号化 (Encryption-at-Rest)
- 外部リモートサーバーや IndexedDB / OPFS に保存する際、クライアント側で AES-256-GCM または XChaCha20-Poly1305 によりチャンク Payload を暗号化。マスターキー・鍵暗号化キー（KEK）・データ暗号化キー（DEK）のエンベロープ暗号化を採用。

## 171.5 バイナリデルタ圧縮規格 (VCDIFF / RFC 3284)
- パックファイル内の Thin Delta 圧縮において、RFC 3284 VCDIFF 規格準拠の Copy / Insert / Run 命令バイトコードを採用。大容量チャンク間の共通差分を高圧縮率で生成。

## 171.6 汎用言語非依存セマンティック 3-Way Merge
- JSON, YAML, TOML, Lockfile 等の構造化設定ファイルに対し、特定プログラミング言語に依存しない木構造・キーバリュースキーマベースのセマンティック統合アルゴリズムを適用。

## 171.7 ロックフリー並列 GC (Tri-Color Marking with Write Barrier)
- メインプロセスおよび Web Worker と並行してバックグラウンド GC を実行可能にするため、SATB (Snapshot-At-The-Beginning) 書き込みバリアを備えた三色（White, Grey, Black）マークアンドスウィープアルゴリズムを採用。

## 171.8 リアルタイム協調編集 (Fugue Sequence CRDT)
- Web ブラウザ上のオンライン IDE 等において、Prolly Tree Sequence Node と Fugue Sequence CRDT の状態ベクトルを統合し、スナップショット作成前の複数ユーザーリアルタイム並行編集を決定論的にアライメント。

## 171.9 Sparse Index および Pathspec Trie 検索高速化
- パス無視・属性判定を Prefix Trie (Pathspec Trie) で $O(K)$ 実行。`.sfvcs/index` 内に `Sparse Index` ディレクトリプレースホルダー CID を導入し、未変更サブツリー展開を $O(1)$ スキップ。

## 171.10 マルチリポジトリ・サブモジュール拡張
- `ENTRY_SUBMODULE` の再帰的 fetch / diff / merge、並びに Sparse Submodule Clone プロトコルによる大規模ワークスペース統合。

## 171.11 WASM サンドボックスプラグインおよびサーバーフック
- Web ブラウザ環境用の WASM サンドボックス拡張機能、および `pre-receive` / `update` / `post-receive` / `proc-receive` サーバーフックを定義。

## 171.12 キーリング管理および鍵失効リスト (CRL)
- 電子署名用 WebCrypto / GPG / Ed25519 鍵の信頼ストア、鍵失効リスト (CRL) 検証、および鍵ローテーション手順。

## 171.13 Racy Git 回避、アルゴリズム堅牢性および競合解決モデル
- **Racy Git 回避**: インデックス書き込み時刻と同値な `mtime` ファイルに対する強制コンテンツ再ハッシュ比較。
- **Prolly Tree 病的入力保護**: 悪意あるハッシュ不発入力に対する `MAX_TREE_DEPTH = 32` の深度制限フォールバック。
- **Concurrent Move / Delete 競合解法**: 移動と削除の平行競合におけるデータ消失防止・決定論的移動優先ルール。
- **Submodule 循環依存防止**: グラフ走査時の循環参照検知とスタックオーバーフロー回避。
- **エンベロープ暗号鍵ローテーション**: $O(1)$ 暗号化 DEK メタデータ更新。
- **ストレージ容量制限 LRU Eviction**: IndexedDB / OPFS での容量超過時 Derived Cache 自動解放。
