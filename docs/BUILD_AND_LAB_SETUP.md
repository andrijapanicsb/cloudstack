# Build and Lab Setup — Fresh Reproducibility

All addresses, hostnames, usernames, and credentials below are **placeholders** —
`<ANGLE_BRACKET_NAME>` — by design, per the handoff's no-private-infrastructure rule. Nothing in
this document identifies real infrastructure. Where a step could not be safely drilled from a
clean checkout in this audit (e.g., anything needing a real vCenter/Ceph/Linstor cluster), that is
stated explicitly rather than assumed to work.

## 1. Prerequisites

| Component | Version used by this PR (Verified from code/docs) | Notes |
|---|---|---|
| Java | 17 (Temurin) | Confirmed working this session: `openjdk 17.0.19` |
| Maven | 3.9.x | Confirmed working this session: 3.9.9 |
| CloudStack | branch `kvm-linstor-importvm-adoption` off `upstream/main` (merge-base `fc832413d2c0`) | |
| KVM conversion host OS | **EL9 or newer** required for the direct-to-pool path (recent `virt-v2v`) | PR body, "Scope of this testing" |
| `virt-v2v` / `virt-v2v-in-place` | Recent enough to support `-i libvirtxml` and in-place conversion | [STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md) §5.3 |
| `nbdkit` + VDDK plugin | VMware VDDK 8.0.3 (matches the vSphere version this PR was validated against, per PR body) | |
| `qemu-img` / `qemu-nbd` / `qemu-io` | Must include RBD block-driver support for the Ceph path | |
| `nbdcopy` (optional) | Falls back to `qemu-img convert` if absent — not a hard requirement | |
| VMware vCenter/ESXi | 7.0.3 and 8.0.3 both referenced in PR body testing | |
| Ceph | Any RBD pool reachable from the KVM conversion host | |
| Linstor | Controller + satellite; KVM conversion host **must be a satellite** with the destination pool attached (enforced in code, not just documented) | |

## 2. Build

```bash
git clone https://github.com/<YOUR_FORK_OWNER>/cloudstack.git
cd cloudstack
git checkout kvm-rbd-vmware-migration   # the actual PR head branch name
mvn -q -pl server -DskipTests compile   # sanity — confirmed exit 0 in this audit
```

## 3. Run the automated test suite (safe, no external infrastructure needed)

These commands were **actually run and passed** in this audit (see
[TEST_STATUS.md](TEST_STATUS.md) §2) — a safe first step for anyone resuming:

```bash
mvn -pl plugins/hypervisors/kvm -am \
  -Dtest=LibvirtVmwareCbtCutoverCommandWrapperTest,LibvirtVmwareCbtPrepareCommandWrapperTest,LibvirtVmwareCbtSyncCommandWrapperTest,LibvirtVmwareCbtCleanupCommandWrapperTest,LibvirtCheckVolumeCommandWrapperTest \
  -DfailIfNoTests=false -Dsurefire.failIfNoSpecifiedTests=false test

mvn -pl server -am -Dtest=UnmanagedVMsManagerImplTest,VmwareCbtStorageTargetTest \
  -DfailIfNoTests=false -Dsurefire.failIfNoSpecifiedTests=false test
```

To run the **full** new/changed test set from this PR (not individually re-verified in this
audit — see the gap noted in [TEST_STATUS.md](TEST_STATUS.md) §7), add the remaining files listed
in [TEST_STATUS.md](TEST_STATUS.md) §1 to the `-Dtest=` list, split across their respective
modules (`plugins/hypervisors/kvm`, `server`, `engine/orchestration`).

## 4. CloudStack deployment (management server + KVM host)

Standard CloudStack KVM deployment — this PR does not change deployment mechanics, only adds
capability. Beyond the standard steps (out of scope to fully restate here — see upstream
CloudStack install docs), this feature specifically needs:

1. **On the KVM host** (`<KVM_HOST_IP>`, via SSH as `<KVM_HOST_SSH_USER>`):
   - Install `virt-v2v`, `nbdkit`, `nbdkit-vddk-plugin` (not pulled in by the CloudStack agent
     RPM automatically).
   - Extract the VMware VDDK tarball to a directory, and set `vddk.lib.dir=<VDDK_ROOT_DIR>` in
     `agent.properties` — **the root extraction directory, not its `lib64` subdirectory**; the
     agent appends `lib64` itself.
   - Provide a virtio-win driver ISO recent enough for your target guest OS versions (an
     EL9-bundled version may be too old for newer Windows Server releases — verify driver
     coverage before relying on Windows guest imports).
   - Set `guest.cpu.mode=custom` and `guest.cpu.model=<A_MODEL_THIS_HOST_ACTUALLY_SUPPORTS>` —
     **the model must appear as `usable='yes'` in `virsh domcapabilities` output on that specific
     host.** Using an unsupported model does not just fail imports — it breaks starting *every*
     new domain on that host, including CloudStack system VMs, while already-running domains keep
     going (so the failure surfaces hours later, disconnected from the change that caused it).
   - Ensure `/etc/cloudstack/agent/uefi.properties` exists and `edk2-ovmf` is installed if testing
     UEFI guests.
   - Restart the agent, then confirm via `listHosts details=all` (or equivalent) that the host
     reports the capability flags listed in [STORAGE_BACKEND_MATRIX.md](STORAGE_BACKEND_MATRIX.md)
     §5.4 as `true` for whichever backend(s) you intend to test.
2. **For Ceph RBD:** a reachable RBD pool; qemu on the KVM host built with the RBD block driver.
   No special "connect" step is needed — RBD is reached directly by URI.
3. **For Linstor:** the KVM host must be registered as a LINSTOR **satellite** with the
   destination storage pool actually attached — this is enforced in code
   (`storagePoolHostDao.findByPoolHost` checks), not just documentation. Confirm
   `modprobe drbd` succeeds and the DRBD kernel module version matches the running kernel before
   testing.
4. **CloudStack global settings** worth reviewing before testing at scale (names only, set values
   appropriate to your environment): cluster memory-overprovisioning factor (small/nested test
   hosts commonly need headroom above the default), storage-pool capacity disable-thresholds
   (note: the correct setting names are `pool.storage.capacity.disablethreshold`,
   `pool.storage.allocated.capacity.disablethreshold`,
   `pool.storage.allocated.resize.capacity.disablethreshold` — **not** `storage.*`, a naming trap
   documented here because it silently blocks new allocations, including system-VM recreation, if
   set on the wrong key).

## 5. VMware source setup

- A source VM/template on a reachable vCenter/ESXi, with VMware Tools installed if static-IP
  preservation is to be tested.
- For warm CBT testing: confirm Changed Block Tracking is *supportable* on the source (hardware
  version, no unsupported disk modes — independent-mode and physical-RDM disks are explicitly
  rejected by this PR's start-time validation, see [ARCHITECTURE.md](ARCHITECTURE.md) §2.3).

## 6. Invoking the feature (API-level, placeholders throughout)

```bash
# Preflight (read-only, safe to run repeatedly)
cloudmonkey checkVmwareCbtMigrationPrerequisites \
  zoneid=<ZONE_ID> vcenter=<VCENTER_HOST> username=<VCENTER_USER> password='<VCENTER_PASSWORD>' \
  datacentername=<VCENTER_DATACENTER> sourcevmname=<SOURCE_VM_NAME> \
  storagepoolid=<DEST_POOL_ID> clusterid=<DEST_CLUSTER_ID>

# Start a warm CBT migration
cloudmonkey startVmwareCbtMigration \
  zoneid=<ZONE_ID> vcenter=<VCENTER_HOST> username=<VCENTER_USER> password='<VCENTER_PASSWORD>' \
  datacentername=<VCENTER_DATACENTER> sourcevmname=<SOURCE_VM_NAME> \
  storagepoolid=<DEST_POOL_ID> clusterid=<DEST_CLUSTER_ID> serviceofferingid=<OFFERING_ID> \
  forced=true

# Run delta cycles (repeat as needed; async job — poll the job result, do not assume synchronous
# success/failure, see ARCHITECTURE.md §2.3 re: async pitfalls)
cloudmonkey syncVmwareCbtMigration id=<MIGRATION_ID>

# Once ready (poll listVmwareCbtMigrations for state=ReadyForCutover), power off the source VM
# gracefully via vCenter/govc, THEN:
cloudmonkey cutoverVmwareCbtMigration id=<MIGRATION_ID>

# Cancel / cleanup
cloudmonkey cancelVmwareCbtMigration id=<MIGRATION_ID>
cloudmonkey deleteVmwareCbtMigration id=<MIGRATION_ID> cleanup=true
```

For cold/direct-VDDK import: `importVm` with `usevddk=true forceconverttopool=true` against a
**powered-off** source VM (required for the direct-to-pool path). For volume adoption:
`importVm importsource=shared` with a plain `networkid=` parameter (not `nicnetworklist`).

## 7. Log collection

- Management server: standard `management-server.log`, filter for the migration UUID.
- KVM agent: standard `agent.log`, plus `/var/log/libvirt/qemu/<instance-name>.log` for the
  actual QEMU command line used (useful for confirming the CPU model actually applied, per the
  guest.cpu.model landmine above).
- The shelled-out copy commands (`nbdkit`/`nbdcopy`/`qemu-img convert`) log their own stdout/stderr
  through `executeLoggedBash` — captured in the agent log around the relevant command timestamp.

## 8. Cleanup / reset between test runs

- `cancelVmwareCbtMigration` + `deleteVmwareCbtMigration cleanup=true` for any in-progress or
  terminal migration.
- Expunge any imported test VM via the standard CloudStack VM lifecycle.
- For RBD: confirm no `cloudstack-cbt-*` images remain in the pool (`rbd ls <pool>` **run from a
  Ceph node, not the KVM host** — the `rbd` CLI is not installed on the KVM host; qemu talks to
  Ceph via librbd directly).
- For Linstor: confirm no `cs-cbt-*`/`cs-<uuid>-d*` resources remain via the Linstor controller.

## 9. What this audit could and could not safely drill

**Could and did:** clean compile (`mvn -q -pl server -DskipTests compile`, exit 0) and the unit
test runs in §3, all from the actual checked-out worktree — no production infrastructure touched.

**Could not (out of scope for this audit, requires real infrastructure not available in this
session):** an actual end-to-end migration against a live vCenter/Ceph/Linstor stack. That
remains the single biggest thing a future maintainer should do before trusting this feature in
production — see [RESTART_CHECKLIST.md](RESTART_CHECKLIST.md).
