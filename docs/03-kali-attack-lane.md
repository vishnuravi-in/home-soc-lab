# 03 — Kali Attacker VM & ATTACK Segment Rules

## VM Specification
| Setting | Value |
|---------|-------|
| Source | Official Kali Linux pre-built VirtualBox image |
| vCPU / RAM | 2 / 2048 MB |
| Network | Internal Network `LAN-Attack` |
| Address | `192.168.20.180/24` (DHCP from OPNsense Dnsmasq) |
| MAC | `08:00:27:c3:fb:76` |

## OPNsense Initial Configuration (Setup Wizard)
| Setting | Value | Rationale |
|---------|-------|-----------|
| Hostname / Domain | `fw01.soclab.internal` | `.internal` is reserved for private use |
| DNS | `1.1.1.1`, `9.9.9.9` | Quad9 adds malicious-domain filtering |
| Resolver | Unbound with DNSSEC | Local DNS resolution + logging |
| Timezone | Europe/London, default NTP | Accurate timestamps for log correlation |
| WAN: Block RFC1918 | Disabled | WAN upstream (10.0.2.0/24) is itself private |
| OPT1 description | `ATTACK` | Readable rule tabs and logs |

## ATTACK Interface Firewall Policy
Rules are evaluated top to bottom; first match wins. All rules log.

| # | Action | Protocol | Source | Destination | Port | Description |
|---|--------|----------|--------|-------------|------|-------------|
| 1 | Pass | TCP/UDP | ATTACK net | ATTACK address | 53 | Allow DNS to firewall |
| 2 | Block | any | ATTACK net | This Firewall | any | Block attacker to firewall |
| 3 | Pass | any | ATTACK net | any | any | Allow attacker outbound |

**Design intent:** the attacker can resolve DNS and reach the corporate segment and
the internet (required for simulations), but cannot reach the firewall's
management plane. Rule 2 uses **Block** (silent drop) rather than **Reject**, so the
firewall gives no confirmation of its presence to an adversary.

DHCP is permitted by OPNsense's automatic rules for interfaces served by Dnsmasq;
no manual rule was added (least privilege). See
[TS-001](troubleshooting/TS-001-dhcp-attack-segment.md).

## Policy Validation
Firewall decisions observed live on the console via the pf log interface:
```sh
tcpdump -n -e -i pflog0
```

| Test (from Kali) | Predicted | Actual | Matching rule |
|------------------|-----------|--------|---------------|
| `ping -c 3 8.8.8.8` | Pass | _fill in_ | Rule 3 |
| `ping -c 3 192.168.20.1` | Block (100% loss) | _fill in_ | Rule 2 |
| `curl -k --max-time 10 https://192.168.20.1` | Block (timeout) | _fill in_ | Rule 2 |

Snapshot: `02-attack-rules-tested`

## Lessons Learned
- A single client VM cannot test one segment and manage the firewall from another;
  console tools (`tcpdump`, `pflog0`) bridge the gap until a management host exists.
- Silent drops (Block) leak less information to an attacker than Reject.
