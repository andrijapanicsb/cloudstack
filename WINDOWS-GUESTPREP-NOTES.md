# VMware→KVM import: Windows guest post-migration fixes

Branch: `vmware-import-windows-guestprep` (off `upstream/main`). Separate from PR #13656.
Status: **code-complete + unit-tested for the deterministic parts; one seam deferred to lab.**
Not committed (awaiting review). Author-only commits when approved.

## Problem (known stock-virt-v2v gaps on Windows)
1. **Static IPs lost** — after conversion the guest sees "new" NICs that default to DHCP; a
   previously-static Windows guest loses its address.
2. **Secondary disks Offline/Read-only** — under the default Windows SAN policy, migrated data
   disks come up Offline (and sometimes read-only).

vmk-forge (the mass-migration tool) automates both; the CloudStack-native path did not.

## Fix A — bring migrated disks online (DONE, deterministic)
- Windows-gated virt-v2v `--firstboot` batch injected on both conversion paths (staged OVA and
  direct VDDK). Script sets SAN policy `OnlineAll` and, via PowerShell Storage cmdlets, clears
  `IsOffline`/`IsReadOnly` on all disks. Storage cmdlets exist on Server 2012+.
- Server sets `windowsGuest` on `ConvertInstanceCommand` from the already-existing OS detection
  (`isWindowsGuest`, null-safe). Agent writes the script to a temp file, passes `--firstboot`,
  cleans it up. Best-effort: if the script can't be written, conversion proceeds without it.
- Files: `ConvertInstanceCommand` (windowsGuest), `UnmanagedVMsManagerImpl.isWindowsGuest` +
  wiring, `LibvirtConvertInstanceCommandWrapper` (`WINDOWS_ONLINE_DISKS_FIRSTBOOT`,
  `writeWindowsOnlineDisksFirstbootScript`, `shellQuote`, both conversion methods).
- Tests: content, real file write, shellQuote, VDDK injects `--firstboot` for Windows, skips for
  non-Windows. **19/19 green.**
- **Lab validation needed:** confirm virt-v2v actually installs the `.bat` firstboot on our
  virt-v2v version, and that the PowerShell Storage cmdlets run on Server 2012/2016.

## Fix B — preserve static IPv4 via virt-v2v --mac (CORE DONE, capture deferred)
- Pure builder `buildVirtV2vMacMapping(mac, ipv4, gateway, prefixLength, dnsServers)` →
  `mac:ip:addr[,gw[,prefix[,dns...]]]` (positional; inner gaps kept empty, e.g. `addr,,24`).
  `buildStaticIpMacParams(instance)` emits one `--mac ...` per usable NIC.
- New command field `staticIpMacParams`, folded into the agent's virt-v2v splice **separately from
  the operator `extraParams` allow-list** (it's server-generated/trusted).
- Gated by new global config **`convert.vmware.instance.preserve.guest.static.ip` (default false)** —
  only useful on L2/no-DHCP destination networks; leave off when the destination has VR DHCP.
- New `UnmanagedInstanceTO.Nic` fields: `ipv4PrefixLength`, `ipv4Gateway`, `dnsServers`.
- Tests: full/ip-only/prefix-without-gateway/null-guards, disabled-returns-null, per-NIC build,
  skip-NIC-without-IPv4. **server 97/97 green** (incl. these 7).

### Deferred seam (task 15): capture prefix/gateway/DNS from VMware guest info
`VmwareHelper.getUnmanageInstanceNics` today captures only bare IPv4 (from
`GuestNicInfo.getIpAddress()`). To fill the new Nic fields:
- **prefix** ← `GuestNicInfo.getIpConfig().getIpAddress()` → `NetIpConfigInfoIpAddress`
  (`getIpAddress()` + `getPrefixLength()`).
- **gateway/DNS** ← `guestInfo.getIpStack()` → `GuestStackInfo.getIpRouteConfig()` (default route
  `0.0.0.0/0` gateway) + `getDnsConfig()` (DNS list). Gateway is not cleanly per-NIC in guest info.
Deferred because: no precedent in the codebase, and the exact field population is VMware-Tools /
guest-state dependent — must be **observed on the real Windows guest first**, then coded, then
validated end-to-end. Without it, Fix B (if the toggle is enabled) currently yields IP-only
mappings, so keep the toggle OFF until the capture lands.

## Alternative worth documenting (may remove most Fix-B need)
On destination networks with **VR DHCP**, don't touch the guest at all: import the NIC with the
**same MAC and IP**, let the Windows NIC come up DHCP, and the VR leases the same address back by
MAC. `--mac` static preservation is then only needed for L2 networks without DHCP.

## Build/verify commands (offline)
Note: local ~/.m2 gets polluted by whichever branch was last installed. To validate this branch:
`mvn -o -pl api,core install -DskipTests` then
`mvn -o -pl plugins/hypervisors/kvm test -Dtest=LibvirtConvertInstanceCommandWrapperTest` and
`mvn -o -pl server test -Dtest=UnmanagedVMsManagerImplTest`.

## Lab checklist (when the Windows guinea pig is up — AFTER the PR #13656 campaign)
1. Baseline: migrate stock (no fixes) → confirm NICs→DHCP and disks Offline (documents the gap).
2. Fix A: migrate with fix → data disks Online + read-write; check `%SystemDrive%\cloudstack-firstboot.log`.
3. Capture: dump real `GuestNicInfo.ipConfig` + `ipStack` for the guest; implement task 15 against it.
4. Fix B: enable the toggle, migrate → static IPv4 (addr+prefix+gw+dns) preserved in-guest.
