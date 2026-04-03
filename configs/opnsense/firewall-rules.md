# OPNsense Firewall Rules

Documented ruleset for the homelab OPNsense firewall running on a Topton N355 appliance. Rules are organized by source zone. OPNsense applies rules on the **inbound interface** of each zone, evaluated top-to-bottom with first-match wins.

> **Note:** This is a documentation-only representation. The actual OPNsense XML configuration contains interface identifiers, internal UUIDs, and network details that are not published here.

## General Principles

- Default policy on every interface is **deny all** (implicit block at bottom).
- Anti-lockout rule is enabled only on the Management interface.
- Bogon and RFC 1918 filtering is enabled on WAN.
- All blocked traffic is logged. Passed traffic is logged selectively.
- Stateful inspection is active on all rules (default OPNsense behavior).

---

## LAN Zone Rules

Trusted workstations and personal devices. Internet access is routed through the VPN gateway.

| # | Source | Destination | Protocol | Port | Action | Rationale |
|---|--------|-------------|----------|------|--------|-----------|
| 1 | LAN net | VPN gateway | * | * | Pass | Route all internet traffic through WireGuard tunnel |
| 2 | LAN net | Monitoring net | TCP | 8086, 3000 | Pass | Allow access to InfluxDB and Grafana dashboards |
| 3 | LAN net | Monitoring net | UDP | 514 | Pass | Send syslog data to monitoring stack |
| 4 | LAN net | Management net | TCP | 443 | Pass | Access OPNsense web GUI from trusted devices |
| 5 | LAN net | Home Assistant net | TCP | 8123 | Pass | Access Home Assistant dashboard |
| 6 | LAN net | IoT net | * | * | Block | Prevent lateral movement into IoT segment |
| 7 | LAN net | Guest net | * | * | Block | No access to guest network |
| 8 | LAN net | * | * | * | Block | Default deny (implicit) |

## IoT Zone Rules

Smart home devices (cameras, sensors, smart plugs). Heavily restricted.

| # | Source | Destination | Protocol | Port | Action | Rationale |
|---|--------|-------------|----------|------|--------|-----------|
| 1 | IoT net | Home Assistant net | TCP | 8123, 1883 | Pass | Allow devices to reach Home Assistant and MQTT broker |
| 2 | IoT net | Monitoring net | UDP | 514 | Pass | Forward syslog to monitoring |
| 3 | IoT net | DNS server (Unbound) | TCP/UDP | 53 | Pass | Allow DNS resolution via internal resolver only |
| 4 | IoT net | WAN | TCP | 443 | Pass | Allow HTTPS for firmware updates (filtered by alias) |
| 5 | IoT net | WAN | TCP | 80 | Block | Block unencrypted outbound HTTP |
| 6 | IoT net | RFC1918 | * | * | Block | Prevent access to any internal network |
| 7 | IoT net | * | * | * | Block | Default deny (implicit) |

## Guest Zone Rules

Visitor Wi-Fi. Internet-only access with no visibility into internal networks.

| # | Source | Destination | Protocol | Port | Action | Rationale |
|---|--------|-------------|----------|------|--------|-----------|
| 1 | Guest net | DNS server (Unbound) | TCP/UDP | 53 | Pass | DNS resolution through internal resolver with blocklists |
| 2 | Guest net | WAN | TCP | 80, 443 | Pass | Basic web browsing |
| 3 | Guest net | WAN | UDP | 443 | Pass | QUIC / HTTP/3 support |
| 4 | Guest net | RFC1918 | * | * | Block | Prevent access to all internal networks |
| 5 | Guest net | * | * | * | Block | Default deny (implicit) |

## Management Zone Rules

Administrative access to network infrastructure. Highest privilege zone.

| # | Source | Destination | Protocol | Port | Action | Rationale |
|---|--------|-------------|----------|------|--------|-----------|
| 1 | Management net | This firewall | TCP | 443, 22 | Pass | Anti-lockout: GUI and SSH access to OPNsense |
| 2 | Management net | LAN net | TCP | 22 | Pass | SSH into trusted workstations for maintenance |
| 3 | Management net | IoT net | TCP | 22, 443 | Pass | Manage IoT devices (firmware, config) |
| 4 | Management net | Monitoring net | TCP | 3000, 8086, 22 | Pass | Access Grafana, InfluxDB, and SSH to Pi |
| 5 | Management net | Home Assistant net | TCP | 8123, 22 | Pass | Manage Home Assistant instance |
| 6 | Management net | VPN gateway | * | * | Pass | Internet access through VPN |
| 7 | Management net | WAN | TCP | 443 | Pass | Fallback direct internet if VPN is down |
| 8 | Management net | * | * | * | Block | Default deny (implicit) |

## VPN Zone Rules

WireGuard tunnel interface (Mullvad). Traffic arriving from the tunnel endpoint.

| # | Source | Destination | Protocol | Port | Action | Rationale |
|---|--------|-------------|----------|------|--------|-----------|
| 1 | VPN net | WAN | * | * | Pass | Outbound traffic exits through VPN tunnel |
| 2 | VPN net | RFC1918 | * | * | Block | VPN return traffic must not reach internal nets |
| 3 | VPN net | * | * | * | Block | Default deny (implicit) |

## Monitoring Zone Rules

TIG stack (Telegraf, InfluxDB, Grafana) running on Raspberry Pi 5. Receive-only design.

| # | Source | Destination | Protocol | Port | Action | Rationale |
|---|--------|-------------|----------|------|--------|-----------|
| 1 | Monitoring net | DNS server (Unbound) | TCP/UDP | 53 | Pass | DNS resolution for NTP sync and package updates |
| 2 | Monitoring net | WAN | TCP | 443 | Pass | Package updates and Grafana plugin downloads |
| 3 | Monitoring net | WAN | UDP | 123 | Pass | NTP time synchronization |
| 4 | Monitoring net | RFC1918 | * | * | Block | No outbound access to internal networks |
| 5 | Monitoring net | * | * | * | Block | Default deny (implicit) |

> The monitoring zone is intentionally receive-only for internal traffic. All zones push logs and metrics **to** the monitoring net, but the monitoring net cannot initiate connections back. This limits blast radius if the monitoring host is compromised.

## Home Assistant Zone Rules

Smart home automation controller. Needs to communicate with IoT devices.

| # | Source | Destination | Protocol | Port | Action | Rationale |
|---|--------|-------------|----------|------|--------|-----------|
| 1 | Home Assistant net | IoT net | TCP/UDP | * | Pass | Control and query IoT devices |
| 2 | Home Assistant net | DNS server (Unbound) | TCP/UDP | 53 | Pass | DNS resolution |
| 3 | Home Assistant net | WAN | TCP | 443 | Pass | Integrations, updates, cloud API calls |
| 4 | Home Assistant net | Monitoring net | UDP | 514 | Pass | Forward logs to syslog collector |
| 5 | Home Assistant net | LAN net | * | * | Block | HA should not access trusted workstations |
| 6 | Home Assistant net | * | * | * | Block | Default deny (implicit) |

---

## Floating Rules

Applied across all interfaces regardless of direction.

| # | Source | Destination | Protocol | Port | Action | Direction | Rationale |
|---|--------|-------------|----------|------|--------|-----------|-----------|
| 1 | * | * | TCP/UDP | 53 | Block | Out | Prevent DNS bypass (all DNS must go through Unbound) |
| 2 | * | * | TCP/UDP | 853 | Block | Out | Block external DoT to force internal resolver |
| 3 | * | * | TCP/UDP | 5353 | Block | Out | Block mDNS leaving local segments |
| 4 | * | This firewall | ICMP | * | Pass | In | Allow ping to firewall for diagnostics |

## NAT Rules

- Outbound NAT: Automatic for WAN, manual override for VPN gateway to ensure kill switch behavior.
- Port forwards: None. No services are exposed to the internet.
- 1:1 NAT: Not used.
