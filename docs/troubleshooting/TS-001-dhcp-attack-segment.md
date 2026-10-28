# TS-001 — Attacker VM Received No IPv4 Address on ATTACK Segment

| Field | Detail |
|-------|--------|
| **Date** | 2026-10-27 |
| **Lab phase** | Phase 2 — Kali deployment & ATTACK segment |
| **Severity** | Medium (blocked phase progress; no data loss) |
| **Status** | ✅ Resolved |
| **Affected systems** | `fw01` (OPNsense), `kali` (attacker VM) |
| **Time to resolve** | ~40 minutes |

---

## 1. Summary
After moving the Kali VM from the `LAN-Corp` segment to the `LAN-Attack` segment,
Kali failed to obtain an IPv4 address. Packet capture on the firewall proved that
Kali's DHCP requests were reaching the firewall's ATTACK interface, and that the
DHCP service (Dnsmasq) was running, but no replies were sent.

**Root cause:** Dnsmasq's DHCP service was not configured to serve the ATTACK
interface. Although a DHCP range for ATTACK existed, the interface was missing
from Dnsmasq's interface list, so requests arriving on `em2` were ignored.

**Fix:** Added the ATTACK interface to *Services → Dnsmasq DNS & DHCP → General →
Interface*. Kali immediately received `192.168.20.180/24`.

---

## 2. Environment
| Component | Detail |
|-----------|--------|
| Firewall | OPNsense `fw01`, ATTACK interface = `em2` / `opt1`, `192.168.20.1/24` |
| DHCP service | Dnsmasq DNS & DHCP (OPNsense default) |
| Client | Kali Linux, `eth0`, MAC `08:00:27:c3:fb:76` |
| Network | VirtualBox Internal Network `LAN-Attack` |

```
Kali (eth0) ──── LAN-Attack ──── em2 (ATTACK) fw01 ──── Dnsmasq :67
```

---

## 3. Symptoms
`ip a` on Kali showed the interface up, but with no IPv4 address:

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP
    link/ether 08:00:27:c3:fb:76 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::4b64:1025:f88f:2028/64 scope link noprefixroute
```

| Evidence | Interpretation |
|----------|----------------|
| `UP,LOWER_UP`, `state UP` | Virtual NIC enabled and cable connected — Layer 1 OK |
| Only `inet6 fe80::` (link-local) | Self-assigned address only |
| No `inet 192.168.20.x` line | **No DHCP lease received** |

---

## 4. Investigation Timeline

### Step 1 — Renew the DHCP lease (Kali)
**Hypothesis:** Kali did not request a new address after its adapter was moved
while the VM was running.
```bash
sudo nmcli networking off && sudo nmcli networking on
ip a
```
**Result:** ❌ Still no IPv4 address. Hypothesis rejected.

---

### Step 2 — Manual address + ARP test (Kali)
**Hypothesis:** Determine whether Kali is on the same Layer 2 segment as the
firewall, independent of DHCP.
```bash
sudo ip addr add 192.168.20.50/24 dev eth0
ping -c 2 192.168.20.1
ip neigh
```
**Result:** ❌ `ping: connect: Network is unreachable`

**Analysis:** This error is generated locally by Kali before any packet is sent —
it means the host has no route to the destination. The manually added address did
not persist (likely cleared by NetworkManager during its DHCP retries). The test
was inconclusive.

Supporting checks suggested:
```bash
nmcli networking          # confirm networking is enabled
ip a show eth0            # confirm address applied
ip route                  # confirm 192.168.20.0/24 route exists
```

> **Lesson:** Distinguish ping errors —
> `Network is unreachable` = local host has no address/route ·
> `Destination Host Unreachable` = no response on local segment ·
> 100% packet loss = dropped further along (often a firewall).

---

### Step 3 — Inspect DHCP configuration (OPNsense GUI)
**Hypothesis:** No DHCP range defined for the ATTACK interface.

Kali was temporarily moved back to `LAN-Corp` to reach the dashboard.
*Services → Dnsmasq DNS & DHCP → DHCP ranges:*

| Interface | Start | End |
|-----------|-------|-------|
| ATTACK | 192.168.20.100 | 192.168.20.200 |
| LAN | 192.168.10.41 | 192.168.10.245 |

**Result:** ❌ ATTACK range present. Hypothesis rejected.

**Side finding:** The LAN range (`.41–.245`) differs from the planned
`.100–.200`. The setup wizard appears to have replaced console-configured values
with defaults. Logged for later correction.

---

### Step 4 — Firewall Live View
**Hypothesis:** The default-deny policy on ATTACK (no rules configured) is
blocking DHCP requests.

*Firewall → Log Files → Live View* showed only outbound NTP from the firewall:
```text
WAN  Out  UDP  10.0.2.15:123 → <public NTP servers>:123  pass
     "let out anything from firewall host itself (force gw)"
```
**Result:** ⚠️ Inconclusive — no ATTACK traffic visible because Kali had to be on
`LAN-Corp` to view the dashboard. A single client VM cannot test the ATTACK
segment and view the GUI simultaneously. Test method changed.

---

### Step 5 — Packet capture on the firewall (OPNsense console)
**Hypothesis:** Determine whether DHCP requests physically reach `em2`.
`tcpdump` observes packets **before** firewall rules are applied.

Console → Option 8 (Shell):
```sh
tcpdump -ni em2 -e port 67 or port 68
```
Kali (on `LAN-Attack`):
```bash
sudo nmcli networking off && sudo nmcli networking on
```
**Result:** ✅ Requests arriving — ❌ no replies:
```text
23:23:41.976946 08:00:27:c3:fb:76 > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800),
  length 330: 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from 08:00:27:c3:fb:76
23:23:52.401000 ... Request from 08:00:27:c3:fb:76
23:23:54.400657 ... Request from 08:00:27:c3:fb:76
23:23:56.586082 ... Request from 08:00:27:c3:fb:76
23:24:01.306157 ... Request from 08:00:27:c3:fb:76
23:24:10.250152 ... Request from 08:00:27:c3:fb:76
11 packets captured
```

| Observation | Conclusion |
|-------------|------------|
| Source MAC `08:00:27:c3:fb:76` matches Kali's `eth0` | Kali is on the correct segment — **network naming/wiring ruled out** |
| `0.0.0.0:68 → 255.255.255.255:67` | Standard DHCP broadcast request |
| Repeated requests with increasing intervals | Client retrying with back-off |
| No reply from `192.168.20.1` | **Firewall receives but does not answer** |

Remaining suspects narrowed to: (a) firewall filtering, (b) DHCP service not
serving this interface.

---

### Step 6 — Confirm DHCP service is listening (OPNsense console)
```sh
sockstat -41 | grep ':67'
```
**Result:** ❌ `sockstat: illegal option -- 1`
Typo: digit `1` entered instead of lowercase `l` (visually identical in the
console font). Re-run with an unambiguous command:
```sh
sockstat | grep ':67'
```
**Result:** ✅
```text
nobody  dnsmasq  93528  4  udp4  *:67
```

| Field | Meaning |
|-------|---------|
| `nobody` | Service runs unprivileged (good security practice) |
| `dnsmasq` / PID `93528` | DHCP service is running |
| `udp4 *:67` | Listening on port 67, all interfaces |

**Conclusion:** Service running, requests arriving, but not answered → the
service must be **ignoring** the ATTACK interface at application level.

---

### Step 7 — Fix applied (OPNsense GUI)
*Services → Dnsmasq DNS & DHCP → General → Interface:* added **ATTACK**
alongside LAN → Save → Apply.

A manual firewall rule (pass UDP 68 → 67) was prepared but **not required**:
OPNsense automatically permits DHCP on interfaces served by its DHCP service.
The rule was therefore not added, in line with least privilege.

---

## 5. Verification
Kali moved back to `LAN-Attack`:
```bash
ip a
```
```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ... state UP
    link/ether 08:00:27:c3:fb:76 brd ff:ff:ff:ff:ff:ff
    inet 192.168.20.180/24 brd 192.168.20.255 scope global dynamic noprefixroute eth0
       valid_lft 86385sec preferred_lft 86385sec
```

| Evidence | Meaning |
|----------|---------|
| `inet 192.168.20.180/24` | Address within ATTACK DHCP range ✅ |
| `scope global dynamic` | Assigned by DHCP ✅ |
| `valid_lft 86385sec` | ~24-hour lease |

Note: Dnsmasq assigns addresses based on a hash of the client MAC rather than
sequentially, so `.180` rather than `.100` is expected behaviour.

Snapshot taken on both VMs: `02-dhcp-attack-working`.

---

## 6. Root Cause Analysis
```
Kali DHCP request ✅ → reaches em2 ✅ → Dnsmasq interface filter ❌ → Dnsmasq listening ✅
```
The DHCP range for ATTACK was defined, but the ATTACK interface was not included
in Dnsmasq's interface list. Dnsmasq listens on `*:67` but discards requests from
interfaces it is not configured to serve, so no error was visible in the GUI.

---

## 7. Commands Reference
| Where | Command | Purpose |
|-------|---------|---------|
| Kali | `ip a` / `ip a show eth0` | Show interface addresses |
| Kali | `ip route` | Show routing table |
| Kali | `ip neigh` | Show ARP/neighbour table (Layer 2 reachability) |
| Kali | `nmcli networking` | Check NetworkManager state |
| Kali | `sudo nmcli networking off && sudo nmcli networking on` | Force DHCP renewal |
| Kali | `sudo ip addr add 192.168.20.50/24 dev eth0` | Temporary static address |
| Kali | `ping -c 2 192.168.20.1` | Reachability test |
| fw01 | `tcpdump -ni em2 -e port 67 or port 68` | Capture DHCP traffic on ATTACK NIC |
| fw01 | `sockstat \| grep ':67'` | Identify process listening on DHCP port |

---

## 8. Lessons Learned
1. **Test from the other end.** When client-side tests are inconclusive, inspect
   the server side.
2. **Packet capture is ground truth.** `tcpdump` proved the network path was
   working and isolated the fault to the service configuration.
3. **Diagnose by elimination.** Each step ruled out one layer: wiring → DHCP
   range → network path → service process → service configuration.
4. **A running service is not necessarily a serving service.** Listening on
   `*:67` did not mean the ATTACK interface was being served.
5. **Know your tooling limits.** One client VM cannot both test a segment and
   view the management GUI on another — a dedicated management host is needed.
6. **Watch for lookalike characters** (`1` vs `l`) in console fonts.
7. **Don't add unnecessary rules.** The prepared firewall rule was not needed and
   was not applied.

## 9. Follow-up Actions
- [ ] Correct LAN DHCP range from `.41–.245` to `.100–.200`
- [ ] Add ATTACK firewall rules (DNS allow, block firewall, allow any) — Phase 2 Part C
- [ ] Deploy Windows VM on `LAN-Corp` as permanent management host (Phase 3)
