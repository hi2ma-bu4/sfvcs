# sfvcs 基本設計書: Reference 更新・CAS 並行制御 & Reflog & Lock 復旧
(`bd-storage-refs-lock.md`)

---

# 1. 概要と目的

本設計書は、ブランチ・タグ・HEAD 等の参照 (Reference) をアトミックに更新する Compare-And-Swap (CAS) 制御、参照変更履歴である Reflog 管理、並びに孤立ロック自動復旧 (Stale Lock Recovery) と Web Locks API 統合の基本設計書である。

本書は `doc/specs/spec-storage.md` の第5節 (5.1, 5.2), 第7節 (7.1, 7.2, 7.3) および `doc/01_architecture/sfvcs-design.md` の第171.13, 171.14節の仕様を完全網羅し、カプセル化された Reference & Lock Management Service モジュールとして詳細を定義する。

---

# 2. カプセル化境界とモジュール構造

本モジュールは **Storage Reference & Concurrency Layer** に位置し、参照整合性の保護および複数プロセス/スレッド間での排他制御を一括して担当する。

```
+-----------------------------------------------------------------------+
|              Domain Services (Commit / Branch / Merge / Rebase)       |
+-----------------------------------------------------------------------+
                                   | Update Ref (Branch, OldCid, NewCid)
                                   v
+-----------------------------------------------------------------------+
|        bd-storage-refs-lock (Reference & Lock Management Service)     |
|  - TypeScript: Reflog Appender & Lock File Lifecycle Manager          |
|  - Rust/WASM Core: Atomic CAS Evaluator & Stale Lock Detector Engine  |
+-----------------------------------------------------------------------+
                                   | Atomic File Lock / Web Locks API
                                   v
+-----------------------------------------------------------------------+
|                   Physical / Virtual FS Storage Layer                 |
+-----------------------------------------------------------------------+
```

---

# 3. CAS (Compare-And-Swap) による Branch 更新 & 孤立ロック復旧

### 3.1 CAS 更新アルゴリズムステップ
ブランチ `refs/heads/main` の参照更新時、アトミックなロックファイル (`refs/heads/main.lock`) を作成し、現在の最新 CID が要求元の予測 CID と一致する場合のみアトミックに置換する。

1. **Lock 獲得**: `.lock` ファイルを作成（`O_EXCL | O_CREAT`）。失敗した場合は Stale Lock 検証へ移行。
2. **CAS 検証**: 現存の CID と `expected_old_cid` が一致するか検証。不一致の場合は `RefLockedError` / `StaleCommitError` を発生。
3. **Atomic Commit**: 一時ファイルをリネーム (`renameSync`) して更新適用。

### 3.2 孤立ロック自動復旧 (Stale Lock Recovery)
プロセス クラッシュ等により残存した `.lock` ファイルに対し、TTL（例: 300 秒）を超過しており、かつプロセス ID (`pid`) が生存していない場合、自動的に孤立ロックとみなして消去・復旧する。

### 3.3 仮想 VCS (ブラウザ環境) での Web Locks API
ブラウザ環境においては、ファイルシステムレベルの `.lock` ファイルの代わりに `navigator.locks.request` API を使用して Web Worker 間・タブ間の排他制御を行う。

---

# 4. Reflog (参照履歴) フォーマット & 自動パージ・期限切れ

### 4.1 Reflog ファイル形式 (`.sfvcs/logs/refs/heads/<branch>`)
```text
0000000000000000000000000000000000000000000000000000000000000000 a1b2c3d4e5f67890a1b2c3d4e5f67890a1b2c3d4e5f67890a1b2c3d4e5f67890 Alice <alice@example.com> 1672531199 +0900 commit: Initial commit
```

### 4.2 Reflog Expiration (期限切れ設定)
`sfvcs gc` 実行時、`gc.reflogExpire` (デフォルト 90 日) および `gc.reflogExpireUnreachable` (デフォルト 30 日) を経過した古く到達不能な Reflog エントリを自動パージする。

---

# 5. Rust / WASM Core & TypeScript インターフェース

```rust
pub fn evaluate_stale_lock(lock_mtime_sec: i64, current_sec: i64, ttl_sec: i64) -> bool {
    (current_sec - lock_mtime_sec) > ttl_sec
}
```

```typescript
export interface ReferenceManagerFacade {
  readRef(refName: string): Promise<Uint8Array | null>;
  compareAndSwapRef(refName: string, expectedOldCid: Uint8Array | null, newCid: Uint8Array, message: string): Promise<boolean>;
  appendReflog(refName: string, oldCid: Uint8Array, newCid: Uint8Array, message: string): Promise<void>;
  purgeExpiredReflogs(expireTimeMs: number): Promise<void>;
}
```
