# Hardware Bill of Materials

Overview of all hardware components in the homelab infrastructure.

## Firewall / Router

| Component | Details |
|-----------|---------|
| **Model** | Topton N355 Mini PC (fanless) |
| **CPU** | Intel N100 (4 cores, 4 threads, 3.4 GHz boost) |
| **RAM** | 16 GB DDR5 |
| **Storage** | 256 GB NVMe SSD |
| **NICs** | 6x Intel i226-V 2.5GbE ports |
| **OS** | OPNsense (FreeBSD-based) |
| **Role** | Firewall, router, VPN gateway, DNS resolver, IDS/IPS |

**Notes:** The Topton N355 was chosen for its fanless design (silent operation), low power consumption (~15W TDP), and six Intel NICs with native FreeBSD driver support. AES-NI hardware acceleration handles WireGuard and Suricata workloads without performance issues. The Intel i226-V NICs had early firmware issues with 2.5GbE negotiation -- resolved by updating NIC firmware to v2.14+.

## Monitoring Server

| Component | Details |
|-----------|---------|
| **Model** | Raspberry Pi 5 |
| **CPU** | Broadcom BCM2712, Cortex-A76 (4 cores, 2.4 GHz) |
| **RAM** | 8 GB LPDDR4X |
| **Storage** | 256 GB microSD + 512 GB USB 3.0 SSD (data) |
| **OS** | Ubuntu Server 24.04 LTS (64-bit) |
| **Role** | TIG stack (Telegraf, InfluxDB, Grafana), syslog collector |

**Notes:** The Pi 5 is a significant performance upgrade over the Pi 4 for InfluxDB write-heavy workloads. The USB SSD stores InfluxDB data to avoid microSD wear. Grafana dashboards serve the LAN and Management VLANs. Telegraf collects syslog (UDP 514), SNMP polling from the managed switch, and Suricata EVE JSON logs from OPNsense.

## Managed Switch

| Component | Details |
|-----------|---------|
| **Model** | TP-Link TL-SG1024DE (or similar 24-port managed GbE) |
| **Ports** | 24x Gigabit Ethernet |
| **Management** | Web GUI, SNMP v2c |
| **Features** | 802.1Q VLAN, port mirroring, IGMP snooping, link aggregation |
| **Role** | VLAN distribution, inter-device connectivity |

**Notes:** A "smart managed" switch is sufficient for this setup since all routing and security decisions happen at the OPNsense layer. The switch handles VLAN tagging and trunking. Port 1 is the trunk port carrying all VLANs from OPNsense. SNMP is enabled for monitoring by Telegraf on the Pi.

## Wireless Access Points

| Component | Details |
|-----------|---------|
| **Model** | TP-Link EAP670 (Wi-Fi 6, AX5400) x2 |
| **Bands** | Dual-band (2.4 GHz + 5 GHz) |
| **Standard** | Wi-Fi 6 (802.11ax) |
| **PoE** | 802.3at PoE+ powered |
| **Management** | Omada SDN controller (self-hosted or standalone) |
| **Role** | Wireless connectivity for LAN, IoT, and Guest SSIDs |

**AP Assignment:**

| AP | Location | SSIDs Served | VLANs |
|----|----------|-------------|-------|
| AP #1 | Living area | Home LAN (VLAN 10), Guest (VLAN 30) | 10, 30 |
| AP #2 | Utility area | IoT (VLAN 20) | 20 |

**Notes:** Each SSID is mapped to a specific VLAN via the AP's multi-SSID configuration. Guest SSID has client isolation enabled to prevent guest-to-guest communication. IoT SSID uses WPA2 for compatibility with older smart home devices. LAN and Guest SSIDs use WPA3-Personal.

## UPS (Uninterruptible Power Supply)

| Component | Details |
|-----------|---------|
| **Model** | APC Back-UPS BX700 (or similar 700VA line-interactive) |
| **Capacity** | 700VA / 390W |
| **Runtime** | ~15 minutes at typical homelab load (~80W) |
| **Outlets** | 4x battery-backed, 4x surge-only |
| **Role** | Power protection for firewall, switch, and Pi |

**Notes:** The UPS protects the core network path (OPNsense + switch + Pi) from brief power outages and provides clean shutdown time. The firewall and Pi are connected to battery-backed outlets. APs are PoE-powered through the switch, so they are also indirectly protected.

## Cabling

| Type | Spec | Usage |
|------|------|-------|
| Ethernet | Cat 6 (various lengths) | All wired connections |
| Fiber | ISP-provided LC-LC single-mode | ONT to wall jack |
| USB | USB-C power delivery | Pi 5 power |
| USB | USB 3.0 Type-A to SATA | Pi external SSD |

## Power Consumption Estimate

| Device | Typical Draw |
|--------|-------------|
| Topton N355 | ~12-15W |
| Raspberry Pi 5 | ~5-8W |
| Managed Switch | ~10-15W |
| APs (x2 via PoE) | ~20-25W |
| **Total** | **~50-65W** |

The entire homelab infrastructure runs on under 65W, making it viable for 24/7 operation with minimal electricity cost.
