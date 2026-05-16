# Windows Static IPv4 Preservation Documentation Notes

Use this wording for the follow-up `apache/cloudstack-documentation` PR covering VMware-to-KVM migration of Windows VMs with static IPv4 configuration.

Suggested content:

- For Windows VMs with static IPv4 configuration, install and run VMware Tools before migration.
- For best results, open the VMware-to-KVM import wizard while the source VM is still running, so CloudStack can detect per-NIC guest network information.
- Before starting the actual import/conversion, gracefully shut down the source VM.
- If the VM is already powered off, CloudStack may still preserve IPv4 configuration when reliable mapping is available, especially for single-NIC VMs.
- For powered-off multi-NIC VMs where VMware Tools no longer exposes per-NIC mapping, CloudStack will not guess IP-to-MAC mappings.

Important nuance:

- CloudStack does not prefer the VM to be running for the actual OVF/VDDK virt-v2v conversion. The VM must still be powered off before starting import.
- The VM being running is useful at discovery/wizard time only, because that is when VMware Tools can provide the reliable mapping:

```text
MAC address -> IPv4 address / prefix length / gateway / DNS servers
```

