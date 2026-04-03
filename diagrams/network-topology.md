# Network Topology

## Physical and Logical Layout

```text
                              ┌─────────────────┐
                              │   ISP Fiber ONT  │
                              │  (Bridge Mode)   │
                              └────────┬─────────┘
                                       │ 1 Gbps
                                       │ WAN (igc0)
                          ┌────────────┴────────────────┐
                          │                             │
                          │    OPNsense Firewall        │
                          │    Topton N355 Mini PC      │
                          │    Intel N100 / 16GB RAM     │
                          │    256GB NVMe                │
                          │                             │
                          │    Services:                │
                          │    - Firewall (pf)          │
                          │    - Suricata IDS/IPS       │
                          │    - Unbound DNS            │
                          │    - WireGuard VPN          │
                          │    - DHCP Server            │
                          │                             │
                          │    NICs:                    │
                          │    igc0 = WAN               │
                          │    igc1 = LAN trunk (VLANs) │
                          │    igc2 = Management        │
                          │    igc3-5 = reserved        │
                          │                             │
                          └──────┬──────────────┬───────┘
                                 │              │
                    VLAN trunk   │              │  Dedicated link
                    (802.1Q)     │              │  VLAN 40 untagged
                                 │              │
                    ┌────────────┴───┐    ┌─────┴──────────┐
                    │                │    │                 │
                    │   Managed      │    │  Admin Laptop   │
                    │   Switch       │    │  (Management)   │
                    │   24-port GbE  │    │  10.10.40.0/24  │
                    │                │    │                 │
                    │   Trunk port 1 │    └─────────────────┘
                    │   (all VLANs)  │
                    │                │
                    └┬───┬───┬───┬───┬────────────────────┘
                     │   │   │   │   │
          ┌──────────┘   │   │   │   └──────────┐
          │              │   │   │              │
     VLAN 10        VLAN 20  │  VLAN 60     VLAN 70
     LAN            IoT      │  Monitoring  Home Asst.
          │              │   │       │           │
          ▼              ▼   │       ▼           ▼
  ┌──────────────┐  ┌────────┴──┐  ┌──────────────┐  ┌──────────────┐
  │              │  │           │  │              │  │              │
  │   Wireless   │  │ Wireless  │  │ Raspberry    │  │ Home         │
  │   AP #1      │  │ AP #2     │  │ Pi 5         │  │ Assistant    │
  │              │  │           │  │              │  │ (VM/Docker)  │
  │  SSIDs:      │  │  SSIDs:   │  │ TIG Stack:   │  │              │
  │  - Home LAN  │  │  - IoT    │  │ - Telegraf   │  │ 10.10.70.0/24│
  │    (VLAN 10) │  │   (VLAN 20)│ │ - InfluxDB   │  │              │
  │  - Guest     │  │           │  │ - Grafana    │  └──────────────┘
  │    (VLAN 30) │  │           │  │              │
  │              │  │           │  │ 10.10.60.0/24│
  └──────────────┘  └───────────┘  └──────────────┘
          │              │
          ▼              ▼
  ┌──────────────┐  ┌───────────────────────┐
  │ LAN Devices  │  │ IoT Devices           │
  │              │  │                       │
  │ - Workstation│  │ - Smart plugs         │
  │ - Laptop     │  │ - Temperature sensors │
  │ - NAS        │  │ - IP cameras          │
  │ - Printer    │  │ - Smart lights        │
  │              │  │ - Robot vacuum        │
  │ 10.10.10.0/24│  │                       │
  └──────────────┘  │ 10.10.20.0/24         │
                    └───────────────────────┘


  Guest Devices (VLAN 30 - 10.10.30.0/24)
  ┌───────────────────────┐
  │ - Visitor phones      │
  │ - Visitor laptops     │
  │ (Client isolation ON) │
  │ (No internal access)  │
  └───────────────────────┘
```

## VPN Tunnel (Logical Overlay)

```text
  ┌──────────────────┐         WireGuard Tunnel           ┌──────────────────┐
  │                  │         (UDP port 51820)            │                  │
  │  OPNsense        │◄═══════════════════════════════════►│  Mullvad VPN     │
  │  wg0 interface   │         Encrypted tunnel           │  Exit node       │
  │  10.64.100.1/32  │                                    │                  │
  │                  │         AllowedIPs = 0.0.0.0/0     │                  │
  └──────────────────┘         (kill switch routing)       └──────────────────┘
          ▲
          │  Gateway for:
          │  - VLAN 10 (LAN)
          │  - VLAN 40 (Management)
          │
          │  NOT used by:
          │  - VLAN 30 (Guest - direct WAN)
          │  - VLAN 60 (Monitoring - limited WAN)
```

## Data Flow: Syslog and Metrics

```text
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ OPNsense │  │   LAN    │  │   IoT    │  │   VPN    │  │ Home Ast │
  │ Firewall │  │ Devices  │  │ Devices  │  │ Gateway  │  │          │
  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │              │              │
       │   syslog     │   syslog     │   syslog     │   syslog     │
       │   UDP/514    │   UDP/514    │   UDP/514    │   UDP/514    │
       │              │              │              │              │
       └──────────────┴──────┬───────┴──────────────┴──────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   Raspberry Pi  │
                    │   (VLAN 60)     │
                    │                 │
                    │  ┌───────────┐  │
                    │  │ Telegraf  │  │  Collects syslog + SNMP
                    │  └─────┬─────┘  │
                    │        │        │
                    │  ┌─────▼─────┐  │
                    │  │ InfluxDB  │  │  Time-series storage
                    │  └─────┬─────┘  │
                    │        │        │
                    │  ┌─────▼─────┐  │
                    │  │ Grafana   │  │  Dashboards + alerting
                    │  │ :3000     │  │
                    │  └───────────┘  │
                    └─────────────────┘
```
