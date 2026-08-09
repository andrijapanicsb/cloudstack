# Storage Backend Matrix — File/NFS vs. Ceph RBD vs. Linstor

Companion to [ARCHITECTURE.md](ARCHITECTURE.md). All facts **Verified** by direct code read
against `D:\GIT\rbd-suite-wt`, HEAD `26c0e9afcb17615c62f103b1b8edd501f362be92`, unless marked
**Reported**/**Inferred**/**Unknown**. Each backend is documented separately — no behavior is
assumed to carry over between backends.

Central dispatch: `server/src/main/java/org/apache/cloudstack/vm/VmwareCbtStorageTarget.java:41-64`
maps destination pool type → target-storage type:

| Pool type | `VmwareCbtTargetStorageType` | `requiresInPlaceFinalization` |
|---|---|---|
| `NetworkFilesystem` / `Filesystem` / `SharedMountPoint` | `QCOW2_FILE` | false (fallback available) |
| `RBD` | `RBD_RAW` | **true** (no fallback unless admin overrides via `vmware.cbt.allow.non.inplace.finalization`) |
| `Linstor` | `RAW_BLOCK_DEVICE` | **true** (same override) |
| anything else | unsupported | — |

---

## 1. Existing / File (NFS, local) backend

| Aspect | Behavior | Citation |
|---|---|---|
| Target selection (cold) | virt-v2v writes qcow2 via `-o local -os <path> -of qcow2` | `LibvirtConvertInstanceCommandWrapper.java:341-352` |
| Target selection (CBT) | File path under `<pool>/cloudstack-cbt/<migrationUuid>/`, directory created if absent | `LibvirtVmwareCbtPrepareCommandWrapper.java:88-97, 176-181` |
| Thin/thick, sparse | Sparse by qcow2 file-format construction; `isVolumeZeroInitialized` is **never called** for this branch | `LibvirtVmwareCbtPrepareCommandWrapper.java:219-227` |
| Copy mechanism | `nbdkit -r -U - vddk ... --run 'qemu-img convert -f raw -O qcow2 "$uri" <path>'` — **always** qemu-img convert, no nbdcopy | `LibvirtVmwareCbtPrepareCommandWrapper.java:378-382` |
| Cleanup | Whole `/cloudstack-cbt/<migrationUuid>/` directory tree deleted (`Files.walk(...).sorted(Comparator.reverseOrder())`) | `LibvirtVmwareCbtCleanupCommandWrapper.java:290-332` |
| Cleanup hardening | **Does NOT get** the in-flight-copy-process-kill or retry-with-backoff hardening added by commits `36f070dc06`/`e25e9ee03f` — those only fire for `RBD_RAW`/`RAW_BLOCK_DEVICE` branches | `LibvirtVmwareCbtCleanupCommandWrapper.java:59-93` (gate), `75-85` (file branch) |
| Retry behavior | Naturally idempotent (`Files.isDirectory` guard → no-op on missing dir), but no process-kill, no backoff | same |
| Capacity checks | **Unknown — no evidence found** (see §5) | — |

---

## 2. Ceph RBD backend

| Aspect | Behavior | Citation |
|---|---|---|
| Target selection (cold, direct VDDK) | `isDirectRbdVddkImport` gates on temp-pool type `RBD`; image name via `buildRbdImageName` (`"%s-disk-%03d"`) | `LibvirtConvertInstanceCommandWrapper.java:257-269, 982-984` |
| Target selection (CBT) | `getRbdTargetImageName` (`cloudstack-cbt-<uuid>-<diskId>-<name>`); stale same-name image deleted first (`deleteRbdTargetIfExists`) | `LibvirtVmwareCbtPrepareCommandWrapper.java:213-214, 287-310` |
| Host capability required (staged) | `HOST_VDDK_RBD_DIRECT_IMPORT_SUPPORT`, filtered via `filterHostsWithVddkRbdDirectImportSupport` | `UnmanagedVMsManagerImpl.java:786-791` |
| Host capability required (CBT) | `HOST_VDDK_BLOCKCOPY_RBD_SUPPORT` | `VmwareCbtMigrationManagerImpl.java:1113-1121, 1179-1181` |
| **Thin/thick & zero handling** | **Hard-coded assumption, not a real capability check.** `nbdcopy --destination-is-zero` is appended **unconditionally** (comment: "The fresh raw image reads as zeros, so `--destination-is-zero` keeps it sparse"). The **fallback** qemu-img-convert path for RBD does **not** pass `-n`/`--target-is-zero` at all. `isVolumeZeroInitialized` is **never overridden for RBD** — `LibvirtStoragePool` (RBD's pool class) inherits the `KVMStoragePool` interface default of `false` and RBD never calls the method. | `LibvirtConvertInstanceCommandWrapper.java:643-649, 679-684`; `LibvirtVmwareCbtPrepareCommandWrapper.java:391-394`; `KVMStoragePool.java:86-88` |
| Copy mechanism | Pre-create with `qemu-img create -f raw <rbd-uri> <size>`, serve locally via `qemu-nbd --fork --persistent --shared=8 --format=raw --bind=127.0.0.1 --port=<p> <rbd-uri>`, then `nbdcopy --destination-is-zero "$uri" nbd://localhost:<p>` | `LibvirtConvertInstanceCommandWrapper.java:639-681`; `LibvirtVmwareCbtPrepareCommandWrapper.java:391-407` (`buildRbdNbdcopyBridgeCommand`) |
| Finalization (in-place) | Each RBD image bridged through its own localhost `qemu-nbd`; synthetic libvirt XML fed to `virt-v2v-in-place -i libvirtxml` with `<disk type='network' protocol='nbd'>` (native `rbd:` disks can't be used — virt-v2v's libvirtxml input has no libvirt connection to resolve the cephx `<auth>` secret) | `LibvirtConvertInstanceCommandWrapper.java:825-920` (`runRbdInPlaceFinalizationOverNbdBridges`) |
| Probe | `LibvirtVmwareCbtRbdProbeCommandWrapper` creates `cloudstack-cbt-probe-<uuid>` via `qemu-img create` + `qemu-io write/read`, always cleaned up in `finally` | `LibvirtVmwareCbtRbdProbeCommandWrapper.java:38, 179-186` |
| **Adoption path** | `LibvirtCheckVolumeCommandWrapper`/`LibvirtGetVolumesOnStorageCommandWrapper` list `RBD`; inspected via `rbd:` URI + qemu-img, **no connect/disconnect step needed** (network-reachable directly); identified by bare **name**, no CloudStack-side transformation | `LibvirtCheckVolumeCommandWrapper.java:50-55, 76-77, 181-184`; `UnmanagedVMsManagerImpl.java:3534` |
| **Multi-volume adoption** | **Not supported via `importVm`** — the shared/local import path rejects `dataDiskOfferingMap` ("Import the root disk first, then adopt each data disk with importVolume"); root-only for the single-call VM-import API. Multi-volume adoption goes through the separate standalone `importVolume`/`VolumeImportUnmanageService` API instead. | `UnmanagedVMsManagerImpl.java` (diff, comment ~line 3215) |
| **In-use guard** | **No RBD-specific cluster-wide in-use check.** Relies only on the pre-existing host-local qemu-img exclusive-open lock (`isDiskFileLocked`) — there is nothing analogous to Linstor's cluster-wide `getVolumeInUseNode`. | `LibvirtCheckVolumeCommandWrapper.java:154, 192-195` |
| Cleanup | `deleteRbdTargetImages`: requires `RBD` pool; extracts image name only if it contains the `cloudstack-cbt-<migrationUuid>-` marker; kills in-flight `nbdcopy`/`qemu-img convert`/`qemu-io` processes matching that marker via `pkill -f`; deletes with up to 3 retries + positive-absence check via `listPhysicalDisks()` | `LibvirtVmwareCbtCleanupCommandWrapper.java:95-100, 105, 186-199, 205-264` |
| Capacity checks | **Unknown — no evidence found** (see §5) | — |

**Asymmetry flagged for [KNOWN_ISSUES.md](KNOWN_ISSUES.md):** RBD's "fresh image is zero"
guarantee is asserted only by hard-coded flag/comment, never verified through the pluggable
`isVolumeZeroInitialized` capability method that Linstor actually implements (§3). If an RBD pool
or provider ever reused non-zeroed extents for a newly created image, the unconditional
`--destination-is-zero`/absence of `-n` would silently produce stale-data corruption on the
target — this is a **design asymmetry**, not something this audit observed failing.

---

## 3. Linstor (DRBD) backend

| Aspect | Behavior | Citation |
|---|---|---|
| Target selection | Pre-created as a **real local block device** before copying: `targetPool.createPhysicalDisk(diskName, RAW, THIN, capacityBytes, null)`, then written to via the returned `KVMPhysicalDisk.getPath()`. Names shortened (`<uuid>-d%02d`) — LINSTOR resource names cap at 48 chars and get a `cs-` prefix. | `LibvirtConvertInstanceCommandWrapper.java:704-720, 986-993`; `LibvirtVmwareCbtPrepareCommandWrapper.java:272-285` |
| DRBD device-path resolution | Not in the KVM agent wrappers at all — done inside the (pre-existing) Linstor storage plugin: `LinstorUtil.getDevicePathFromResource` → `/dev/drbd/by-res/<rscName>/0`; `LinstorStorageAdaptor.getPhysicalDiskPath` prepends `RSC_PREFIX="cs-"` | `plugins/storage/volume/linstor/.../LinstorUtil.java:366-368, 404-414, 70`; `LinstorStorageAdaptor.java:95` |
| **Thin/thick & zero handling** | **A real, queried capability check** — genuinely different from RBD's hard-coded assumption: `LinstorStoragePool.isVolumeZeroInitialized` → `LinstorUtil.resourceSupportZeroBlocks(pool, "cs-"+volumeName)`, which queries the Linstor controller's `viewResources` and returns `true` **only if every diskful volume's `ProviderKind` is `LVM_THIN`, `ZFS`, or `ZFS_THIN`** (plain LVM/"thick" → `false`). **Fails closed**: unreachable controller, empty response, missing volumes, or unknown provider kind all return `false`. | `LinstorStoragePool.java:219-228`; `LinstorUtil.java:627-679` (commit `0cbd57417f`) |
| Copy mechanism | No qemu-nbd bridge — `nbdkit ... vddk ... --run 'nbdcopy [--destination-is-zero] "$uri" <devicePath>'` (or qemu-img-convert fallback) writes **directly** into the pre-created `/dev/drbd...` device | `LibvirtConvertInstanceCommandWrapper.java:747-751`; `LibvirtVmwareCbtPrepareCommandWrapper.java:375-382` |
| Finalization | `virt-v2v-in-place` with `<disk type='block'><source dev='<devicePath>'/>` — **no NBD hop at all** | `LibvirtConvertInstanceCommandWrapper.java:767-791, 962-980` (`buildDirectBlockDeviceLibvirtXml`) |
| Probe | `executeBlockDeviceProbe` pre-creates a real device, `qemu-io write/read` directly — no `qemu-img create`, no RBD URI | `LibvirtVmwareCbtRbdProbeCommandWrapper.java:89-118` |
| **Adoption path — resource attach required first** | Before qemu-img can inspect the volume, the resource must be made **available/diskless-attached** on the inspecting host: `poolMgr.connectPhysicalDisk(...)` called first, `disconnectPhysicalDisk` after — RBD needs **neither** step | `LibvirtCheckVolumeCommandWrapper.java:74-92`; `LibvirtGetVolumesOnStorageCommandWrapper.java:67-85` (commit `0009e1f96d`) |
| Naming | `diskpath` API parameter is the **bare** identifier without the `cs-` prefix — the prefix is applied transparently inside the plugin; CloudStack server code never manipulates the string (`checkVolumeCommand.setSrcFile(diskPath)`, unchanged). Note the `cs-` convention itself **pre-dates this PR** (existing Linstor plugin naming scheme) — this PR's contribution is *wiring the adoption/CheckVolume/GetVolumesOnStorage paths through it*. | `UnmanagedVMsManagerImpl.java:3534`; `LinstorUtil.java` |
| **In-use-on-another-node guard** | `KVMStoragePool.getVolumeInUseNode(String)` defaults to `null` ("`IN_USE_NODE_UNKNOWN`... fail closed"), overridden by `LinstorStoragePool.getVolumeInUseNode` → `LinstorUtil.isResourceInUse`, which queries the controller's `resourceList` and returns the node name of any resource whose DRBD state `isInUse()==true`. **Fails closed** to `IN_USE_NODE_UNKNOWN` if the controller is unreachable/returns null. Both `LibvirtCheckVolumeCommandWrapper` and `LibvirtGetVolumesOnStorageCommandWrapper` OR this into the `IS_LOCKED` detail. **RBD has no equivalent** (§2). | `KVMStoragePool.java:97-99`; `LinstorStoragePool.java:230-238`; `LinstorUtil.java:305-337` (commit `0cbd57417f`); `LibvirtCheckVolumeCommandWrapper.java:154-169`; `LibvirtGetVolumesOnStorageCommandWrapper.java:145-159` |
| **Multi-volume adoption** | Commit `06aa3e573e` message: "adopt existing Linstor **root and data** volumes." Per the same rule as RBD, the single `importVm` call still only adopts the ROOT disk; root+data multi-volume adoption for Linstor happens via the **separate** `VolumeImportUnmanageService`/`importVolume` API (that commit extended `SUPPORTED_STORAGE_POOL_TYPES_FOR_KVM` to include Linstor there) — same rule as RBD, not a Linstor-specific limitation. | `UnmanagedVMsManagerImpl.java` diff; `06aa3e573e` commit message |
| Cleanup | `deleteBlockDeviceTargetVolumes`: requires `Linstor` pool; extracts volume name using a **shorter** marker `cbt-<first-8-hex-of-migrationUuid>-` (not the full UUID — 48-char LINSTOR name limit); kills in-flight processes matching that shorter marker; same `deletePhysicalDiskWithRetries`/`isTargetAlreadyAbsent` idempotency machinery as RBD, **plus backoff** (`Thread.sleep(2000L * attempt)`) because deletion can transiently fail right after killing a copy process ("DRBD demotion, qemu teardown") | `LibvirtVmwareCbtCleanupCommandWrapper.java:127-174, 201-264` |
| Prerequisite this PR added | `listPhysicalDisks` for Linstor was **previously unimplemented** (`throw new UnsupportedOperationException`) — this PR implements it, which is a prerequisite for the idempotent-absence check to work at all for Linstor | `LinstorStorageAdaptor.java:593-621` diff |
| Capacity checks | **Unknown — no evidence found** (see §5) | — |

---

## 4. `getSupportedImageFormatForCluster` — shared behavior

`VolumeOrchestrator.getSupportedImageFormatForCluster(hyperType, poolType)` forces
`ImageFormat.RAW` for **both** RBD and Linstor pool types on KVM (`VolumeOrchestrator.java:1338-1348`,
unit-tested: `testGetSupportedImageFormatForClusterKvmRbdIsRaw`,
`testGetSupportedImageFormatForClusterKvmNonRbdIsQcow2` in `VolumeOrchestratorTest.java` — note the
test name says "NonRbd" but the assertion covers the Linstor case too per the storage-agent's
code read). **Verified.**

---

## 5. Cross-cutting findings

### 5.1 Capacity checks and reservations

**Unknown — no evidence found in the diffed files.** Grepping `UnmanagedVMsManagerImpl.java` and
`VmwareCbtMigrationManagerImpl.java` for `getAvailable()`, `capacity`, `CapacityManager`,
`getUsed()` returns no free-space/pool-capacity check for **any** backend. The only size-related
validation found is a pre-existing disk-offering-size ≥ source-disk-size check
(`checkUnmanagedDiskAndOfferingForImport`) and generic CloudStack account resource-limit
`CheckedReservation`s — both govern quotas, not physical free space on the destination pool. This
absence is **uniform across all three backends**, not a per-backend gap.

### 5.2 General vs. backend-specific retry hardening

Commits `36f070dc06` ("make VMware CBT cancel/cleanup abort in-flight copies and retry deletes")
and `e25e9ee03f` ("make target cleanup idempotent so cancelled migrations can be deleted") touch
only `LibvirtVmwareCbtCleanupCommandWrapper.java`, and their logic
(`killInFlightCopyProcesses`, `deletePhysicalDiskWithRetries`, `isTargetAlreadyAbsent`) is invoked
identically from the RBD branch and the Linstor branch (different marker strings only) — **general
to both block backends, explicitly does not apply to the file/NFS branch** (§1). Idempotency
guarantee: a second cleanup pass treats a target the pool listing confirms is gone as **success**,
not failure — but only if the pool can be positively enumerated; if `listPhysicalDisks()` itself
fails, the target is conservatively assumed to still exist. Commit `26c0e9afcb` (current HEAD)
adds retried removal of the *VMware-side* baseline/delta snapshot on cancel/fail — that is
source-side (vCenter), identical across all three destination backends.

### 5.3 Required host-side packages/services

| Requirement | Applies to | Citation |
|---|---|---|
| `virt-v2v`, `nbdkit` + VDDK plugin, `qemu-img`/`qemu-nbd`/`qemu-io` | All backends | `LibvirtComputingResource.java:6302-6307` |
| `nbdcopy` (optional — pure optimization) | All backends | `LibvirtComputingResource.java:6341-6343` — "callers fall back to qemu-img convert when it is absent" |
| qemu built with RBD block driver | RBD only | `LibvirtComputingResource.java:6313-6316, 6413-6414` (`hostSupportsQemuRbd`/`hostSupportsVddkBlockCopyRbd`) |
| In-place virt-v2v (`virt-v2v-in-place` binary or `--in-place` CLI option) | RBD + Linstor (required); File (optional fallback exists) | `LibvirtComputingResource.java:6330-6363` |
| Host must be a LINSTOR satellite with the destination pool attached | Linstor only, **enforced in code** via `storagePoolHostDao.findByPoolHost` checks, not just documented | `UnmanagedVMsManagerImpl.java` diff (~698-726, ~771-777, ~913-934) |

No local `linstor` CLI or DRBD kernel-module check was found in the KVM agent wrapper files —
availability is implicit via the pre-existing `LinstorStorageAdaptor`/`LinstorUtil` plugin, which
calls the Linstor controller's REST API (`DevelopersApi`), not a local CLI.

### 5.4 Host-capability probing (renamed in this PR)

Commit `17e7822265` ("Consolidate KVM host-capability probe names across RBD features") renamed
the probe keys to feature-neutral names, now in `api/src/main/java/com/cloud/host/Host.java:60-74`:
`host.vddk.support`, `host.vddk.blockcopy.support`, `host.vddk.blockcopy.inplace.finalization.support`,
`host.vddk.blockcopy.rbd.support`, `host.vddk.rbd.direct.import.support`,
`host.virtv2v.inplace.support`, `host.qemu.rbd.support`. Reported on every `ReadyCommand`.
**RBD and Linstor are gated by genuinely different capability flags** — CBT-to-RBD requires
`HOST_VDDK_BLOCKCOPY_RBD_SUPPORT`; CBT-to-Linstor requires only
`HOST_VDDK_BLOCKCOPY_INPLACE_FINALIZATION_SUPPORT`, **not** the RBD flag.

### 5.5 Docs-vs-code contradictions found (cross-referenced against [KNOWN_ISSUES.md](KNOWN_ISSUES.md))

- `docs/vmware-cbt-migration.md:114,126,406,412` still references the **pre-rename** key
  `host.vmware.cbt.support`. Current code emits/checks `host.vddk.blockcopy.support`. This same
  doc's "Related host detail keys" list (lines 120-130) also **omits** all 5 RBD/Linstor-specific
  keys from §5.4 — an operator following only this doc would not know they exist.
  `docs/vmware-cbt/README.md` (the newer/larger doc) correctly uses the renamed keys — the two
  docs now disagree with each other, and `vmware-cbt-migration.md` disagrees with the code.
- Everything else spot-checked (zero-block skip conditions, qemu-nbd bridging only for RBD, `cs-`
  prefix behavior, single-writer DRBD guard, `docs/vmware-cbt-migration.md:153-163`
  "Destination-specific requirements", `docs/vmware-linstor-migration.md` lines 32-238) was found
  **consistent** with the code — recorded here so a future reader doesn't have to re-verify what
  was already checked.
