# sfvcs 基本設計書: Rebase / Cherry-Pick / Stash / Revert 3-Way 応用 & 状態永続化
(`bd-merge-rebase-state.md`)

---

# 1. 概要と目的

本設計書は、リビジョン系列の付け替え (Rebase)、単一コミットの適用 (Cherry-Pick)、作業一時退避 (Stash)、コミット打消し (Revert)、およびテキスト/構造コンフリクト表現・状態永続化ファイル (State Machine Files) の基本設計書である。

本書は `doc/specs/spec-merge.md` の第5節 (5.1, 5.2), 第6節 (6.1, 6.2, 6.3, 6.4), 第10節 (10.1) および `doc/01_architecture/sfvcs-design.md` の関連仕様を完全網羅し、カプセル化された Workflow Merge Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Domain Service & Workflow Execution Layer** に属し、一連のマルチコミット変換ステートマシンを管理する。

```
+-----------------------------------------------------------------------+
|     CLI (rebase / cherry-pick / stash / revert)                       |
+-----------------------------------------------------------------------+
                                   | Execute Workflow Sequence
                                   v
+-----------------------------------------------------------------------+
|      bd-merge-rebase-state (Workflow State Engine Service)            |
|  - TypeScript: State Machine Files Manager (.sfvcs/rebase-merge)      |
|  - Rust/WASM Core: Sequenced 3-Way Step Executor Engine               |
+-----------------------------------------------------------------------+
                                   | Read / Write State Files
                                   v
+-----------------------------------------------------------------------+
|                Index / Working Tree / Storage Layer                   |
+-----------------------------------------------------------------------+
```

---

# 3. Rebase, Cherry-Pick, Stash, Revert への 3-Way Merge 応用

### 3.1 Rebase (`sfvcs rebase <upstream>`)
1. リビジョン系列 $C_1, C_2, \dots, C_n$ を抽出し、各コミット $C_i$ について以下を実行:
   - Base = $\text{Parent}(C_i)$
   - Ours = カレント適用結果 (Upstream Head)
   - Theirs = $C_i$
2. 競合発生時、状態ファイル (`.sfvcs/rebase-merge/`) を永続化して中断し、ユーザーのコンフリクト解消待ちに移行。

### 3.2 Cherry-Pick (`sfvcs cherry-pick <commit>`)
- Base = $\text{Parent}(C)$, Ours = `HEAD`, Theirs = $C$ を用いた 3-Way Structural Merge。

### 3.3 Stash (`sfvcs stash [save|pop]`)
- インデックスとワーキングツリーのコミットオブジェクト（2 つまたは 3 つの特殊 Commit Node）を生成し、Reflog `refs/stash` に記録。

### 3.4 Revert (`sfvcs revert <commit>`)
- Base = $C$, Ours = `HEAD`, Theirs = $\text{Parent}(C)$ の反転 3-Way Merge。

---

# 4. コンフリクト表現形式 & 状態永続化ファイル

### 4.1 テキストコンフリクトマーカー (In-file Conflict Markers)
```text
<<<<<<< OURS (HEAD)
current changes
||||||| BASE
common ancestor content
=======
incoming changes
>>>>>>> THEIRS (commit-hash)
```

### 4.2 状態永続化ファイル (State Machine Files)
- `.sfvcs/MERGE_HEAD`: マージ中コミット CID。
- `.sfvcs/REBASE_HEAD`: リベース中カレントコミット CID。
- `.sfvcs/MERGE_MSG`: 自動生成マージメッセージ。
- `.sfvcs/sequencer/todo`: 残存コミットシーケンスキュー。

---

# 5. Rust / WASM Core & TypeScript インターフェース

```typescript
export interface RebaseManagerFacade {
  startRebase(upstreamBranch: string): Promise<{ status: "completed" | "conflict"; currentCommit?: Uint8Array }>;
  continueRebase(): Promise<{ status: "completed" | "conflict" }>;
  abortRebase(): Promise<void>;
}
```
