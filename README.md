# 🏠 Homelab Infrastructure

Fully segmented home network built as a hands-on security lab. Defense-in-depth architecture with 7 isolated zones, real-time threat detection and centralized monitoring.

## 🏗️ Architecture
```text
┌─────────────────────────────────────────────────────┐
│                    OPNsense Firewall                 │
│              (Topton N355 - 6 NIC ports)             │
├──────┬──────┬──────┬──────┬──────┬──────┬───────────┤
│ WAN  │ LAN  │ IoT  │Guest │ Mgmt │ VPN  │ Monitor   │
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

- **Firewall**: OPNsense with per-zone rule sets
- **IDS/IPS**: Suricata with ET Open + custom rules
- **VPN**: WireGuard with kill switch, Mullvad integration
- **DNS**: Unbound with DNS-over-TLS, DNSSEC validation, blocklists
- **Hardening**: Automated scripts with rollback capability

## 📊 Monitoring

- **Telegraf** → **InfluxDB** → **Grafana** (TIG stack on Raspberry Pi)
- Centralized syslog from all network devices
- Custom dashboards for traffic, threats, DNS queries
- Alerting on anomalous patterns

## 🧠 Design Decisions

Every design choice is based on risk analysis:
- Smart home devices are isolated because they phone home to unknown servers
- Management zone has separate access path in case VPN tunnel fails
- Monitoring is receive-only to prevent lateral movement from compromised dashboards
- Guest network has no visibility into any internal zone

## 📝 Lessons Learned

- ISP fiber connection quirks prevented Suricata on WAN interface — documented and moved to internal segment
- NIC compatibility issues with security tooling — solved with automated driver configuration
- DNS blocklists revealed surprising amount of IoT telemetry traffic

## 🔗 Links

- [Full write-up on my website](https://bastiaan365.com)
- [My Grafana dashboards](https://github.com/bastiaan365/grafana-dashboards)
- [Hardening scripts](https://github.com/bastiaan365/ubuntu-hardening-scripts)

---

*This is a documentation repository. Sensitive configuration details (IPs, hostnames, credentials) are anonymized or replaced with placeholders.*
