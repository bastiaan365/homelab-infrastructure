# VLAN Assignments

All network segmentation is implemented using 802.1Q VLANs trunked from the OPNsense firewall (Topton N355) to the managed switch. Each VLAN maps to a dedicated OPNsense interface with its own firewall ruleset, DHCP scope, and DNS policy.

## VLAN Table

| VLAN ID | Name | Subnet | Gateway | DHCP Range | Purpose | Internet Access | Cross-Zone Access |
|---------|------|--------|---------|------------|---------|-----------------|-------------------|
| 10 | LAN | 10.10.10.0/24 | 10.10.10.1 | .100-.200 | Trusted workstations, personal devices | Yes, via VPN tunnel | Monitoring (read), Management (GUI), Home Assistant (dashboard) |
| 20 | IoT | 10.10.20.0/24 | 10.10.20.1 | .100-.200 | Smart home devices, cameras, sensors | Yes, HTTPS only (filtered) | Home Assistant only (MQTT + API). All other zones blocked |
| 30 | Guest | 10.10.30.0/24 | 10.10.30.1 | .100-.250 | Visitor Wi-Fi | Yes, direct (HTTP/HTTPS only) | No cross-zone access. Fully isolated |
| 40 | Management | 10.10.40.0/24 | 10.10.40.1 | .100-.120 | Network admin, OPNsense GUI, SSH | Yes, via VPN (direct fallback) | Full access to all zones for administration |
| 50 | VPN | 10.10.50.0/24 | 10.10.50.1 | Static only | WireGuard tunnel gateway (Mullvad) | Yes, encrypted tunnel | No cross-zone access. Egress only |
| 60 | Monitoring | 10.10.60.0/24 | 10.10.60.1 | .100-.110 | TIG stack (Telegraf, InfluxDB, Grafana) | Limited (updates, NTP) | Receive-only. Accepts syslog and metrics from all zones |
| 70 | Home Assistant | 10.10.70.0/24 | 10.10.70.1 | .100-.110 | Smart home automation controller | Yes, HTTPS only | Outbound to IoT zone. Inbound from LAN (dashboard) |

## VLAN Design Notes

### Trunk Configuration
- **Trunk port:** OPNsense LAN NIC (igc1) carries all VLANs as tagged traffic to the managed switch.
- **Native VLAN:** VLAN 10 (LAN) is the native/untagged VLAN on the trunk.
- **Access ports:** Each switch port serving end devices is configured as an untagged access port on the appropriate VLAN.

### DHCP
- Each VLAN has its own DHCP scope served by OPNsense.
- DHCP lease time is 1 hour for Guest, 12 hours for IoT, and 24 hours for all other zones.
- Static DHCP mappings are used for infrastructure devices (Raspberry Pi, APs, switch).

### DNS
- All VLANs use the OPNsense Unbound resolver (gateway IP) as their DNS server.
- DNS-over-TLS forwarding to Quad9 (9.9.9.9) and Cloudflare (1.1.1.1) as upstream.
- DNSSEC validation is enabled.
- DNS blocklists (Steven Black unified list) are applied to IoT, Guest, and Home Assistant zones.
- Firewall floating rules block all direct DNS (port 53, 853) to external servers, forcing all clients through Unbound.

### Wireless
- **LAN SSID:** Mapped to VLAN 10, WPA3-Personal.
- **IoT SSID:** Mapped to VLAN 20, WPA2-Personal (compatibility with older IoT devices).
- **Guest SSID:** Mapped to VLAN 30, WPA3-Personal, client isolation enabled.

### Inter-VLAN Routing
- All inter-VLAN routing is handled by OPNsense (router-on-a-stick).
- Cross-zone traffic is controlled entirely by firewall rules on each VLAN interface.
- No static routes are needed; all subnets are directly connected to the firewall.
