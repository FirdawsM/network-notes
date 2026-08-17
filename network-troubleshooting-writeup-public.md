# Incident Notes: Office WiFi Failure Due to Subnet Collision & Stale NAT Rules

**Date:** August 2026
**System:** Ubuntu Desktop laptop
**Network:** Corporate WiFi (name withheld)
**Symptom:** Connected to WiFi with a valid IP, but zero internet access. Everyone else on the same network was unaffected.

---

## Summary

My laptop showed as "Connected" to the office WiFi network but couldn't reach the internet. Every other device on the same network worked fine, which pointed to a local, machine-specific misconfiguration rather than an ISP or office network problem.

Root cause: **A KVM/libvirt virtual network (`net-wan`) was configured on the exact same private subnet as the office WiFi (`192.168.X.0/24`)**. Even though the virtual interface itself was down, its leftover `iptables` NAT/MASQUERADE rules were still active and matched traffic by subnet — not by interface — so they intercepted and mishandled real WiFi traffic meant for the internet.

---

## Troubleshooting Process (Bottom-Up / OSI-Aligned)

Standard IT troubleshooting order, from the bottom of the stack up:

| Step | OSI Layer | Command | Result |
|---|---|---|---|
| 1. Is the interface up? | Physical / Data Link (L1–L2) | `ip link` | WiFi interface (`wlan0`) was up |
| 2. Do I have a valid IP? | Network (L3) | `ip addr show` | Valid IP assigned via DHCP on the `192.168.X.0/24` range |
| 3. Can I reach the gateway? | Network (L3) | `ping <gateway-ip>` | Success — 0% packet loss |
| 4. Can I reach the internet by IP? | Network (L3) | `ping 8.8.8.8` | **Failed — 100% packet loss** |
| 5. Can I resolve DNS? | Application (L7)-ish | `ping google.com` | Failed — `Name or service not known` |
| 6. Check for local firewall | Network (L3) | `sudo ufw status` | Inactive — ruled out |
| 7. Check NAT/firewall rules by subnet | Network (L3) | `sudo iptables -t nat -L -n -v \| grep 192.168.X` | **Found MASQUERADE + ACCEPT rules tied to a stale virtual bridge on the same subnet as my real WiFi** |

Since the gateway responded but nothing beyond it did, the problem was isolated to something between my laptop and the wider internet — specifically routing/NAT, not DNS and not the WiFi link itself.

---

## Root Cause Detail

- KVM/libvirt had created a virtual network (`net-wan`) using the same `/24` subnet as the real office WiFi.
- This created a **duplicate route** in the routing table for that subnet — one via the real WiFi adapter, one via the virtual bridge.
- Bringing the virtual bridge interface down (`ip link set <bridge> down`) stopped the interface but **did not remove the associated iptables NAT rules**, which still matched traffic by IP range regardless of interface state.
- Those rules were masquerading (rewriting) traffic on that subnet, corrupting real outbound packets before they could properly reach the internet.

---

## Fix

### Immediate fix (stop the collision)

```bash
# Identify which KVM virtual network owns the conflicting subnet
sudo virsh net-dumpxml net-wan | grep -A2 "ip address"

# Destroy the conflicting virtual network (removes both the interface and its iptables rules)
sudo virsh net-destroy net-wan

# Confirm the stale rules are gone
sudo iptables -t nat -L -n -v | grep 192.168.X
# (should return nothing)

# Confirm internet access is restored
ping -c 4 8.8.8.8
curl -I https://google.com

# Temporarily prevent this network from auto-starting on boot
sudo virsh net-autostart net-wan --disable
```

This got internet working again immediately, but only by keeping the conflicting lab network turned off — the underlying collision (same subnet used by both the lab and real WiFi) was still there and would recur any time the lab network was started while on that WiFi.

### Permanent fix (eliminate the collision entirely)

Rather than manually remembering to disconnect WiFi or keep the lab network off every time, the subnet itself was reassigned to a range that will never collide with a real-world router:

```bash
# Edit the virtual network's IP range
sudo virsh net-edit net-wan
```

Changed from:
```xml
<ip address='192.168.100.1' netmask='255.255.255.0'>
  <dhcp>
    <range start='192.168.100.128' end='192.168.100.254'/>
  </dhcp>
</ip>
```

To an unused, non-default range:
```xml
<ip address='192.168.99.1' netmask='255.255.255.0'>
  <dhcp>
    <range start='192.168.99.128' end='192.168.99.254'/>
  </dhcp>
</ip>
```

```bash
# Apply the change
sudo virsh net-start net-wan

# Confirm the new range is active
sudo virsh net-dumpxml net-wan | grep -A2 "ip address"

# Verified: internet stays up even with the lab network active
ping -c 4 8.8.8.8

# Safe to re-enable autostart now that the collision is structurally impossible
sudo virsh net-autostart net-wan
```

**Result:** confirmed via successful `ping -c 4 8.8.8.8` (0% packet loss) *while the lab network was active* — the two networks no longer overlap, so autostart could be safely re-enabled.

**Note:** The lab's own configuration (its VMs, services, and any pfSense/firewall rules referencing the old `192.168.100.x` range) was not touched by this fix and may need separate updating wherever that old range was hardcoded — this network reassignment only changed the KVM network's IP scheme, not the lab's internal setup.

---

## Status: Resolved

Both the immediate symptom (no internet) and the root cause (overlapping subnets) are fixed. The lab network can now run alongside real WiFi with autostart re-enabled, with no further conflict expected. Any lab-internal references to the old subnet (e.g. pfSense WAN config) are a separate follow-up, unrelated to host connectivity.

## Key Lesson

> A virtual interface being **down** does not mean its **firewall/NAT rules are gone**. iptables rules match by IP range and interface name independently of link state — they persist until the owning network is explicitly destroyed (`virsh net-destroy`), not just disabled at the link layer.

This is a good example of why grepping `iptables -t nat -L` by subnet is a standard move once basic connectivity (interface, IP, gateway) checks out but internet access still fails.

---

## OSI Model Reference (Full 7-Layer, for notes)

| Layer | Name | Relevant to this incident? |
|---|---|---|
| 7 | Application | DNS resolution, HTTP request (`curl`) |
| 6 | Presentation | Not directly involved |
| 5 | Session | Not directly involved |
| 4 | Transport | TCP handshake to reach google.com over port 443 |
| 3 | Network | **Primary layer of the issue** — IP routing, NAT, iptables |
| 2 | Data Link | Duplicate interface contributing to the collision |
| 1 | Physical | WiFi radio link — confirmed up early and ruled out |

Most real-world "no internet" issues live in Layers 1–4; Layers 5–7 rarely come into play unless the problem is application-specific (e.g., a broken TLS cert or a misbehaving app).

---

## Tools Used

- `ip addr show`, `ip link`, `ip route` — interface & routing inspection
- `ping` — reachability testing at each hop
- `nmcli` — WiFi connection management
- `iptables -t nat -L -n -v` — NAT/firewall rule inspection
- `virsh` (libvirt CLI) — KVM virtual network inspection and management
- `curl -I` — real-world HTTP-level connectivity confirmation

---

*Written up for personal reference / portfolio. Feel free to reuse this troubleshooting checklist for similar "connected but no internet" issues.*
