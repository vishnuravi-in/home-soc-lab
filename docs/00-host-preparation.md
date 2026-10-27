# 00 — Host Preparation & Capacity Planning

## Hardware Assessment
| Component | Specification | Lab Impact |
|-----------|---------------|------------|
| Model | ASUS TUF Gaming F15 (FX506LH) | 2× SO-DIMM, 2× M.2 slots — upgradeable |
| CPU | Intel Core i5-10300H (4C/8T) | VT-x and VT-d supported |
| RAM | 8 GB DDR4 (1 of 2 slots used) | **Primary constraint** |
| Storage | 512 GB NVMe (C: 361 GB / D: 90 GB partitions) | Same physical disk |

## Pre-flight Checks
| Check | Command / Location | Result |
|-------|--------------------|--------|
| Virtualisation in firmware | Task Manager → Performance → CPU | ✅ Enabled |
| Hyper-V conflict | `systeminfo \| Select-String "Hyper-V\|hypervisor"` | ✅ Not active — VirtualBox runs natively |
| Physical disk layout | `Get-Partition` | C: and D: on one NVMe |

## Hypervisor Choice
**Oracle VirtualBox (Type 2)** with Extension Pack.
- Free and open source; already installed on host
- Internal Networks provide isolated segments without extra tooling
- Migration path: repurpose this laptop as a **Proxmox VE (Type 1)** server after a future hardware upgrade

## Storage Layout
```
C:\SOC-Lab\ISOs\    # Installation media (removable after build)
D:\SOC-Lab\VMs\     # Firewall, attacker and endpoint VMs
```
- VM disks are **dynamically allocated** to conserve space
- Wazuh SIEM will be placed on C: due to log growth; D: has ~86 GB free
- Snapshot retention: max 2–3 per VM

## Integrity Verification
All installation media verified against vendor-published SHA256 checksums:
```powershell
Get-FileHash "C:\SOC-Lab\ISOs\<file>" -Algorithm SHA256
```

## Capacity Plan
| VM | vCPU | RAM (now) | RAM (after upgrade) |
|----|------|-----------|---------------------|
| OPNsense | 1 | 1.5 GB | 2 GB |
| Kali Linux | 2 | 2 GB | 4 GB |
| Windows 11 + Sysmon | 2 | — | 4 GB |
| Wazuh all-in-one | 4 | — | 8 GB |

**Decision:** A RAM upgrade to 24 GB (adding 1× 16 GB DDR4-3200 SO-DIMM) is a
prerequisite for the SIEM tier. Firewall and attacker tiers are built first.
