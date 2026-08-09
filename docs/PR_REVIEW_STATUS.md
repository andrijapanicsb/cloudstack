# PR Review Status — apache/cloudstack#13656

All facts below **Verified** via `gh pr view`/`gh pr checks`/`gh api graphql` against the live
GitHub PR, run during this audit on **2026-08-09**. CI check results are a live snapshot as of
that date/time — re-run the commands below to get current state; do not treat this file as
permanently current.

## Identity

| Field | Value |
|---|---|
| URL | https://github.com/apache/cloudstack/pull/13656 |
| Number | 13656 |
| Title | "VMware to KVM: cold (VDDK) and warm (CBT) migration into Ceph/RBD and Linstor, plus importVm adoption of existing RBD root volumes and Linstor root/data volumes" |
| Base branch | `main` (upstream) |
| Head branch | `kvm-rbd-vmware-migration` (in the author's fork, `andrijapanicsb/cloudstack`) |
| State | OPEN |
| Mergeable | MERGEABLE (per GitHub's own check, at observation time) |
| Review decision | APPROVED |
| Milestone | "4.24.0" (#47) — **contradicts** `PendingReleaseNotes`'/schema-file's "4.23.0.0" framing, see [KNOWN_ISSUES.md](KNOWN_ISSUES.md) §1 |
| Created | 2026-07-20T22:37:56Z |
| Last updated | 2026-08-09T04:30:43Z |
| Local worktree | `D:\GIT\rbd-suite-wt`, local branch name `kvm-linstor-importvm-adoption` (**note: local branch name differs from the pushed/PR head branch name `kvm-rbd-vmware-migration`** — verified by SHA ancestry, not by name-matching: `git ls-remote fork refs/heads/kvm-rbd-vmware-migration` returns the exact same SHA as local HEAD) |
| Local HEAD | `26c0e9afcb17615c62f103b1b8edd501f362be92` |
| **Sync with fork** | **Zero unpushed commits** — local HEAD is byte-identical to the fork's `kvm-rbd-vmware-migration` tip, confirmed via `git ls-remote`. (A prior draft of this handoff, before this check was re-run with a live `git ls-remote`, believed there were 2 unpushed commits — that was stale/wrong and is corrected here.) |

## Diff scope vs. upstream `main`

| Metric | Value |
|---|---|
| Merge-base | `fc832413d2c0f4012f2c512acc56866d156a17c4` |
| Ahead of `upstream/main` | 41 commits |
| Behind `upstream/main` | 11 commits |
| Files changed | 126 |
| Lines | +21515 / −203 |
| Test-file portion of that diff | 21 files, +3168 / −5 (see [TEST_STATUS.md](TEST_STATUS.md)) |

First commit in range: `3552a28192` "VMware CBT native RBD warm migration to KVM".
Last (current HEAD): `26c0e9afcb` "KVM VMware CBT: do not orphan the baseline snapshot when a
migration is cancelled or fails" (the one product fix made in this session, before the handoff
request — see [ARCHITECTURE.md](ARCHITECTURE.md) §2.8).

**Rebase risk:** being 11 commits behind `upstream/main` carries some conflict risk. Exactly one
file was identified as a likely conflict point: `VolumeOrchestrator.java`
(`engine/orchestration/src/main/java/org/apache/cloudstack/storage/allocator/...` — flagged
because this PR modifies it too, `getSupportedImageFormatForCluster`, and upstream `main` has
moved 11 commits since the merge-base). **Not re-verified with a live rebase attempt in this
audit** (per the handoff's explicit rule: do not rebase/rewrite history while preparing a
handoff) — flagged as the one file to check first if/when a real rebase is eventually performed.

## CI checks — live snapshot, 2026-08-09 (a fresh run triggered by the last push)

| Result | Count |
|---|---|
| pass | 7 |
| pending | 19 |
| fail | 1 |

Passed: `CodeQL`, `CodeQL (actions)`, `build` (3 matrix legs), `codecov/patch`, `Run pre-commit`.
Failed: `codecov/project` (cause not yet investigated for *this* run — it was still fresh at
observation time). Everything else (`test (component/...)`, `test (smoke/...)`) was still
queued/running.

**Historical CI evidence from earlier in this session** (two red marks from a *previous* run,
both root-caused as non-product issues — preserved because they will likely recur on future runs
and should not be mistaken for new regressions):
1. "Sonar JaCoCo Coverage" — fails at a PR-comment-posting step due to the standard GitHub
   Actions fork-PR `GITHUB_TOKEN` read-only restriction, not a coverage regression.
2. A `component/test_project_usage ...` shard — failed because the CI's CloudStack simulator
   management server never came up (`db.properties` not found, port 8096 connection refused for
   15+ min), so the named tests never ran at all — infrastructure failure, not a test-assertion
   failure.

Re-run `gh pr checks 13656 --repo apache/cloudstack` to get current results before acting on this
section.

## Reviews

| Reviewer | State | Date |
|---|---|---|
| `Damans227` | COMMENTED | 2026-07-21 |
| `andrijapanicsb` (author, self-comment) | COMMENTED | 2026-07-21 |
| `rp-` | COMMENTED | 2026-07-23 |
| `wido` | **APPROVED** | 2026-08-03 |
| `rp-` | **APPROVED** | 2026-08-04 |

Total issue comments on the PR: **39**.

## Review threads

| Resolved? | File | Reviewer | Nit |
|---|---|---|---|
| No | `LibvirtComputingResource.java:33` | `Damans227` | New `HOST_VDDK_BLOCKCOPY_*` static imports break existing alphabetical import order |
| No | `VmwareCbtSyncPlan.java:26` | `Damans227` | Mixes legacy `commons.collections` with `commons.collections4` used elsewhere in the new files |
| Yes | `ui/public/locales/en.json:3560` | `Damans227` | Casing regression ("KVM Host"→"KVM host", "Storage Pool"→"Storage pool") |
| Yes | `LibvirtVmwareCbtRbdProbeCommandWrapper.java:138` | `Damans227` | Exception-message capitalization inconsistency within the file |

**Two unresolved threads, both cosmetic nits (import ordering, import-library consistency) — not
functional concerns, not blocking merge from a correctness standpoint.**

## What this means for a resuming maintainer

- The PR is **approved and mergeable** per GitHub's own signals as of this snapshot.
- The two open review threads are trivial and can be fixed in five minutes each whenever picked
  back up.
- The real work before merge is closing the gaps in [KNOWN_ISSUES.md](KNOWN_ISSUES.md) and
  [TEST_STATUS.md](TEST_STATUS.md) — particularly the zero-Marvin-coverage gap and the
  version-target contradiction — not addressing review feedback, which is essentially done.
- Before merging, re-check whether `upstream/main` has moved further (it was 11 commits ahead at
  audit time) and re-run a rebase-conflict check against `VolumeOrchestrator.java` specifically.
