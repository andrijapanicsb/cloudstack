# HANDOFF — VMware→KVM Migration (Apache CloudStack PR #13656)

**Written:** 2026-08-09. **Reason:** the primary author is pausing active development on this
contribution for an extended period. This document set is meant to let a future maintainer — with
**no** access to prior conversations, this author's memory, this laptop, or any undocumented test
environment — resume the work from a clean checkout.

**Labeling convention used throughout this document set:** every non-trivial claim is marked
**Verified** (confirmed by this audit reading actual code/git/CI state), **Reported** (a claim
from the PR author's own description or design docs, not independently re-run), **Inferred**
(a reasonable conclusion from Verified facts), or **Unknown** (deliberately not chased down).

**This document does not claim the handoff is complete merely because it and its companions
exist.** Read [TEST_STATUS.md](TEST_STATUS.md) §0 and [KNOWN_ISSUES.md](KNOWN_ISSUES.md) before
concluding this feature is production-ready — it is approved and unit-tested, but has **zero**
automated end-to-end coverage and several documented, unresolved gaps.

---

## Document map

| Document | Contents |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | How the whole thing works: orchestrator, API layer, VMware-side service, KVM agent wrappers, schema, convergence with cold import |
| [MIGRATION_STATE_MACHINE.md](MIGRATION_STATE_MACHINE.md) | Phase-by-phase table for the warm CBT path — triggers, success/failure handling, leak risk, retry-safety, citations |
| [STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md) | File/NFS vs. Ceph RBD vs. Linstor, documented separately — target selection, zero-init handling, copy mechanism, adoption path, cleanup, capability gating |
| [TEST_STATUS.md](TEST_STATUS.md) | Full test/failure ledger: unit test inventory, actual `mvn test` results run this session, CI status, PR author's self-reported functional test report, historical lab-regression context, 31-scenario negative-testing coverage table, performance evidence |
| [KNOWN_ISSUES.md](KNOWN_ISSUES.md) | Contradictions, residual risks, deliberate non-goals — 12 numbered items |
| [PR_REVIEW_STATUS.md](PR_REVIEW_STATUS.md) | Live PR/CI/review snapshot, rebase risk |
| [LOCAL_STATE_INVENTORY.md](LOCAL_STATE_INVENTORY.md) | What is committed vs. what exists only locally/ephemerally |
| [BUILD_AND_LAB_SETUP.md](BUILD_AND_LAB_SETUP.md) | Fresh-checkout build/test instructions, fully placeholder'd lab setup |
| [RESTART_CHECKLIST.md](RESTART_CHECKLIST.md) | The ten highest-priority next steps |

---

## Resume in 30 minutes

1. `git clone` the fork, `git checkout kvm-rbd-vmware-migration` (the PR's actual head branch —
   note the *local* worktree this handoff was written from calls it
   `kvm-linstor-importvm-adoption`; both names point at the same commit,
   `26c0e9afcb17615c62f103b1b8edd501f362be92`, verified by SHA not by name).
2. `mvn -q -pl server -DskipTests compile` — should exit 0.
3. Run the two test commands in [BUILD_AND_LAB_SETUP.md](BUILD_AND_LAB_SETUP.md) §3 — should show
   172/172 passing in under 5 minutes total.
4. Read [RESTART_CHECKLIST.md](RESTART_CHECKLIST.md) in full (10 items, ~5 minutes).
5. Read [KNOWN_ISSUES.md](KNOWN_ISSUES.md) in full (12 items, ~10 minutes) — this is the fastest
   way to understand what's genuinely unresolved versus what merely looks unfinished.

At that point you have the same operational picture this audit had, minus direct access to a real
vCenter/Ceph/Linstor lab (which nobody preserved — see
[LOCAL_STATE_INVENTORY.md](LOCAL_STATE_INVENTORY.md) §6).

---

## Section 11 — Final response (per the archival-handoff mandate)

**PR:** https://github.com/apache/cloudstack/pull/13656 — OPEN, APPROVED, milestone "4.24.0".

**Upstream base:** `upstream/main`, merge-base `fc832413d2c0f4012f2c512acc56866d156a17c4`.

**Feature branch:** local `kvm-linstor-importvm-adoption` ≡ pushed `kvm-rbd-vmware-migration`
(fork `andrijapanicsb/cloudstack`), HEAD `26c0e9afcb17615c62f103b1b8edd501f362be92`. **Zero
unpushed commits** — local and remote are identical, verified by `git ls-remote`.

**Archival branch:** `handoff/cloudstack-vmware-kvm-2026-08-09`, created off the same HEAD,
containing only this `docs/` documentation set as new commits on top — see the git log on that
branch for the exact commit(s). Pushed to the `fork` remote
(`https://github.com/andrijapanicsb/cloudstack.git`), **not** to `upstream`. The existing feature
branch was not touched, not rebased, not force-pushed. No PR was opened against upstream by this
handoff process; whether to open one against the author's own fork is left for explicit
confirmation (see the final message accompanying this handoff in the originating session).

**Final `git status` on the feature branch before this handoff's commits:** clean, no
staged/unstaged/untracked files, no stashes.

**Documents created:** all nine listed in the table above, plus this master document — ten total,
matching the ten required by the handoff mandate.

**Logical change groups in the diff (126 files, +21515/−203):** (1) CBT migration manager +
state/policy classes (`VmwareCbtMigrationManagerImpl`, `VmwareCbtMigrationCutoverPolicy`,
`VmwareCbtStorageTarget`); (2) API commands/responses (7 new commands); (3) schema/DB (3 new
tables + seed config); (4) VMware-side service (`VmwareCbtMigrationServiceImpl`); (5) KVM agent
wrappers (5 new CBT wrappers + a heavily extended pre-existing cold-import wrapper); (6)
`UnmanagedVMsManagerImpl` extensions (RBD/Linstor-aware adoption and cold-import paths,
convergence point for both migration modes); (7) Linstor storage-plugin extensions
(`isVolumeZeroInitialized`, `getVolumeInUseNode`, `listPhysicalDisks` implementation); (8) design
docs (4 new/updated files under `docs/`); (9) 21 test files (155 new/changed test methods); (10)
`PendingReleaseNotes` (+20 lines). Full detail with file:line citations in
[ARCHITECTURE.md](ARCHITECTURE.md) and [STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md).

**Verification commands and results:** see [BUILD_AND_LAB_SETUP.md](BUILD_AND_LAB_SETUP.md) §2-3
and [TEST_STATUS.md](TEST_STATUS.md) §2 — `mvn -q -pl server -DskipTests compile` exit 0; two
`mvn test` invocations, 172/172 passing, both actually run and observed in this session.

**Test breakdown:** 155 new/changed unit tests across 21 files, 172 of them re-executed and
passing this session, 0 disabled/skipped/ignored anywhere in the diff, **0 automated
integration/Marvin coverage** (explicit negative finding), extensive but non-repository-preserved
manual/lab testing (Reported), 31 named negative/recovery scenarios individually assessed as
tested/untested with citations. Full detail: [TEST_STATUS.md](TEST_STATUS.md).

**Unresolved review comments:** 2 open (both cosmetic nits — import ordering, collections-library
consistency), 2 resolved. 2 approvals (`wido`, `rp-`). Full detail:
[PR_REVIEW_STATUS.md](PR_REVIEW_STATUS.md).

**Rebase/merge risks:** 11 commits behind `upstream/main` at audit time (will be more by the time
this is read); one flagged likely-conflict file, `VolumeOrchestrator.java`; not re-verified with a
live rebase attempt (out of scope for a handoff — no history rewriting performed).

**Known data-consistency/cleanup risks:** VMware-side snapshot orphan on total cleanup-retry
exhaustion (residual, logged not thrown); manual-reconciliation gap on
import-success-plus-concurrent-cancel race (documented in code, not automatic); RBD's hard-coded
zero-initialization assumption vs. Linstor's genuinely-queried equivalent (asymmetry, not an
observed failure); no capacity/free-space check on any backend. All four detailed in
[KNOWN_ISSUES.md](KNOWN_ISSUES.md).

**Per-backend storage status:** File/NFS — mature, pre-existing pattern, but its cleanup path
lacks the retry/process-kill hardening the two block backends have. Ceph RBD — fully implemented,
unit-tested, zero-init assumption is a design gap not a bug. Linstor — fully implemented,
unit-tested, has the most backend-specific logic (resource attach-before-inspect, cluster-wide
in-use guard, real zero-init query) and is the more defensively engineered of the two block
backends. Full detail: [STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md).

**Every important unverified assumption:** where the KVM domain is actually defined/started after
import (Unknown); whether the mid-migration-disk-resize "reject and require full resync" policy
described in the architecture doc is actually enforced in code (Unknown); whether virt-v2v
finalization is transactional on failure (Unknown); guest-tools-absent behavior for static-IP
capture (Unknown); the true pass/fail detail of the historical ~36-cell lab regression round
(Reported only, not repository-verifiable). Full list: [KNOWN_ISSUES.md](KNOWN_ISSUES.md) §12 and
scattered "Unknown" markers throughout [ARCHITECTURE.md](ARCHITECTURE.md) and
[MIGRATION_STATE_MACHINE.md](MIGRATION_STATE_MACHINE.md).

**Secrets/private info intentionally excluded:** all vCenter/CloudStack/storage credentials, all
lab IPs/hostnames, all private homelab identifiers. [BUILD_AND_LAB_SETUP.md](BUILD_AND_LAB_SETUP.md)
uses placeholders throughout; only secret **names** (e.g. `vddk.lib.dir`, config keys) are
documented, never values.

**State that exists ONLY on the current laptop / author's private infrastructure:** raw
lab-regression evidence (screenshots, in-guest checksums, timing logs) and the author's private,
tool-specific lessons-learned notes — neither is repository-recoverable; see
[LOCAL_STATE_INVENTORY.md](LOCAL_STATE_INVENTORY.md) §6 for exactly what and why.

**Ten highest-priority restart steps:** [RESTART_CHECKLIST.md](RESTART_CHECKLIST.md), in full.

**Verdict:** a new maintainer **can** resume this work from the preserved repositories and this
documentation set alone, for every aspect covered by code, git history, CI/review state, and the
automated test suite — all of that is fully repository-backed and independently re-verifiable by
commands documented above. They **cannot** recover the original hands-on lab-validation evidence
(screenshots, checksums) from a prior regression round — that exists only as narrative in this
document set (labeled Reported throughout) and would need to be **re-produced**, not retrieved, by
rebuilding a test lab per [BUILD_AND_LAB_SETUP.md](BUILD_AND_LAB_SETUP.md). The feature's code,
architecture, and unit-test coverage are accurately and completely preserved; its end-to-end,
real-infrastructure validation is not, and was never committed anywhere it could be.
