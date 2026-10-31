# Firewall Practice Lab: OPNsense Rules, NAT and Log Analysis

By Siri Bandaru

## Overview

I built a small firewall lab to practise how a firewall decides what traffic to allow: zones, rule order, NAT and reading logs. The lab uses **OPNsense** (a free, open-source firewall), not Palo Alto. The same ideas apply to Palo Alto firewalls (trust and untrust zones, top-down policy, source NAT, traffic logs), and I am applying them there next.

**Skills practised:** firewall rule design, default deny, rule ordering, stateful inspection, source NAT, log analysis, troubleshooting.

## Lab setup

```
Internet -- VMware NAT -- [OPNsense firewall] -- LAN segment "trust" -- [Kali Linux client]
                           WAN = untrust          LAN = trust
```

| Component | Details |
| --- | --- |
| Hypervisor | VMware Workstation |
| Firewall | OPNsense VM, 2 GB RAM, 20 GB disk, two adapters |
| WAN (untrust) | Adapter 1 on VMware NAT |
| LAN (trust) | Adapter 2 on a private LAN segment named `trust`, 192.168.1.0/24 |
| Client | Kali Linux VM on the `trust` segment only, so all of its traffic passes through the firewall |

The client used a fixed address (192.168.1.50) with the firewall as its gateway and DNS server. The DHCP service did not start on my OPNsense version, so I did not use DHCP for this lab.

**Baseline:** before any drill, the client could ping the gateway and 8.8.8.8 and resolve google.com.

## Drill 1: Default deny

**Goal:** show that with no rule allowing traffic, everything is blocked.

**What I did:** disabled the default allow rule on the LAN interface, applied the change, and cleared the state table.

**What happened:** pings and name lookups from the client failed. Live View showed the client's traffic blocked.

**Evidence:**

- `Drill1-Firewall-rules-disabled.png` shows the LAN rules with the default allow disabled
- `Drill1-Firewall-Log-View-live.png` shows the blocked traffic in Live View

**Lesson:** a firewall with no allow rule blocks by default. Some green entries in the log belonged to the firewall's own traffic (a built-in rule for traffic from the firewall itself), so I filtered the log by the client's address, 192.168.1.50, to see only its traffic.

## Drill 2: Allow only DNS and HTTPS

**Goal:** allow only what is needed.

**What I did:** added two Pass rules on LAN with logging on: DNS (TCP/UDP port 53) and HTTPS (TCP port 443), both from LAN net to any.

**What happened:** `nslookup` and `curl -I https://google.com` worked. Ping to 8.8.8.8 kept working at first even though no rule allowed it. After I cleared the state table (`pfctl -F states`), ICMP was blocked as expected.

**Evidence:**

- `Drill2-Firewall-rules.png` shows the two new rules
- `Drill2-terminal-screenshots.png` shows the test commands
- `Drill2-Live-view-logs.png` shows allowed and blocked entries

**Lesson:** the firewall is stateful. A connection that was already allowed kept passing until its state was cleared, so after every rule change I applied the change and cleared the state table before testing.

## Drill 3: Rule order (shadowed rule)

**Goal:** show that rules are checked top to bottom and the first match wins.

**What I did:** added a Block rule for all traffic from LAN net and placed it above the two allow rules. Then I moved it below them.

**What happened:** with the block rule on top, DNS and HTTPS failed even though the allow rules existed. Live View named the block rule as the match. After moving it below the allows, DNS and HTTPS worked again.

**Evidence:**

- `Drill3-before-firewall-rules.png` and `Drill3-Live-view-before.png` and `Drill3-terminal-before.png` show the broken state (block rule on top)
- `Drill3-after-firewall-rules.png` and `Drill3-Live-view-after.png` and `Drill3-terminal-after.png` show the fixed state

**Lesson:** order matters. A broad deny above a specific allow makes the allow useless. I kept a logged block rule at the bottom as a cleanup rule.

## Drill 4: Source NAT

**Goal:** show what NAT does, and how a policy can allow traffic while the connection still breaks.

**What I did:** disabled outbound NAT in the Source NAT settings, tested the client, then switched it back to automatic, clearing the state table after each change.

**What happened:** with NAT disabled, Live View on LAN still showed the traffic as passed, which means the security policy was not the cause. The client's commands still worked. The most likely reason is that the lab's WAN sits behind VMware's own NAT, which translated the addresses a second time, so the missing firewall NAT did not break connectivity here. On a real internet-facing WAN, this configuration would fail, because replies cannot return to a private address. I switched outbound NAT back to automatic afterwards.

**Evidence:**

- `Drill4-NAT-disabled.png` shows outbound NAT disabled (the change), and `Drill4-NAT-enabled.png` shows it switched back to automatic (the fix)
- `Drill4-terminal-before.png` and `Drill4-terminal-after.png` show the client tests with NAT disabled, then after the fix
- `Drill4-Live-before.png` and `Drill4-Live-after.png` show the log with NAT disabled, then after the fix

**Lesson:** the security policy decides allow or deny, and NAT decides whether replies can find their way back to a private address. When the log says allowed but nothing works, check NAT and routing next.

## Other problems I fixed along the way

- **WAN lost its address:** the firewall had no gateway after VMware's virtual network changed. Fixed by restarting VMware's NAT and DHCP services and renewing the WAN lease.
- **DHCP service would not start:** the log showed a Dnsmasq error about reading the system DNS file. I switched to a fixed client address instead of spending the lab on it.
- **Client could not reach the firewall:** the firewall's LAN address had changed, so I was pinging an address nobody owned. Found by reading the LAN address on the firewall console.

## How this maps to Palo Alto

| This lab (OPNsense) | Palo Alto equivalent |
| --- | --- |
| LAN and WAN interfaces | Trust and untrust zones |
| Firewall rules, top to bottom | Security policy, first match wins |
| Block rule at the bottom with logging | Cleanup rule |
| Source NAT (outbound) | Source NAT policy |
| Live View | Traffic log |
| Port-based rules (53, 443) | App-ID rules (dns, ssl) |

## Limitations

This is a lab on OPNsense, not production experience and not Palo Alto. OPNsense matches by port, while a Palo Alto firewall identifies the application itself (App-ID). I have not used Panorama, GlobalProtect or high availability.

## Next steps

- Repeat the drills on a Palo Alto firewall lab and compare the two
- Study for the PCNSA certification
