# Codex Handoff: VMware IPv4 Preservation via virt-v2v

This file captures the working context from the Codex chat so a new session can continue without relying on chat history.

## Workspace

- Parent folder: `C:\Users\andri\Documents\Codex\2026-05-07\https-github-com-prashanthr2-vmware-to`
- Active CloudStack worktree: `C:\Users\andri\Documents\Codex\2026-05-07\https-github-com-prashanthr2-vmware-to\cloudstack-next-import-prs`
- Current branch: `preserve-v2v-static-ip`
- Fork remote: `fork -> https://github.com/andrijapanicsb/cloudstack.git`
- Apache remote: `origin -> https://github.com/apache/cloudstack.git`
- Static IPv4 branch pushed to fork: `fork/preserve-v2v-static-ip`
- Latest commit pushed: `fceac43b54f6dbd5af986ef845cd9104263b3e97`
- Commit message: `Preserve VMware IPv4 config during v2v import`
- Working tree was clean after push.

## Related PRs

- PR `#13128`: `https://github.com/apache/cloudstack/pull/13128`
  - Title: `Allow preserving duplicate MACs during VM import`
  - Branch: `andrijapanicsb:allow-duplicate-mac-import`
  - Purpose: duplicate MAC support and UI wording around MAC conflict behavior.
- Static IPv4 preservation work has not yet been opened as an Apache PR.
  - GitHub fork PR creation URL from push:
    `https://github.com/andrijapanicsb/cloudstack/pull/new/preserve-v2v-static-ip`

## User Goal

Extend VMware-to-KVM import/migration in Apache CloudStack so Windows VMs can preserve IPv4 configuration during virt-v2v conversion.

Problem: Windows treats the converted VirtIO NIC as a new adapter after migration from VMware NICs, so static IPv4 settings may be lost and the VM may fall back to DHCP.

Desired mechanism: use native virt-v2v `--mac` support:

```text
--mac aa:bb:cc:dd:ee:ff:ip:ipaddr[,gw[,len[,ns,ns,...]]]
```

Example:

```text
--mac 00:50:56:ac:90:79:ip:192.168.10.50,192.168.10.1,24,8.8.8.8,1.1.1.1
```

## Design Decisions Agreed

- Do not inspect guest disks.
- Do not use Cloudbase-init.
- Do not depend on DHCP.
- Do not use virt-inspector, guestfish, qemu-img inspection, or invasive customer-data inspection.
- Use VMware Tools / vCenter guest info only.
- Do not rely on VMware saying the NIC is static vs DHCP.
  - The user observed DHCP flags can be unreliable.
  - Operator opt-in is the safety mechanism.
- Feature must be explicit opt-in.
- Prefer per-NIC opt-in.
- Running VMware VM should be allowed to open the wizard because `Guest.Net` data is best while VMware Tools is running.
- Current OVF/export and VDDK virt-v2v flows still require the source VM to be powered off before actual import/conversion.
- UI should allow opening the wizard while powered on, capture data, warn strongly, poll state, disable OK until powered off.
- API automation flow:
  - call discovery/list API while VM is running to capture NIC data,
  - shut down VM externally,
  - call `importVm` with captured `preservestaticipniclist` values.
- Powered-off single-NIC fallback from `Guest.IpStack` is acceptable.
- Powered-off multi-NIC fallback from `Guest.IpStack` should not guess IP-to-MAC mapping when `Guest.Net` is empty.

## Implemented Behavior

- VMware guest networking data is collected from vCenter/VMware Tools.
- Per NIC, CloudStack can carry:
  - MAC address,
  - IPv4 address,
  - prefix length,
  - gateway,
  - DNS servers,
  - static/DHCP flag where present, but code does not require static.
- UI exposes per-NIC checkboxes:
  - `Preserve detected IPv4 configuration via virt-v2v`
- UI sends selected NIC values through API parameter:
  - `preservestaticipniclist[n].nic`
  - `preservestaticipniclist[n].macaddress`
  - `preservestaticipniclist[n].ipaddress`
  - `preservestaticipniclist[n].gateway`
  - `preservestaticipniclist[n].prefixlength`
  - `preservestaticipniclist[n].dnsservers`
- Backend builds one virt-v2v `--mac ...:ip:...` argument per selected eligible NIC.
- Missing/incomplete data does not fail migration; NIC is skipped for static IPv4 preservation.
- Non-Windows VM is skipped for static IPv4 preservation.
- Common extra params path is used, so both OVF/export and VDDK conversion paths should receive the same virt-v2v args.
- Running VMware VM warning now renders with a blank line and explains that OK is disabled until powered off.
- UI polls VMware power state every 10 seconds and enables OK once powered off.

## Important Test VM Data From User

User tested with real Windows Server 2016 VM, VMware Tools installed.

VM: `i-2-545-VM`

Running VM with two NICs produced:

NIC 1:

- MAC: `02:01:01:06:00:23`
- IP: `10.1.1.15`
- Prefix: `24`
- Gateway: `10.1.1.1`
- DNS: `10.1.1.1`, `10.0.32.1`, `8.8.8.8`

NIC 2:

- MAC: `02:01:01:c0:00:01`
- IP: `192.168.0.114`
- Prefix: `24`
- Gateway: `10.1.1.1`
- DNS: `10.1.1.1`, `10.0.32.1`, `8.8.8.8`

UI correctly showed these values and per-NIC preservation checkboxes.

## Files Changed In Static IPv4 Branch

Key files touched by the static IPv4 work:

- `api/src/main/java/org/apache/cloudstack/api/ApiConstants.java`
- `api/src/main/java/org/apache/cloudstack/api/command/admin/vm/ImportVmCmd.java`
- `api/src/main/java/org/apache/cloudstack/api/response/NicResponse.java`
- `api/src/main/java/org/apache/cloudstack/vm/UnmanagedInstanceTO.java`
- `server/src/main/java/com/cloud/api/ApiResponseHelper.java`
- `server/src/main/java/org/apache/cloudstack/vm/UnmanagedVMsManagerImpl.java`
- `server/src/test/java/org/apache/cloudstack/vm/UnmanagedVMsManagerImplTest.java`
- `ui/public/locales/en.json`
- `ui/src/views/tools/ImportUnmanagedInstance.vue`
- `ui/src/views/tools/ManageInstances.vue`
- `vmware-base/src/main/java/com/cloud/hypervisor/vmware/util/VmwareHelper.java`
- `notes/windows-static-ip-migration-docs.md`

Note: `notes/windows-static-ip-migration-docs.md` was committed only to preserve documentation draft context. Before opening the Apache CloudStack code PR, consider removing this file from the code PR and moving its content to the separate `apache/cloudstack-documentation` repo.

## Validation Already Run

Targeted backend tests passed:

```text
mvn -pl server -am "-Dtest=UnmanagedVMsManagerImplTest#testBuildVirtV2vStaticIpExtraParamsWindowsStaticNic+testBuildVirtV2vStaticIpExtraParamsMultipleNics+testBuildVirtV2vStaticIpExtraParamsDhcpNicAllowedWhenSelected+testBuildVirtV2vStaticIpExtraParamsMissingGatewaySkipped+testBuildVirtV2vStaticIpExtraParamsDnsOptional+testBuildVirtV2vStaticIpExtraParamsCapturedNicValues+testBuildVirtV2vStaticIpExtraParamsNonWindowsSkipped+testAppendVirtV2vStaticIpExtraParams" "-Dsurefire.failIfNoSpecifiedTests=false" test
```

Result: passed 8 tests.

UI lint passed:

```text
node ..\..\tools\node-v10.24.1-win-x64\node_modules\npm\bin\npm-cli.js run lint -- --no-fix
```

`vmware-base` compile passed:

```text
mvn -f vmware-base\pom.xml -DskipTests compile
```

`git diff --check` had passed earlier during the implementation phase.

## Local UI Setup

UI worktree:

```text
C:\Users\andri\Documents\Codex\2026-05-07\https-github-com-prashanthr2-vmware-to\cloudstack-next-import-prs\ui
```

Local UI URL:

```text
http://localhost:5050
```

`.env.local` was:

```text
CS_URL=http://10.0.34.215:8080
```

Node used:

```text
C:\Users\andri\Documents\Codex\2026-05-07\https-github-com-prashanthr2-vmware-to\tools\node-v10.24.1-win-x64
```

The dev server crashed once during hot reload with Node heap OOM. It was restarted with:

```powershell
$env:NODE_OPTIONS='--max_old_space_size=4096'
```

## Open Items / Not Done Yet

1. End-to-end real migration test:
   - UI capture worked.
   - Need confirm migrated Windows VM actually boots with intended IPv4 config after virt-v2v.

2. Missing `Allow duplicate MAC addresses` toggle in user test UI:
   - Local UI can show it only if remote API metadata exposes `allowduplicatemacaddresses`.
   - User saw only `Generate new MAC if required`.
   - Likely remote management server was not running backend JAR with PR `#13128` / duplicate-MAC API parameter.

3. Selective NIC import:
   - User wanted checkboxes next to NIC rows to import only a subset of source NICs.
   - This is NOT just UI.
   - Backend currently expects every source NIC to have a network mapping and imports all NICs.
   - Supporting selected NIC import would require backend filtering/validation/import changes.
   - Treat as separate follow-up unless user insists.

4. UI form is large:
   - User noted the modal is huge even on 2560x1440.
   - No full redesign was done.
   - Only warning text was patched.

5. Documentation PR:
   - Needs separate PR in `apache/cloudstack-documentation`.
   - Draft wording exists in `notes/windows-static-ip-migration-docs.md`.
   - Documentation should explain:
     - VMware Tools should be installed/running for best detection,
     - discover/capture while VM is running when possible,
     - shut down gracefully before import,
     - static IPv4 preservation is explicit opt-in per NIC,
     - CloudStack does not inspect disks,
     - powered-off single-NIC fallback may work, but multi-NIC powered-off mapping may not be possible without `Guest.Net`.

## Potential PR Cleanup Before Opening Static IPv4 PR

- Remove `notes/windows-static-ip-migration-docs.md` from CloudStack code PR or move to documentation repo.
- Rebase/stack decision:
  - Current branch includes duplicate-MAC commit plus static IPv4 commit.
  - Duplicate-MAC work already exists as PR `#13128` from branch `allow-duplicate-mac-import`.
  - Static IPv4 PR can be opened as stacked on top of duplicate-MAC work, or rebased after `#13128` is merged.
- If opening before `#13128` is merged, make it clear it depends on `#13128`.

## User Preferences / Tone / Constraints

- User wants English now.
- Do not mention AI/Codex in PRs or commits unless explicitly asked.
- User is comfortable with blunt wording but wants practical, direct answers.
- User is not a programmer and often asks for step-by-step instructions.
- User prefers saving work to fork before deleting local folders.

