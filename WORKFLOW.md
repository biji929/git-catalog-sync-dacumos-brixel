# Catalog Sync — WORKFLOW.md
**Name:** Brixel Jay Dacumos (dacumos.brixel)
**Repository:** git-catalog-sync-dacumos-brixel

---

## Task 1 — Clone A: Grace Period

Checked out `feature/late-fee-policy` in Clone A and added a 1-day grace period to `calculateLateFee` (no fee if `daysLate <= 1`). Tests passed, committed, and pushed cleanly since this was the first change on the branch.

![Task 1 screenshot](screenshots/task1.png)

---

## Task 2 — Clone B: Rounding (Rejected)

In Clone B — without fetching Clone A's push — changed the fee calculation to round (`Math.round`) instead of truncate (`Math.floor`). Committed successfully, but `git push` was rejected as `non-fast-forward` because Clone A's grace-period commit had already moved the remote branch ahead of what Clone B knew about.

![Task 2 screenshot](screenshots/task2.png)

---

## Task 3 — Clone B: First Merge

Ran `git fetch origin` then `git merge origin/feature/late-fee-policy` in Clone B. Git flagged a conflict in `catalog.js` because both commits touched the same lines of `calculateLateFee`. Resolved it by combining both changes — grace period check first, then rounding:

```javascript
function calculateLateFee(daysLate, ratePerDay) {
  if (daysLate <= 1) {
    return 0;
  }
  return Math.round(daysLate * ratePerDay);
}
```

Tests passed, committed the merge, and pushed successfully.

![Task 3 screenshot](screenshots/task3.png)

---

## Task 4 — Clone C: Max Fee Cap (Rejected)

In Clone C — still at the original starter state, never fetched — added a $20 maximum fee cap using `Math.min(fee, 20)`. Committed successfully, but `git push` was rejected. The branch had moved twice on the remote (grace period, then the rounding merge) since Clone C last saw it.

![Task 4 screenshot](screenshots/task4.png)

---

## Task 5 — Clone C: Three-Way Merge

Ran `git fetch origin` then `git merge origin/feature/late-fee-policy` in Clone C. This conflict reconciled all the work so far: Clone C's $20 cap against the already-merged grace period + rounding. Resolved so all three behaviors survive together, in order:

```javascript
function calculateLateFee(daysLate, ratePerDay) {
  if (daysLate <= 1) {
    return 0;
  }
  const fee = Math.round(daysLate * ratePerDay);
  return Math.min(fee, 20);
}
```

Tests passed, committed the merge, and pushed successfully.

![Task 5 screenshot](screenshots/task5.png)

---

## Task 6 — Clone A: Minimum Fee via Rebase

Back in Clone A, without fetching since Task 1, added a $1 minimum fee (`Math.max(fee, 1)`) on top of the *original* grace-period-only version. Committed, then `git push` was rejected since Clone A hadn't seen the rounding or cap merges.

Resolved with `git fetch origin` + `git rebase origin/feature/late-fee-policy` instead of a merge. The rebase replayed the minimum-fee commit on top of the already-merged history, producing a conflict between the cap/rounding version and the floor/minimum version. Resolved so all four behaviors survive together, in order:

```javascript
function calculateLateFee(daysLate, ratePerDay) {
  if (daysLate <= 1) {
    return 0;
  }
  let fee = Math.round(daysLate * ratePerDay);
  fee = Math.min(fee, 20);
  fee = Math.max(fee, 1);
  return fee;
}
```

Tests passed, staged the resolution, ran `git rebase --continue`, and the rebase completed. `git push` succeeded **without `--force`**, since the rebase had rewritten the local commit to descend from what was already on the remote.

![Task 6 screenshot](screenshots/task6.png)

---

## Task 7 — Merge to Main, Tag, Push

In Clone A: `git checkout main`, then `git merge feature/late-fee-policy` (fast-forward, since main hadn't diverged), `git push`, then `git tag v1.0-synced` and `git push --tags`.

![Task 7 screenshot](screenshots/task7.png)

---

## Written Answers

### 1. Walk through the final `calculateLateFee` function and name which contributor's change is responsible for each part.

```javascript
function calculateLateFee(daysLate, ratePerDay) {
  if (daysLate <= 1) {
    return 0;                                   // Task 1 (Clone A) — 1-day grace period
  }
  let fee = Math.round(daysLate * ratePerDay);   // Task 3 (Clone B) — rounding instead of truncating
  fee = Math.min(fee, 20);                       // Task 5 (Clone C) — $20 maximum fee cap
  fee = Math.max(fee, 1);                        // Task 6 (Clone A) — $1 minimum fee
  return fee;
}
```

- **Grace period (`if (daysLate <= 1) return 0;`)** — added in Clone A, Task 1. Short-circuits the whole function before any fee math runs.
- **Rounding (`Math.round` instead of `Math.floor`)** — added in Clone B, Task 2, and merged into the branch in Task 3.
- **$20 cap (`Math.min(fee, 20)`)** — added in Clone C, Task 4, and merged in during the three-way merge in Task 5.
- **$1 minimum (`Math.max(fee, 1)`)** — added in Clone A, Task 6, and brought in via rebase rather than merge.

The order matters: the fee is rounded first, then capped at $20, then floored at $1 — so a very large fee gets capped before the minimum check would ever matter, and a tiny fee gets bumped up to $1 only after rounding.

### 2. Compare Task 3's two-way conflict to Task 5's three-way conflict — what got harder with a third line of work?

Task 3's conflict only had to interleave two people's changes — Clone A's grace period and Clone B's rounding touched different parts of the function's logic, so resolving it was mostly a matter of keeping both pieces and deciding where each one went. Task 5's conflict had to reconcile three sets of changes at once: the already-merged grace period + rounding on one side, and Clone C's cap on the other. The difficulty wasn't just "more lines to keep" — it was reasoning about the correct **combined order of operations** across all three changes together (does the cap apply before or after rounding? does the grace period still short-circuit everything else?), since by that point the conflict was really two people's already-resolved decisions colliding with a third person's independent one.

### 3. What's the actual difference between how you resolved Task 5 (merge) and Task 6 (rebase)?

Task 5 used `git merge`, which created a new merge commit tying Clone C's branch history together with what was already on the remote — the commit graph shows both lines of work joining at that point, preserving the order everything actually happened in. Task 6 used `git rebase`, which instead rewrote Clone A's local commit so it now sits *after* everyone else's already-pushed work, as if it had been made against the latest version of the branch from the start. There's no merge commit — the result is a linear history. That's also why the Task 6 push didn't need `--force`: since the rebased commit descends directly from what was already on the remote, git sees it as a normal fast-forward-compatible update rather than a history rewrite of shared commits.

### 4. If this were a real team of three, what one process change would have prevented all three rejected pushes?

Everyone fetching (or pulling) the branch immediately before starting new work, rather than working from a local copy that could already be stale. All three rejections happened for the same underlying reason: someone started editing based on a version of `feature/late-fee-policy` that the remote had already moved past. A simple habit of `git fetch` (or `git pull`) right before checking out the branch to start a change — combined with pushing small changes frequently instead of batching them — would have caught each divergence immediately instead of after the fact.
