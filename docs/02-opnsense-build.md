# 02 — OPNsense Firewall Build

## VM Specification
| Setting | Value |
|---------|-------|
| Guest OS | FreeBSD (64-bit) |
| vCPU / RAM | 1 / 1536 MB |
| Disk | 20 GB VDI, dynamically allocated |
| Audio / USB | Disabled (attack surface reduction) |

## Network Adapters
| VirtualBox Adapter | Attached To | OPNsense NIC | Role | Address |
|--------------------|-------------|--------------|------|---------|
| Adapter 1 | NAT | `em0` | WAN | 10.0.2.15/24 (DHCP) |
| Adapter 2 | Internal: `LAN-Corp` | `em1` | LAN | 192.168.10.1/24 |
| Adapter 3 | Internal: `LAN-Attack` | `em2` | OPT1 | 192.168.20.1/24 |

## Build Steps
1. Verified installer checksum (SHA256) against the vendor-published value
2. Installed via console installer (UFS filesystem, UK keymap)
3. Set boot order to **Hard Disk first** to prevent re-booting into the installer
4. Assigned interfaces (console Option 1): WAN `em0`, LAN `em1`, OPT1 `em2`
5. Configured static IPs and DHCP scopes (console Option 2)
6. Verified outbound connectivity via console ping to `8.8.8.8`
7. Snapshot taken: `01-interfaces-assigned`

## Troubleshooting Log
| Issue | Root Cause | Resolution |
|-------|------------|------------|
| VM rebooted into installer | ISO still attached after install | Ejected ISO; set boot order to Hard Disk first |
| `em1` reported as invalid interface | VirtualBox Adapters 2/3 not enabled | Enabled adapters, attached to Internal Networks |
| Root password rejected | Keyboard layout mismatch between installer and console | Verified typed characters at login prompt; re-entered accordingly |
| "Same interface name" error | `em0` re-entered at LAN prompt | Matched prompts by label: LAN = `em1` |
| DHCP range rejected | Partial address (`.100`) entered | Entered full addresses within interface subnet |
| Kali received no IPv4 address on ATTACK segment | Dnsmasq DHCP not serving the ATTACK interface | Proved request arrival with `tcpdump`, confirmed service with `sockstat`, added ATTACK to Dnsmasq interface list — see [TS-001](troubleshooting/TS-001-dhcp-attack-segment.md) |

## Lessons Learned
- Read the evidence surrounding an error (e.g., the detected interface list), not
  only the error itself.
- Keyboard layout mismatches can produce authentication failures that resemble
  password attacks in logs — rule out benign causes before escalating.
