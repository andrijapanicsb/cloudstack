# Local State Inventory — what exists only on this machine

Purpose: a future maintainer resuming on a **clean machine** needs to know what, if anything, is
NOT recoverable from the preserved git repositories alone. All facts **Verified** this session
via `git status`/`git branch -vv`/`git worktree list`/`git stash list` against
`D:\GIT\rbd-suite-wt`.

## 1. Working tree state

`git status --short` at audit time: **clean**. No staged, unstaged, or untracked files. No
stashes (`git stash list` empty).

## 2. This repo's branch state

- Current branch: `kvm-linstor-importvm-adoption`, HEAD `26c0e9afcb17615c62f103b1b8edd501f362be92`.
- **Fully pushed** — identical to the fork's `kvm-rbd-vmware-migration` (the actual PR head
  branch), confirmed by SHA, not name. See [PR_REVIEW_STATUS.md](PR_REVIEW_STATUS.md).
- No local-only tags reachable from the merge-base.

## 3. Other local branches/worktrees in the broader `D:\GIT\` tree (context, not part of this PR)

These are **separate, adjacent pieces of work** by the same author, not part of PR #13656's
history, listed here only so a future maintainer doesn't mistake them for uncommitted PR work or
accidentally delete something live:

| Worktree path | Branch | Tip commit | Note |
|---|---|---|---|
| `D:\GIT\rbd-suite-wt` | `kvm-linstor-importvm-adoption` | `26c0e9afcb` | **This is the PR #13656 worktree.** |
| `D:\GIT\andrijapanicsb\cloudstack-ap` | `cbt-native-rbd-support` | `c6d58ef432` | Tracks `origin/cbt-native-rbd-support` in a different personal fork remote |
| `D:\GIT\andrijapanicsb\cloudstack-vr-labels` | `clarify-vr-public-ip-label` | `7cf4a333e4` | Tracks `fork/clarify-vr-public-ip-label` |
| `D:\GIT\andrijapanicsb\cloudstack-guest-os-recognition` | `guest-os-recognition-work` | `bbe7804230` | 2 ahead / 155 behind `upstream/main` — old |
| `D:\GIT\andrijapanicsb\cloudstack-ap-vddk-public` | `import-custom-offering-prefill` | `765f166f7c` | 1 ahead / 164 behind `upstream/main` — old |
| `D:\GIT\andrijapanicsb\cloudstack-ap-vddk-rbd-4.22` | `vddk-rbd-direct-import-4.22` | `46ad18ebbd` | Tracks `upstream/4.22`, not `main` |
| `D:\GIT\andrijapanicsb\cloudstack-ap-vddk-single-pool` | `vddk-rbd-direct-import-main` | `4870053df6` | Tracks `origin/vddk-rbd-direct-import-main` |
| `D:\GIT\win-guestprep-wt` | `vmware-import-windows-guestprep` | `bc4c88cd65` | 1 ahead / 4 behind `upstream/main` — Windows guest-prep work, related but separate PR track |

Additional local-only branches in `rbd-suite-wt`'s own repo object store (no separate worktree,
listed by `git branch -vv`, not currently checked out anywhere): `backup-before-reorder`,
`linstor-fold`, `main` (local, tracks `origin/main` — a different personal fork, not upstream),
`rbd-suite-clean`, `rbd-suite-combined`, `rbd-suite-combined-prerebase`, `rbd-suite-preclean`,
`vddk-auto-select-single-pool`, `vddk-auto-select-single-pool-public`,
`improve-vm-import-guest-os-recognition`. These appear to be intermediate snapshots from prior
reorganization/rebase attempts on this same feature — **not deleted, not touched, by this
handoff**, per the explicit "do not delete branches" rule. A future maintainer who wants to
understand this PR's history should treat only `kvm-linstor-importvm-adoption` (≡ pushed
`kvm-rbd-vmware-migration`) as authoritative; the others are working debris.

## 4. Remotes configured in `D:\GIT\rbd-suite-wt`

| Remote | URL | Role |
|---|---|---|
| `upstream` | `https://github.com/apache/cloudstack.git` | The real upstream — PR base |
| `fork` | `https://github.com/andrijapanicsb/cloudstack.git` | **The PR's actual host repo** — `kvm-rbd-vmware-migration` lives here |
| `origin` | `https://github.com/andrijapanicsb/cloudstack-ap.git` | A **different** personal repo, NOT the PR host — do not confuse with `fork` |

## 5. Secrets / credentials

None found in the tracked diff (secrets scan performed earlier this session across the full
`fc832413d2c0..HEAD` diff — clean). None are stored in this repository. Required secret **names**
(not values) that any lab reproduction will need are documented in
[BUILD_AND_LAB_SETUP.md](BUILD_AND_LAB_SETUP.md) with placeholders only.

## 6. What exists ONLY on the current laptop / author's private infrastructure — NOT recoverable from this repo

- **Raw lab-regression evidence** (console screenshots, in-guest checksum output, timing logs)
  from the ~36-cell test matrix referenced in [TEST_STATUS.md](TEST_STATUS.md) §6. These lived on
  ephemeral scratch files and a private homelab environment that is rebuilt/renamed on every
  redeploy. **Not committed anywhere.** A future maintainer cannot recover this evidence; they can
  only re-run representative cells against a freshly built lab (see
  [BUILD_AND_LAB_SETUP.md](BUILD_AND_LAB_SETUP.md)).
- **The author's private lessons-learned memory** (a ~36-item list of self-inflicted test-harness
  failure causes — wrong API field casing, async-job results read synchronously, stale checksums,
  orchestration gaps, etc.) — this is genuinely useful operational knowledge for whoever re-runs
  lab testing, but it lives in a private, non-repository memory store tied to the author's tooling
  and was **not** mechanically dumped into this handoff verbatim (it is anecdotal and
  tool-specific). The generalizable parts relevant to *this feature's correctness* (not to any
  particular test harness) have been folded into [KNOWN_ISSUES.md](KNOWN_ISSUES.md) and
  [TEST_STATUS.md](TEST_STATUS.md) where they bear on the product rather than the test tooling.
- **Any in-progress reasoning/conversation state** predating this handoff document. This document
  set is the durable substitute for that state.

## 7. Ignored/generated files

Not separately audited beyond the standard `.gitignore`; `git status --short` (clean, tracked
files only) is sufficient evidence that no generated build artifacts are staged or would be
accidentally committed by a normal `git add`.
