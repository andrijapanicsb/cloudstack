# Architecture — VMware→KVM Migration (PR #13656)

Archival technical reference for a future maintainer with no access to prior conversations,
memory, or the original author's laptop. Every claim is labeled:

- **Verified** — confirmed by reading the actual code in this repository at the cited
  `file:line`, on `D:\GIT\rbd-suite-wt` branch `kvm-linstor-importvm-adoption`,
  HEAD `26c0e9afcb17615c62f103b1b8edd501f362be92`.
- **Reported** — a claim made by the PR author (in the PR description, PR body's embedded
  "Functional Test Report", or design docs) that was not independently re-verified in this
  audit.
- **Inferred** — a reasonable conclusion drawn from Verified facts, not itself directly observed.
- **Unknown** — deliberately not chased down; do not assume either way.

See also: [MIGRATION_STATE_MACHINE.md](MIGRATION_STATE_MACHINE.md) (phase-by-phase detail),
[STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md) (per-backend divergence).

---

## 1. Two distinct feature families

This PR ships two separable capabilities that must not be conflated (**Verified**, by reading
both code paths):

1. **VM conversion/migration** — copies bytes out of a VMware datastore into a *new*
   CloudStack-managed volume. Two sub-modes:
   - **Cold / direct-VDDK import** (`ImportVmCmd`, pre-existing command, code extended by this
     PR) — a point-in-time conversion via `virt-v2v`/nbdkit-VDDK.
   - **Warm CBT migration** (new: `startVmwareCbtMigration` / `syncVmwareCbtMigration` /
     `cutoverVmwareCbtMigration` / `cancelVmwareCbtMigration` / `deleteVmwareCbtMigration` /
     `listVmwareCbtMigrations` / `checkVmwareCbtMigrationPrerequisites`) — an initial full copy
     followed by repeated incremental Changed-Block-Tracking (CBT) delta rounds, then a final
     cutover.
2. **`importVm importsource=shared` adoption** — takes over an *already-provisioned* RBD or
   Linstor volume with no byte copying at all
   (`server/src/main/java/org/apache/cloudstack/vm/UnmanagedVMsManagerImpl.java:3439-3577`,
   **Verified**).

Both feature families converge on the same VM-materialization code,
`importVirtualMachineInternal` — see §5.

---

## 2. Core orchestrator: `VmwareCbtMigrationManagerImpl`

File: `server/src/main/java/org/apache/cloudstack/vm/VmwareCbtMigrationManagerImpl.java`
(2976 lines). **Verified.**

### 2.1 Migration state enum

`api/src/main/java/org/apache/cloudstack/vm/VmwareCbtMigration.java:23-46`:

```
Created → InitialSync → Replicating ⇄ (sync loop) → ReadyForCutover → CuttingOver → ReadyForImport → Completed
                                                                                            ↘ Failed
                                                                                            ↘ Cancelled
```

`isTerminal()` returns true only for `Completed`, `Failed`, `Cancelled`
(`VmwareCbtMigration.java:34-36`, **Verified**). `Created` is set in the VO constructor but the
code advances to `InitialSync` before ever persisting the row
(`VmwareCbtMigrationManagerImpl.java:532`), so `Created` is not observed as a stored state in
practice (**Verified**).

Two finer-grained sub-state enums exist:
- `VmwareCbtMigrationDisk.State`: `Created, Prepared, Syncing, Ready, Failed`
- `VmwareCbtMigrationCycle.State`: `Created, QueryingChangedAreas, CopyingChangedBlocks, Completed, Failed`

Full phase-by-phase table (triggers, success/failure paths, leak risk, retry-safety,
citations): see [MIGRATION_STATE_MACHINE.md](MIGRATION_STATE_MACHINE.md).

### 2.2 Preflight (`checkVmwareCbtMigrationPrerequisites`)

`VmwareCbtMigrationManagerImpl.java:257-314`. Read-only, advisory, creates no records. Checks
destination zone/cluster/pool + `VmwareCbtStorageTarget` support, CBT-capable convert host,
source VM resolvability, and (if resolvable) a full source-VM preflight via
`VmwareCbtMigrationService.getPreflightInfo` (VMware-side, §4). Returns `ready = !findings.hasFailures()`.
**Verified.**

### 2.3 Start (`startVmwareCbtMigration`)

`VmwareCbtMigrationManagerImpl.java:458-538`. In order: resolves account/zone/cluster/pool,
selects CBT host, validates source VM name, resolves vCenter credentials, runs
**start-time enforcement preflight** (`validateSourceVmPreflightForStart`, throws on: CBT
unsupported, pending snapshot consolidation, zero discovered disks — lines 316-344), validates
service offering ≥ source CPU/memory, validates Windows-conversion support, **enables CBT on the
source VM if not already enabled**, discovers source disks, validates disk-offering/NIC mappings
against source topology (deliberately before any data copy, comment at 2459-2464). Persists the
`VmwareCbtMigrationVO` + per-disk rows, sets `state=InitialSync`, then — **synchronously in the
same call** — invokes `runInitialFullSync(...)`. **Verified.**

`StartVmwareCbtMigrationCmd extends BaseAsyncCmd` (`api/.../StartVmwareCbtMigrationCmd.java:70`)
with no `getSyncObjId()` override. Consequence (**Inferred** from standard CloudStack async-job
dispatch + **Verified** `execute()` call site at `StartVmwareCbtMigrationCmd.java:344-350`): the
entire preflight + full-copy chain runs inside the async-job executor thread; a failure is
**not visible in the synchronous HTTP response**, only in the polled job result. Start requests
are not serialized against any existing migration (none exists yet at call time).

### 2.4 Delta sync (`syncVmwareCbtMigration`)

`VmwareCbtMigrationManagerImpl.java:555-649`. Requires state ∈ `{Replicating, ReadyForCutover}`
(a sync call after cutover-readiness is legal — allows extra manual delta rounds). Creates a
delta VMware snapshot, queries `QueryChangedDiskAreas` for all disks, dispatches
`VmwareCbtSyncCommand` to the KVM host, applies per-disk results, advances CBT `changeId`s,
computes dirty-rate, and calls `VmwareCbtMigrationCutoverPolicy.decide(...)` (§2.5) to choose
`Replicating` (continue) vs `ReadyForCutover`. The delta snapshot is **always** removed in the
`finally` block regardless of outcome. **Verified.**

`SyncVmwareCbtMigrationCmd extends BaseAsyncCmd`, serialized per-migration:
`getSyncObjType()="migration"`, `getSyncObjId()=id` — sync and cutover calls against the same
migration cannot run concurrently at the async-job-queue level. **Verified**
(`SyncVmwareCbtMigrationCmd.java:87-95`).

### 2.5 Cutover-readiness decision — `VmwareCbtMigrationCutoverPolicy`

File: `server/src/main/java/org/apache/cloudstack/vm/VmwareCbtMigrationCutoverPolicy.java`.
`decide(completedCycles, quietCycles, lastChangedBytes, lastCycleDurationSeconds)` (51-66):

1. `completedCycles >= maxCycles` → force `READY_FOR_CUTOVER_MAX_CYCLES`.
2. `completedCycles < minCycles` → `CONTINUE` (floor enforced regardless of dirtiness).
3. `lastChangedBytes == 0` → immediate `READY_FOR_CUTOVER` (a fully quiet cycle short-circuits
   the quiet-streak counter).
4. Otherwise increments the quiet-streak when `isQuietCycle`, returns `READY_FOR_CUTOVER` once
   the streak reaches `quietCyclesRequired`.

`isQuietCycle` = `changedBytes <= quietDirtyBytesThreshold AND dirtyRate <=
quietDirtyRateBytesPerSecondThreshold`. **Verified.**

Config keys (`VmwareCbtMigrationManagerImpl.java:146-189`, **Verified**):

| Key | Default |
|---|---|
| `vmware.cbt.migration.min.cycles` | 1 |
| `vmware.cbt.migration.max.cycles` | 5 |
| `vmware.cbt.migration.quiet.cycles` | 2 |
| `vmware.cbt.migration.quiet.bytes` | 1 GiB |
| `vmware.cbt.migration.quiet.dirty.rate` | 16 MiB/s |
| `vmware.cbt.migration.agent.command.timeout` | 1 day (seeded via `schema-42210to42300.sql:718-732`) |
| `vmware.cbt.allow.non.inplace.finalization` | (seeded same file) |

### 2.6 Cutover (`cutoverVmwareCbtMigration`)

`VmwareCbtMigrationManagerImpl.java:651-775`. Requires state `ReadyForCutover` **or**
`ReadyForImport` (the latter re-enters `importCutoverMigration` directly — a retry path for a
migration whose conversion succeeded but whose CloudStack import failed). Fresh cutover path:
re-validates target disks, resolves CBT host/storage-target/finalization support, **requires the
source VM to already be powered off** (`requireSourceVmPoweredOff`, throws with an instruction to
shut it down gracefully and retry — no automatic forced power-off exists), does a
compare-and-swap (CAS) transition to `CuttingOver` (§2.7), runs one final delta sync
(`isFinalCycle=true`), dispatches `VmwareCbtCutoverCommand` for destination finalization
(virt-v2v in-place or fallback), then a second CAS to `ReadyForImport`, then
`importCutoverMigration`. **Verified.**

`CutoverVmwareCbtMigrationCmd` is also `BaseAsyncCmd`, serialized per-migration the same way as
Sync. **Verified.**

#### 2.6.1 `importCutoverMigration` — `VmwareCbtMigrationManagerImpl.java:722-775`

Re-checks non-terminal state immediately before the expensive import (comment: "a cancel may
have landed while the preceding conversion was running"). Calls
`unmanagedVMsManager.importConvertedVmwareCbtInstanceToKvm(...)` — the convergence point with
cold import (§5). On success: `vmId` set, `state=Completed` via CAS; **if that final CAS loses to
a concurrent cancel, the manager logs a warning that a live imported VM now exists but the
migration record stays `Cancelled`, and this must be reconciled manually** — a documented,
non-automatic gap, not a silent bug. On import failure: `state` reverts to `ReadyForImport` with
the error recorded, so a retry via another `cutoverVmwareCbtMigration` call is explicitly
supported. **Verified.**

### 2.7 Concurrency control — the only lock primitive

`VmwareCbtMigrationDaoImpl.updateIfNotTerminal`
(`engine/schema/src/main/java/.../dao/VmwareCbtMigrationDaoImpl.java:64-70`) is a plain
`UPDATE ... WHERE id=? AND state NOT IN (Completed,Failed,Cancelled)`. **No `SELECT ... FOR
UPDATE`, no optimistic-lock version column** — race detection relies entirely on the affected-row
count of one conditional UPDATE. Used at every terminal-state-sensitive write: progress
persistence after every sync, mark-failed, both CAS points in cutover, the import-completion CAS,
and the cancel transition. The async-job-framework's per-migration sync object (§2.4, §2.6)
serializes *API-triggered* operations against the same migration, but does **not** protect
against, e.g., an in-flight background sync racing a cancel issued from elsewhere — that race is
exactly what `updateIfNotTerminal` is designed to lose gracefully against (loser gets an
exception or a skipped step, never silent double-processing). **Verified.**

### 2.8 Snapshot lifecycle and the `26c0e9afcb` fix

- **Baseline snapshot** (`createBaselineSnapshot`, lines 2073-2078): created inside
  `runInitialFullSync`, named `cloudstack-cbt-<uuid>-baseline`, **not quiesced**
  (`quiesce=false`), removed in `runInitialFullSync`'s `finally` block regardless of outcome.
- **Delta snapshot** (`createDeltaSnapshot`, 2141-2148): named
  `cloudstack-cbt-<uuid>-<cycleNumber>`, created per cycle inside `syncVmwareCbtMigration` and
  `runFinalDeltaSync`, always removed in `finally`.
- **The fix in the current HEAD commit** (`26c0e9afcb17615c62f103b1b8edd501f362be92`, message
  "KVM VMware CBT: do not orphan the baseline snapshot when a migration is cancelled or fails"):
  adds `removeSnapshotByMorIfPossible` (lines 2262-2299), a retry wrapper around vCenter's
  `removeSnapshot` (4 attempts, 15s apart — constants at lines 124-125 —
  `SNAPSHOT_REMOVAL_ATTEMPTS=4`, `SNAPSHOT_REMOVAL_RETRY_INTERVAL_MS=15000L`), because vCenter
  answers *"operation not allowed in current state"* while a source-side reader
  (nbdkit/VDDK) is still attached right after a cancel. On exhaustion it **logs a WARN and does
  not throw** — cleanup/cancel/delete flows are never blocked by a stuck snapshot, but a snapshot
  can still be left on the source VM if all 4 retries fail (residual risk, tracked in
  [KNOWN_ISSUES.md](KNOWN_ISSUES.md)). `removeLingeringSourceSnapshots` (2306-2325) sweeps every
  distinct `snapshotMor` still recorded on any disk row for a migration, called from both
  `cancelVmwareCbtMigration` (line 968, **after** `sendCleanupCommandIfPossible` returns — that
  command is what stops nbdkit/VDDK readers holding the source disk open) and
  `deleteVmwareCbtMigration` (before `clearStoredSourceCredentials` and before disk-row removal,
  because afterward the snapshot can no longer be authenticated for or looked up by MOR).
  **Verified — this is the one product fix made in this session, before the handoff request was
  issued; it is a normal commit on the existing feature branch, not a handoff artifact.**

---

## 3. API layer

`api/src/main/java/org/apache/cloudstack/api/command/admin/vm/` and
`api/src/main/java/org/apache/cloudstack/api/response/`. **Verified.**

| Command | API name | Sync/Async | Sync object |
|---|---|---|---|
| `CheckVmwareCbtMigrationPrerequisitesCmd` | `checkVmwareCbtMigrationPrerequisites` | Sync | n/a |
| `StartVmwareCbtMigrationCmd` | `startVmwareCbtMigration` | **Async** | none |
| `SyncVmwareCbtMigrationCmd` | `syncVmwareCbtMigration` | **Async** | `migration`/id |
| `CutoverVmwareCbtMigrationCmd` | `cutoverVmwareCbtMigration` | **Async** | `migration`/id |
| `CancelVmwareCbtMigrationCmd` | `cancelVmwareCbtMigration` | Sync | n/a |
| `DeleteVmwareCbtMigrationCmd` | `deleteVmwareCbtMigration` | Sync | n/a |
| `ListVmwareCbtMigrationsCmd` | `listVmwareCbtMigrations` | Sync (list) | n/a |

Each async `execute()` blocks the async-job executor thread for the full duration of the
operation it performs — a full disk copy for Start, one delta cycle for Sync, final delta +
finalization + import for Cutover. **Verified** (confirmed directly for Start at
`StartVmwareCbtMigrationCmd.java:344-350`; the same call pattern applies to Sync/Cutover).

Response objects: `VmwareCbtMigrationResponse` (+ nested `Cycle`/`Disk` responses),
`VmwareCbtMigrationPreflightResponse` (+ nested `Disk`/`Finding` responses).

`ImportVmCmd` and `ListImportVMTasksCmd` in the same directory are **pre-existing** cold-import
commands, extended by this PR's changes to `UnmanagedVMsManagerImpl` rather than newly added.

---

## 4. VMware-side service — `VmwareCbtMigrationServiceImpl`

File: `plugins/hypervisors/vmware/src/main/java/com/cloud/hypervisor/vmware/manager/VmwareCbtMigrationServiceImpl.java`
(587 lines). All vSphere Web Services API calls go through
`VmwareContextFactory.getContext(vcenter, username, password)`. **Verified.**

- `getPreflightInfo` — CBT support/enabled, consolidation-needed, snapshot count, per-disk device
  info (device key, disk mode, RDM flags via reflection), NIC info, CPU/memory sizing, OS ID.
- `ensureChangeTrackingEnabled` — throws if unsupported; else `setChangeTrackingEnabled(true)` if
  not already enabled.
- `createSnapshot` — calls `ensureChangeTrackingEnabled` first (defense in depth); **memory=false
  always**; quiesce passed through (manager always calls with `quiesce=false`, §2.8).
- `queryChangedDiskAreas` — calls vSphere `queryChangedDiskAreas` via **reflection** (the VIM SDK
  signature apparently isn't directly bound), paginating with `startOffset +=
  responseStart+responseLength` until full disk capacity is covered — necessary because
  `QueryChangedDiskAreas` returns a bounded extent count per call.
- `getPowerState` — maps VMware power state to `UnmanagedInstanceTO.PowerState`, used by
  `requireSourceVmPoweredOff`.
- `removeSnapshot` — `removeSnapshotTask(snapshot, removeChildren=false, consolidate=true)`,
  waits for task completion, throws on failure/timeout. **Does not itself retry** — retry is a
  manager-layer concern (§2.8).

VDDK itself is invoked **only on the KVM host side** (§5) — this VMware-side service performs
vSphere Web Services API calls (snapshot/CBT/power-state/metadata) exclusively, never touches
disk bytes.

---

## 5. KVM agent-side command routing

All wrappers: `plugins/hypervisors/kvm/src/main/java/com/cloud/hypervisor/kvm/resource/wrapper/`.
**Verified**, see [STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md) for the per-backend
divergence within these wrappers.

| Wrapper | Command | Role |
|---|---|---|
| `LibvirtVmwareCbtPrepareCommandWrapper` (506 lines) | `VmwareCbtPrepareCommand` | Initial full copy from the baseline snapshot |
| `LibvirtVmwareCbtSyncCommandWrapper` (489 lines) | `VmwareCbtSyncCommand` | Per-cycle changed-block delta copy |
| `LibvirtVmwareCbtCutoverCommandWrapper` (758 lines) | `VmwareCbtCutoverCommand` | virt-v2v finalization of the target disk(s) |
| `LibvirtVmwareCbtCleanupCommandWrapper` (333 lines) | `VmwareCbtCleanupCommand` | Target-disk teardown + in-flight-copy-process kill |
| `LibvirtVmwareCbtRbdProbeCommandWrapper` (218 lines) | `VmwareCbtRbdProbeCommand` | Destination reachability probe (RBD or block-device) |
| `LibvirtConvertInstanceCommandWrapper` (1213 lines, +748 by this PR) | `ConvertInstanceCommand` | Cold/direct-VDDK conversion (pre-existing, heavily extended) |

All copy operations use `nbdkit -r -U - vddk ...` (the VDDK plugin) to expose the source disk
(pinned to a specific snapshot MOR) as an NBD export, then either `nbdcopy` (block/RBD targets,
preferred when available, via a localhost `qemu-nbd` bridge for RBD) or `qemu-img convert`
(fallback, and always for qcow2/file targets) to write it to the destination. Host-capability
gating (`hostSupportsVddkBlockCopy` et al.) lives in `LibvirtComputingResource.java`, reported to
the management server on every `ReadyCommand`.

---

## 6. Schema / DB

`engine/schema/src/main/resources/META-INF/db/schema-42210to42300.sql`, lines 649-774.
**Verified.**

- `cloud.vmware_cbt_migration` (lines 650-704): FKs to `data_center`, `account`, `user`,
  `vm_instance` (SET NULL), `vmware_data_center` (SET NULL), `cluster`, `host` (SET NULL),
  `storage_pool` (SET NULL); indexes on `zone_id`, `state`, `source_vm_name`. Additional
  `IDEMPOTENT_ADD_COLUMN` statements for several columns (706-716) alongside the full
  `CREATE TABLE`, suggesting the table evolved incrementally within this same upgrade file.
- Two `INSERT INTO cloud.configuration` seed rows (718-732).
- `cloud.vmware_cbt_migration_disk` (734-754) and `cloud.vmware_cbt_migration_cycle` (756-774),
  both `ON DELETE CASCADE` from `vmware_cbt_migration`; cycle table unique on
  `(migration_id, cycle_number)`.

VOs: `VmwareCbtMigrationVO`, `VmwareCbtMigrationDiskVO`, `VmwareCbtMigrationCycleVO` under
`engine/schema/src/main/java/com/cloud/vm/`. Passwords are `@Encrypt`-annotated
(`VmwareCbtMigrationVO.java:88-90`).

**Schema-target contradiction (record, do not resolve):** this file is named
`schema-42210to42300.sql` (targets 4.23.0.0) and `PendingReleaseNotes` files its entry under a
`4.23.0.0:` heading, but the GitHub PR milestone is **"4.24.0"** (milestone #47, confirmed live
via `gh pr view 13656 --json milestone`). See [KNOWN_ISSUES.md](KNOWN_ISSUES.md) §1.

---

## 7. Convergence with cold import — `UnmanagedVMsManagerImpl`

File: `server/src/main/java/org/apache/cloudstack/vm/UnmanagedVMsManagerImpl.java`. This PR's
diff to this file is **+642 lines** (`git diff fc832413d2c0f4012f2c512acc56866d156a17c4 HEAD --
server/.../UnmanagedVMsManagerImpl.java`), almost entirely additive. **Verified.**

The single shared convergence point: `importConvertedVmwareCbtInstanceToKvm`
(`UnmanagedVMsManagerImpl.java:1890-1941`) resolves template/offering/names, re-fetches source
metadata for sanity, checks resource limits and networking, resolves guest OS if unset, then
calls the **same** `importVirtualMachineInternal(...)` that the cold `ImportVmCmd` path uses.
This is the one code path that materializes an `UnmanagedInstanceTO` as a CloudStack KVM VM for
both cold and CBT-warm migrations. **Verified.**

Cold import additionally supports a "direct VDDK straight to pool" mode
(`vddkDirectConvertToPoolAllowedTypes`, `useVddk` flag) with no CBT-path equivalent, because the
CBT wrappers already write incrementally to the final destination throughout the migration
lifecycle rather than doing one point-in-time conversion.

**Unknown (not chased further, would need runtime testing):** exactly where the KVM libvirt
domain is defined and whether it auto-starts after import, versus being left `Stopped` pending an
operator `startVirtualMachine` call. `importVirtualMachineInternal` drives the standard
"import unmanaged instance" flow; the domain-define/start code was not conclusively located
inside this PR's touched files during this audit.

---

## 8. Windows-guest handling

**Reported** (PR body's Functional Test Report) + spot-checked structurally by the architecture
agent: a first-boot script is injected during virt-v2v conversion for Windows guests to bring
migrated data disks online (`(<vm>) Windows guest detected; injecting first-boot script to online
migrated disks`). Static-IP preservation ("Fix B") captures the source's VMware-Tools-reported
guest IP and matches it against the destination network's CIDR, falling back to `"auto"` when out
of range (unit-tested: `testAutoFillPreservesStaticIpThatFitsTargetCidr` and 5 sibling cases in
`UnmanagedVMsManagerImplTest`, **Verified** as of the test-ledger audit — see
[TEST_STATUS.md](TEST_STATUS.md)). UEFI/Secure-Boot/q35/video-hardware details are auto-applied
for imported Windows UEFI VMs unless the caller already supplied them
(`testApplyVmwareImportHardwareDetailsForUefiWindows` et al.).

**Reported** (PR body, "Windows guests" section): boot verification only was in scope; in-guest
networking and data contents were explicitly out of scope for the Windows test cells. See
[TEST_STATUS.md](TEST_STATUS.md) for the full breakdown of what was and wasn't independently
re-verified in this audit.
