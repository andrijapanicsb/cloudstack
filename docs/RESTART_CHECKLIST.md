# Restart Checklist — Ten Highest-Priority Steps to Resume This Work

Written for someone who has never seen this PR before, on a clean machine, with only the
preserved repositories and this documentation set. Ordered by priority, not necessarily by
sequence — some can run in parallel.

1. **Read [HANDOFF.md](HANDOFF.md) first**, then this file. It links every other document and has
   the 30-minute resume section.

2. **Verify the PR is still in the state this handoff describes.**
   ```bash
   gh pr view 13656 --repo apache/cloudstack --json state,mergeable,reviewDecision,milestone,headRefName
   gh pr checks 13656 --repo apache/cloudstack
   ```
   If `state` is no longer `OPEN`, or CI/review status has materially changed, treat
   [PR_REVIEW_STATUS.md](PR_REVIEW_STATUS.md) as historical, not current.

3. **Re-verify upstream drift.** This PR was 11 commits behind `upstream/main` at handoff time —
   it will be more behind now.
   ```bash
   git fetch upstream
   git rev-list --left-right --count upstream/main...kvm-rbd-vmware-migration
   ```
   Check `VolumeOrchestrator.java` specifically for conflicts first — it was flagged as the one
   likely collision point (see [PR_REVIEW_STATUS.md](PR_REVIEW_STATUS.md)).

4. **Run the automated test suite** (§3 of [BUILD_AND_LAB_SETUP.md](BUILD_AND_LAB_SETUP.md)) to
   confirm nothing has silently broken since this handoff. It's fast (~4 minutes), needs no
   external infrastructure, and passed 172/172 at handoff time.

5. **Close the two trivial open review nits** (import ordering, `commons.collections` vs
   `collections4` consistency — [KNOWN_ISSUES.md](KNOWN_ISSUES.md) §11). Five minutes of work,
   removes review friction for whoever looks at this PR next.

6. **Resolve the version-target contradiction** ([KNOWN_ISSUES.md](KNOWN_ISSUES.md) §1) with a
   CloudStack committer/maintainer before merge — `PendingReleaseNotes`/schema file say 4.23.0.0,
   the live milestone says 4.24.0. Don't guess; ask.

7. **The single biggest remaining risk is the zero-Marvin/integration-test-coverage gap**
   ([KNOWN_ISSUES.md](KNOWN_ISSUES.md) §10, [TEST_STATUS.md](TEST_STATUS.md) §3). If resuming
   development seriously, prioritize either (a) a real end-to-end lab run against actual
   vCenter/Ceph/Linstor (see [BUILD_AND_LAB_SETUP.md](BUILD_AND_LAB_SETUP.md)), or (b) at minimum
   an automated Marvin test skeleton, over any further feature work.

8. **Do not trust any specific pass/fail cell count from prior lab-regression rounds** without
   re-deriving it — see [TEST_STATUS.md](TEST_STATUS.md) §6 for why (36 documented self-inflicted
   false-result causes in that methodology). If you inherit access to lab tooling referenced only
   in prior conversation history, treat its claims as **Reported**, re-verify before publishing
   any number.

9. **Re-check the fix in the current HEAD commit** (`26c0e9afcb`, snapshot-cleanup retry) is still
   present and not accidentally reverted by a later rebase — grep for
   `SNAPSHOT_REMOVAL_ATTEMPTS` in `VmwareCbtMigrationManagerImpl.java`. If it's gone, the original
   defect (orphaned VMware snapshots on cancel/fail) is back.

10. **Before any real production use**, address the two flagged residual risks explicitly:
    the RBD hard-coded-zero-assumption asymmetry vs. Linstor's real check
    ([KNOWN_ISSUES.md](KNOWN_ISSUES.md) §3), and the absence of any capacity/free-space check on
    any backend ([KNOWN_ISSUES.md](KNOWN_ISSUES.md) §6). Neither has been observed failing, but
    neither has been proven safe either.

## What NOT to do without explicit new authorization

- Do not rebase, squash, or rewrite this branch's history.
- Do not force-push `kvm-rbd-vmware-migration`.
- Do not close, merge, or edit the upstream PR description without a human decision.
- Do not delete any of the adjacent local-only branches/worktrees listed in
  [LOCAL_STATE_INVENTORY.md](LOCAL_STATE_INVENTORY.md) §3 — they may still be needed context for
  related, separate work.
