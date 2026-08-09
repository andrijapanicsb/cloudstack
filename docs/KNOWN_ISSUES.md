# Known Issues, Contradictions, and Deliberate Non-Goals — PR #13656

Everything here is either a **contradiction** (two sources disagree — recorded, not resolved,
per handoff policy), a **residual risk** (a real gap in the shipped code), or a **deliberate
non-goal** (explicitly scoped out by the author, not a bug). Each entry states which it is.

---

## 1. Contradiction: target release version

- `PendingReleaseNotes` (repo root) files this PR's entry under a `4.23.0.0:` heading.
- The SQL upgrade file is named `schema-42210to42300.sql` (4.22.1.0 → **4.23.0.0**).
- The live GitHub PR milestone is **"4.24.0"** (milestone #47), confirmed via
  `gh pr view 13656 --json milestone` at the time of this audit.

**Not resolved here.** A future maintainer must ask a maintainer/committer which target is
authoritative before merge — do not silently rename the schema file or move the release-note
entry without that confirmation.

---

## 2. Contradiction: stale host-capability key name in one doc

`docs/vmware-cbt-migration.md` (lines 114, 126, 406, 412) references `host.vmware.cbt.support` —
the **pre-rename** key. Commit `17e7822265` ("Consolidate KVM host-capability probe names across
RBD features") renamed it to `host.vddk.blockcopy.support`
(`api/src/main/java/com/cloud/host/Host.java:63`), and that is what the code actually emits/checks
(`LibvirtReadyCommandWrapper.java:58`, `VmwareCbtMigrationManagerImpl.java:1103`).
`docs/vmware-cbt/README.md` correctly uses the renamed key. The same stale doc's "Related host
detail keys" list also **omits** all 5 RBD/Linstor-specific capability keys entirely — an operator
following only `vmware-cbt-migration.md` would not know they exist. See
[STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md) §5.4-5.5 for the full key list.
**Action for a future maintainer:** fix `docs/vmware-cbt-migration.md` to match
`docs/vmware-cbt/README.md`; this is a pure documentation fix, low risk, no code change needed.

---

## 3. Design asymmetry: RBD "zero-initialized" assumption is hard-coded, not queried

Linstor's `isVolumeZeroInitialized` genuinely queries the storage controller and fails closed
(only `LVM_THIN`/`ZFS`/`ZFS_THIN` return `true`). RBD's equivalent guarantee
(`--destination-is-zero` on the nbdcopy bridge path, no `-n`/`--target-is-zero` on the fallback
qemu-img-convert path) is asserted **only by a hard-coded flag and a code comment**
("The fresh raw image reads as zeros") — `isVolumeZeroInitialized` is never called for RBD at all.
See [STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md) §2 for full citations.

**Risk:** if an RBD pool or Ceph configuration ever reused non-zeroed extents for a freshly
created image (e.g., certain non-default RBD provisioning/thin-pool configurations, or a bug in
image creation), the unconditional zero-assumption would silently leave stale bytes in
unwritten regions of the target — this would only surface as data corruption, not an error.
**This audit did not observe this failing** — it is a design gap, not a reproduced bug.
**Recommended future fix:** either implement a real RBD-side check (Ceph pool feature flags for
newly-created image zero-fill are queryable) or at minimum document the assumption prominently
in `docs/vmware-cbt/README.md`.

---

## 4. Residual risk: VMware-side snapshot orphan on total retry exhaustion

Commit `26c0e9afcb` (current HEAD) adds a 4-attempt, 15-second-interval retry around VMware
snapshot removal, specifically to fix the common case (vCenter refusing removal while a source
reader is still attached right after cancel). **If all 4 attempts still fail** (e.g., a genuinely
stuck vCenter task, not just a transient reader-still-attached race), the code logs a `WARN` and
does **not** throw — cleanup/cancel/delete flows are never blocked, but the snapshot can be left
on the customer's source VM and must be removed manually. See
[ARCHITECTURE.md](ARCHITECTURE.md) §2.8 and [MIGRATION_STATE_MACHINE.md](MIGRATION_STATE_MACHINE.md)
row 4 for citations. This is an accepted trade-off (fail-open on cleanup rather than blocking the
operator), not an oversight — but a future maintainer should know the log line to grep for:
`Unable to remove VMware CBT snapshot ... after 4 attempt(s)`.

---

## 5. Residual risk: manual-reconciliation gap on import-success + concurrent-cancel race

If `importCutoverMigration` succeeds (a real KVM VM now exists) but the final CAS to
`Completed` loses to a concurrent cancel that landed in the same window, the manager logs a
warning and leaves the migration record `Cancelled` while a live imported VM exists —
**this is not automatically reconciled.** `VmwareCbtMigrationManagerImpl.java:761-763`
(**Verified**, code comment explicitly documents this as a known, accepted gap). A future
maintainer investigating an "orphaned VM with no matching non-terminal migration record" should
check for this specific log line before assuming a different root cause.

---

## 6. Deliberate non-goal: no capacity/free-space checks on any backend

Confirmed absent for File/NFS, Ceph RBD, and Linstor alike — see
[STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md) §5.1. Not a per-backend bug; uniformly
absent. A migration into a near-full pool will fail with whatever raw I/O error the underlying
tool (qemu-img/nbdcopy) surfaces, with no CloudStack-level pre-check or friendlier error message.

---

## 7. Deliberate non-goal / explicit design limitation: single-writer, root-only `importVm` adoption

`importVm importsource=shared` only ever adopts the **single ROOT disk** for both RBD and
Linstor — multi-volume (root+data) adoption requires the separate `importVolume`/
`VolumeImportUnmanageService` API, called once per additional disk. This applies identically to
both block backends (not a Linstor-specific gap) — see
[STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md) §2-3. Documented in the PR's own test-plan
notes (author's memory: "Fix A (Windows disk-online injection) has no dedicated multi-disk
Windows test cell" — an acknowledged gap in that separate test-plan document, `Reported`).

---

## 8. Deliberate non-goal: explicitly untested paths (self-disclosed by the PR author)

Per the PR body's own "Scope of this testing" section (quoted in full in
[TEST_STATUS.md](TEST_STATUS.md) §5):
- Concurrent/parallel migrations were never tested.
- The staged (non-`forceconverttopool`) conversion path for RBD/Linstor was never exercised —
  only the direct-to-pool path was tested, deliberately, to validate the most efficient path.
- UI-wizard-driven parameter collection was never tested — only direct API calls.

These are not silently discovered gaps; the author disclosed them. They remain real coverage
holes a future maintainer should close before considering the feature production-hardened.

---

## 9. Pre-existing, not introduced by this PR: late resource-limit validation

Account/domain resource limits are checked at the final import step, not at migration start — a
disk copy can run to completion before a limit rejection is reported. The migration stays
retryable (nothing is lost except time). Tracked upstream, not by this PR:
[apache/cloudstack#13780](https://github.com/apache/cloudstack/issues/13780). **Reported**
(PR body), cross-referenced against the live issue number.

---

## 10. Coverage gap: zero Marvin/integration/system test coverage

See [TEST_STATUS.md](TEST_STATUS.md) §3 for the full negative-finding evidence. All 155
new/changed test methods are unit-level with mocked collaborators. This is the single largest gap
between "the code has been reviewed/unit-tested" and "the feature has been proven to work
end-to-end via a repeatable, CI-executable test" — everything end-to-end is manual/narrative
(§ TEST_STATUS.md §5-6).

---

## 11. Two trivial, unresolved review nits (not blocking, not investigated further)

From live PR review threads (`gh api graphql`, this audit):
- Alphabetical-order nit on new `HOST_VDDK_BLOCKCOPY_*` static imports in
  `LibvirtComputingResource.java` (unresolved thread, reviewer `Damans227`).
- Mixed `org.apache.commons.collections` (legacy) vs `commons.collections4` imports across the new
  CBT wrapper files (unresolved thread, same reviewer) — cosmetic, not a functional risk.

Two other nits (casing in `ui/public/locales/en.json`, exception-message capitalization
inconsistency in `LibvirtVmwareCbtRbdProbeCommandWrapper.java`) are already marked **resolved**.
Full detail in [PR_REVIEW_STATUS.md](PR_REVIEW_STATUS.md).

---

## 12. Unknown, not chased down in this audit — flagged so it isn't silently assumed either way

- Where exactly the KVM libvirt domain is defined/started after import, and whether the imported
  VM auto-starts or is left `Stopped`. See [ARCHITECTURE.md](ARCHITECTURE.md) §7.
- Whether the "reject resize, require full resync" policy for mid-migration disk resize
  (documented in the PR's own architecture doc as a response to a known ESXi 8.0U2 CBT bug) is
  actually enforced anywhere in code — not traced in this audit.
- Whether `virt-v2v` finalization is transactional (i.e., whether a failure mid-finalization can
  leave a destination disk in a half-converted, unusable state) — not evidenced in code either way.
- Guest-tools-absent behavior for static-IP capture — not evidenced.
