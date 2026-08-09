# Migration State Machine — Warm CBT Path

Companion to [ARCHITECTURE.md](ARCHITECTURE.md). All facts here are **Verified** by direct code
read against `D:\GIT\rbd-suite-wt`, branch `kvm-linstor-importvm-adoption`, HEAD
`26c0e9afcb17615c62f103b1b8edd501f362be92`, unless marked otherwise. This maps the requested
16-phase model onto what the code actually implements — phases that don't exist as distinct code
constructs are marked so explicitly, not silently merged.

## Stored-state diagram

```
Created --(persist, same call)--> InitialSync --(runInitialFullSync ok)--> Replicating
                                        |                                       |
                                        | (fail)                        syncVmwareCbtMigration
                                        v                                (repeated, CutoverPolicy
                                      Failed                              decides continue/ready)
                                                                                |
                                                          +---------------------+
                                                          v
                                                  ReadyForCutover --(cutover: CAS)--> CuttingOver
                                                          ^                                |
                                                          | (sync still legal here)         | final delta + finalize
                                                          |                                 v
                                                   Replicating <---------------------  ReadyForImport --(import ok: CAS)--> Completed
                                                                                              |                                  ^
                                                                                              | import fails, revert            | retry cutover while ReadyForImport
                                                                                              +----------------------------------+

Any non-terminal state --(cancel)--> Cancelled       Failed / Cancelled / Completed --(delete)--> [row removed]
```

## Phase table

| # | Named phase (request) | Maps to (code) | Trigger | On success | On failure | Leak risk | Retry-safe? | Citation |
|---|---|---|---|---|---|---|---|---|
| 1 | Preflight validation | `checkVmwareCbtMigrationPrerequisites` | API call, advisory | Findings + `ready` flag, **no persistence** | Findings list failures, no exception | None (read-only) | Fully idempotent | `VmwareCbtMigrationManagerImpl.java:257-314` |
| 2 | Source VM/volume discovery | Inside `startVmwareCbtMigration`, `listSourceDisks` | Continuation of Start | Disk list + device keys captured | `ServerApiException` before persist (no record created) | None | N/A — creates a new migration on retry | `VmwareCbtMigrationManagerImpl.java:491`; `VmwareCbtMigrationServiceImpl.java:105-138` |
| 3 | CBT eligibility/enablement | `validateSourceVmPreflightForStart` + `ensureSourceVmChangeTrackingEnabledForStart` | Continuation of Start | CBT confirmed enabled | Throws if unsupported | None | Same as #2 | `VmwareCbtMigrationManagerImpl.java:316-344, 490` |
| 4 | VMware snapshot creation/lifecycle | `createBaselineSnapshot` / `createDeltaSnapshot` / `removeSnapshotByMorIfPossible` | Per full-sync and per delta cycle | Snapshot created, used, then removed in `finally` (retried up to 4×, 15s apart) | Removal exhausted → **WARN logged, not thrown**; snapshot may be left on source VM | **Yes — on total retry exhaustion, snapshot orphan on source VM** (residual risk after `26c0e9afcb`) | Removal is retried automatically within one attempt; a leaked snapshot needs manual vCenter cleanup | `VmwareCbtMigrationManagerImpl.java:2073-2078, 2141-2148, 2262-2299` |
| 5 | Initial/base disk transfer | `runInitialFullSync` → `VmwareCbtPrepareCommand` → `LibvirtVmwareCbtPrepareCommandWrapper` | Continuation of Start (synchronous, same call) | `state=Replicating`, disks get `targetPath`/`changeId` | `markInitialSyncDisksFailed` + `markMigrationFailed`, `state=Failed` | Partial target file/RBD image/block volume may be left on the destination pool — **not cleaned up automatically**, only via a subsequent Cancel/Delete | No automatic retry of "initial sync" specifically — must Cancel+Delete and start a fresh migration | `VmwareCbtMigrationManagerImpl.java:2014-2071` |
| 6 | Changed-block discovery | `queryChangedDiskAreas` (paginated vSphere reflection call) | Start of each delta cycle | Extent list returned, capped/paginated to cover full disk | Exception propagates to cycle/migration failure | None | N/A (read-only query) | `VmwareCbtMigrationServiceImpl.java:179-201, 412-459` |
| 7 | Incremental transfer rounds | `syncVmwareCbtMigration` → `VmwareCbtSyncCommand` → `LibvirtVmwareCbtSyncCommandWrapper` | Repeated caller-driven calls, state ∈ {Replicating, ReadyForCutover} | Cycle row `Completed`, `changeId` advanced, byte/dirty-rate counters updated | Cycle + migration marked `Failed` (unless a concurrent cancel already won the CAS) | Delta snapshot always removed in `finally` — same exhaustion risk as #4 | Migration `Failed` after a delta error is **not** auto-retried by CloudStack — caller must intervene | `VmwareCbtMigrationManagerImpl.java:555-649` |
| 8 | Conditions for another delta round | `VmwareCbtMigrationCutoverPolicy.decide` | End of every successful cycle | `CONTINUE` / `READY_FOR_CUTOVER` / `READY_FOR_CUTOVER_MAX_CYCLES` | N/A — pure decision function | N/A | N/A | `VmwareCbtMigrationCutoverPolicy.java:51-73` |
| 9 | Source quiescing/shutdown | `requireSourceVmPoweredOff`, checked at the top of a fresh cutover | Manual/external — **no automated forced shutdown exists** (documented non-goal, PR design doc) | Cutover proceeds | `ServerApiException`, cutover aborts before any state change, instructs operator to shut down gracefully and retry | None | Fully retryable — call cutover again once source is off | `VmwareCbtMigrationManagerImpl.java:940-949`; `docs/vmware-cbt/README.md:1683-1687` (**Reported** design-doc statement re: no forced shutdown) |
| 10 | Final delta transfer | `runFinalDeltaSync` (same mechanics as #7, `isFinalCycle=true`) inside `cutoverVmwareCbtMigration` | After CAS to `CuttingOver` | Final cycle completes, snapshot removed | `markMigrationFailed`, `state=Failed` | Same snapshot-removal risk as #4 | Not retryable from `Failed`; retryable if still `ReadyForCutover`/`ReadyForImport` | `VmwareCbtMigrationManagerImpl.java:665-681, 865-938` |
| 11 | Destination volume finalization | `VmwareCbtCutoverCommand` → `LibvirtVmwareCbtCutoverCommandWrapper` (virt-v2v in-place or fallback) | Continuation of cutover, after final delta succeeds | Disk results applied, CAS to `ReadyForImport` | `markMigrationFailed`, `state=Failed`; or if CAS to `ReadyForImport` loses to a concurrent cancel, import is skipped (logged) | Partially-converted target disk state on failure — **Unknown whether virt-v2v itself is transactional; not evidenced in code** | Cutover is re-callable while state is `ReadyForCutover`/`ReadyForImport` | `VmwareCbtMigrationManagerImpl.java:693-720` |
| 12 | KVM domain/VM definition | Inside `importVirtualMachineInternal` (shared with cold import) | `importCutoverMigration` → `unmanagedVMsManager.importConvertedVmwareCbtInstanceToKvm` | VM record created | Import throws → `state` reverts to `ReadyForImport` with error recorded | If import **succeeds** but the terminal CAS loses to a concurrent cancel: **live imported VM exists, migration record stuck `Cancelled` — documented manual-reconciliation gap, not automatic** | **Explicitly retryable** — another `cutoverVmwareCbtMigration` call while `ReadyForImport` re-enters import directly | `VmwareCbtMigrationManagerImpl.java:722-775` |
| 13 | Destination VM startup | Not evidenced as a distinct migration-manager phase | — | — | — | — | — | **Unknown — not evidenced in code, would need runtime testing.** Whether the imported VM auto-starts or is left `Stopped` pending an operator `startVirtualMachine` call was not conclusively traced within this PR's touched files. |
| 14 | Post-start validation | Not evidenced as a distinct phase | — | — | — | — | — | **Unknown — not evidenced in code.** No CloudStack-side post-start guest validation step (e.g. guest-agent ping, boot-proof capture) was found in the manager; any such validation described in the PR body's Functional Test Report was performed **manually by the author outside CloudStack** (console screenshots, in-guest checksums) — see [TEST_STATUS.md](TEST_STATUS.md). |
| 15 | Source-side cleanup | `cancelVmwareCbtMigration` / `deleteVmwareCbtMigration` → `removeLingeringSourceSnapshots` + target cleanup command | Cancel (any non-terminal state) or Delete (state ∈ {Failed, Cancelled, Completed}) | Target disks removed (if applicable), lingering source snapshots swept, credentials cleared, rows removed (Delete only) | Cancel: CAS loss → cleanup skipped entirely, no double cleanup, safe to call twice. Delete: throws `PARAM_ERROR` if state not eligible; nothing touched | Snapshot-orphan risk same as #4; target-disk cleanup is **skipped entirely** (only logged) if another non-terminal migration shares the same convert host | Cancel: safe to call twice (idempotent no-op on second call). Delete: not literally idempotent (second call 404s via `getMigration`) but safe — no partial re-cleanup against a removed record | `VmwareCbtMigrationManagerImpl.java:951-1015` |
| 16 | Failure rollback/recovery at every phase | See "Retry-safe?" column above, per phase | — | — | — | — | — | Summarized per-row above; there is **no single unified rollback routine** — each phase has its own failure handling as cited. |

## Additional points required by the original request, not captured in the table above

- **CBT generation-ID/state validation:** `queryChangedDiskAreas` is called with the disk's
  stored `changeId` each cycle; the manager does not independently validate that the `changeId`
  is still consistent with the source VM's actual CBT generation before use — it relies on
  vSphere's own error response if the ID is stale/invalid. **Inferred** from the absence of any
  separate "changeId validation" step in `syncVmwareCbtMigration` (`VmwareCbtMigrationManagerImpl.java:555-649`).
- **Behavior when CBT is unavailable/invalid/reset/inconsistent:** `ensureChangeTrackingEnabled`
  throws if `capability.isChangeTrackingSupported()` is false (`VmwareCbtMigrationServiceImpl.java:140-153`).
  There is no code path that detects a *mid-migration* CBT reset (e.g. an administrator disabling
  CBT on the source VM out-of-band) — such a reset would surface as a vSphere API error on the
  next `queryChangedDiskAreas` call, which propagates as a normal cycle failure (row #7).
  **Inferred**, not exercised by any test found (see [TEST_STATUS.md](TEST_STATUS.md)).
- **Behavior after snapshot consolidation:** `getPreflightInfo` reports
  `runtimeInfo.isConsolidationNeeded()` and start-time validation rejects a source VM with
  pending consolidation (`VmwareCbtMigrationManagerImpl.java:331-335`) — this is a **pre-check**,
  not a runtime reaction to consolidation happening mid-migration. **Verified** for the pre-check;
  mid-migration behavior is **Unknown**.
- **Whether full reseed is ever required:** the PR's own architecture doc states that ESXi
  8.0U2 is documented (by Broadwell/Broadcom) to return incorrect sectors after hot-extending a
  VMDK, and "the safe policy is to reject disk resize during migration and require full resync
  after resize" (`docs/vmware-cbt/architecture.md:537-553`, **Reported**, design-doc statement —
  not independently verified against a live resize-during-migration test in this audit).
- **Cross-volume data-consistency guarantee:** each disk is synced independently per cycle; there
  is no evidence of a multi-disk consistency barrier (e.g., a single crash-consistent snapshot
  covering all disks simultaneously is in fact what `createSnapshot`/`createDeltaSnapshot`
  produce — VMware VM-level snapshots span all disks atomically at the moment of creation, so
  **per-cycle** consistency across disks is achieved by construction). Cross-cycle consistency
  (i.e., whether disk A's cycle-3 data and disk B's cycle-3 data represent the exact same instant)
  follows from all disks sharing one VM-level snapshot per cycle. **Verified** at the mechanism
  level (`createDeltaSnapshot` creates one VM snapshot covering all disks,
  `VmwareCbtMigrationServiceImpl.java:155-177`); not independently tested for actual bit-level
  consistency in this audit.
- **Downtime calculation/bounding:** no code-level downtime budget/timeout was found that bounds
  the cutover window. The PR body reports one measured example (Windows warm CBT: final delta
  moved ~15.8 MiB against a 45 GiB disk set, 0.034%) — **Reported**, a single sample, not a
  guarantee. See [TEST_STATUS.md](TEST_STATUS.md) §Performance.
- **Which operations can leak snapshots/temp volumes/partial destination VMs:** consolidated in
  the "Leak risk" column above. The two live residual risks are: (a) VMware-side snapshot orphan
  on total retry exhaustion (rows #4, #7, #10, #15), and (b) partial destination-pool artifacts
  from an initial full sync that failed before any Cancel/Delete was issued (row #5) — both are
  tracked in [KNOWN_ISSUES.md](KNOWN_ISSUES.md).
