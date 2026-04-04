# 🏠 Homelab Infrastructure

My home network runs as a proper security lab. Started building this when I was at KLM and wanted to understand the enterprise tooling I was deploying on 15,000+ endpoints — but from the ground up, not just clicking through a GUI. Running it since 2023, gradually replaced most components as I learned what actually mattered.

The short version: OPNsense as the firewall, 7 VLANs to isolate everything from each other, Suricata watching traffic in real time, WireGuard for remote access, and a Raspberry Pi running the TIG stack for metrics. Not because it's overkill — because most serious home security problems come from trusting your own internal network too much.

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                   OPNsense Firewall                   │
│              (Topton N355 - 6 NIC ports)             │
├──────┬──────┬──────┬──────┬──────┬──────┬───────────┤
│ WAN  │ LAN  │ IoT  │Guest │ Mgmt │ VPN  │  Monitor  │
│      │Trust │Isol. │Only  │Admin │Tunnel│ TIG Stack │
└──────┴──────┴──────┴──────┴──────┴──────┴───────────┘
```

## 🔒 Network Segments

| Zone | Purpose | Internet | Cross-zone |
|---|---|---|---|
| LAN | Trusted devices, workstations | ✅ Via VPN | Limited |
| IoT | Smart home devices | ✅ Filtered | ❌ Blocked |
| Guest | Visitor WiFi | ✅ Direct | ❌ Blocked |
| Management | Admin access, OPNsense GUI | ✅ Via VPN | ✅ All zones |
| VPN | WireGuard tunnel (Mullvad) | ✅ Encrypted | ❌ Blocked |
| Monitoring | TIG stack, syslog | ❌ None | 📥 Receive only |
| Home Assistant | Smart home automation | ✅ Filtered | 📤 IoT only |

## 🛡️ Security Stack

- **Firewall:** OPNsense with per-zone rule sets — rules are explicit, no implicit trust
- **IDS/IPS:** Suricata with ET Open + custom rules for IoT-specific patterns
- **VPN:** WireGuard with kill switch enabled, routing through Mullvad
- **DNS:** Unbound with DNS-over-TLS, DNSSEC, and blocklists covering ~280k domains
- **Hardening:** Automated scripts with rollback capability (see [ubuntu-hardening-scripts](https://github.com/bastiaan365/ubuntu-hardening-scripts))

## 📊 Monitoring

Telegraf → InfluxDB → Grafana on a Raspberry Pi 5 (8GB). Collects from SNMP, syslog, and OPNsense API. Alerting on anomalous patterns — mostly useful for catching IoT devices doing things they shouldn't.

See [grafana-dashboards](https://github.com/bastiaan365/grafana-dashboards) for the actual dashboard configs.

## 🧠 Design Decisions

A few things I spent longer than expected figuring out:

**Why 7 VLANs?** I started with 3 and kept hitting cases where trust boundaries were wrong. Smart home devices in particular need their own zone — they phone home to servers I can't verify, and I don't want that traffic anywhere near my workstations.

**Why WireGuard over OpenVPN?** Speed and simplicity. WireGuard's codebase is dramatically smaller (around 4000 lines vs OpenVPN's hundreds of thousands), which means less attack surface. Mullvad supports it natively and doesn't log.

**Monitoring zone is receive-only.** If the Grafana instance gets compromised, it can't reach anything useful. Learned this pattern from reading about defense-in-depth in enterprise networks — applies just as well at home.

**Suricata ended up on internal rather than WAN.** My ISP fiber connection has quirks that prevented reliable WAN packet inspection. Moving it to the internal interface still catches lateral movement and C2 beaconing, which is most of what I care about anyway.

## 📝 Lessons Learned

The DNS blocklist revealed how chatty IoT devices really are. Some devices were making DNS queries every 30 seconds to telemetry endpoints. That's now blocked network-wide, and I can see in Grafana exactly what's getting blocked and when.

The Management VLAN having a separate access path from the VPN tunnel is intentional — if the VPN breaks, I still need to reach OPNsense from my workstation to fix it. Simple thing, but I didn't think about it until I locked myself out the first time.

## 🔗 Related

- Full write-up at [bastiaan365.com](https://bastiaan365.com)
- [Grafana dashboards](https://github.com/bastiaan365/grafana-dashboards) — visualising everything above
- [DNS security setup](https://github.com/bastiaan365/dns-security-setup) — the Unbound config
- [Ubuntu hardening scripts](https://github.com/bastiaan365/ubuntu-hardening-scripts) — for the Linux nodes

> All sensitive config details (IPs, hostnames, MAC addresses) are anonymized or replaced with placeholders throughout this repository.
