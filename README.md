# DNS Role

Production-ready systemd-resolved configuration for Debian and Ubuntu. Takes control of DNS resolution, DNSSEC validation, DNS-over-TLS, DNS Service Discovery, and name service switching—so you don't have to fight with `/etc/resolv.conf` ever again.

## License

MIT

## Why Use This Role?

**The Problem**: Linux DNS configuration is fragmented. NetworkManager, systemd-resolved, resolvconf, and direct `/etc/resolv.conf` edits all compete. VPNs, containers, and cloud-init add more chaos. The result: DNS that works until it mysteriously doesn't.

**The Solution**: This role takes an opinionated stance—systemd-resolved is the single source of truth. It handles:

- **Package management**: Installs systemd-resolved and removes conflicting resolvers
- **Secure defaults**: DNSSEC validation and DNS-over-TLS enabled out of the box
- **Split-horizon DNS**: Route specific domains to different DNS servers (VPNs, internal networks)
- **Multi-homed hosts**: Per-interface DNS configuration that actually works
- **DNS Service Discovery**: Advertise services on the local network via mDNS
- **Production hardening**: LLMNR disabled by default to prevent credential harvesting
- **Observable**: Prometheus metrics exported for every configuration knob
- **Version-aware**: Auto-detects systemd version, skips unsupported features gracefully

## Quick Start

```yaml
# Basic: Use Cloudflare DNS with DNSSEC and DoT
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
          - 1.0.0.1
```

```yaml
# VPN split-tunnel: Route corp.example.com via VPN DNS
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
        dns_link_dns:
          - interface: wg0
            servers:
              - 10.100.0.1
            domains:
              - "~corp.example.com"
```

```yaml
# DNS-SD: Advertise services on the local network
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_mdns: true
        dns_mdns_interfaces:
          - eth0
        dns_sd_enabled: true
        dns_sd_services:
          - name: "%H Web Server"
            type: "_http._tcp"
            port: 80
          - name: "Home Assistant on %H"
            type: "_home-assistant._tcp"
            port: 8123
```

## Requirements

| Requirement | Version |
|-------------|---------|
| systemd | 243+ (basic), 250+ (all features) |
| Platform | Debian 11+, Ubuntu 22.04+ |
| Ansible | 2.15+ |
| Collections | `ansible.utils` |

## Platform Support

| Distribution | Version | systemd | Feature Support |
|-------------|---------|---------|-----------------|
| Debian | 12 (Bookworm) | 252 | Full |
| Debian | 13 (Trixie) | 256 | Full |
| Ubuntu | 24.04 (Noble) | 255 | Full |
| Debian | 11 (Bullseye) | 247 | Limited* |
| Ubuntu | 22.04 (Jammy) | 249 | Limited* |

*Limited: No `StaleRetentionSec`, `CacheFromLocalhost`. Role auto-detects and skips.

## Role Variables

### Core Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_enabled` | `true` | Enable/disable this role |
| `dns_resolved_enabled` | `true` | Enable systemd-resolved service |
| `dns_install_package` | `true` | Install systemd-resolved package |
| `dns_disable_conflicts` | `true` | Disable conflicting resolvers |
| `dns_config_filename` | `dns.conf` | Config file in resolved.conf.d |

### Conflict Resolution

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_remove_resolvconf` | `true` | Remove resolvconf package |
| `dns_disable_avahi` | `{{ dns_sd_enabled }}` | Disable avahi-daemon (when DNS-SD enabled) |
| `dns_disable_dnsmasq` | `false` | Disable dnsmasq |
| `dns_disable_unbound` | `false` | Disable unbound |

### DNS Servers

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_servers` | `[]` | Primary DNS servers (empty = DHCP/network defaults) |
| `dns_fallback_servers` | `['1.1.1.1', '8.8.8.8']` | Fallback DNS servers |
| `dns_domains` | `[]` | Search domains |

### Per-Link DNS

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_link_dns` | `[]` | Per-interface DNS and routing domains (runtime only) |

### Security

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_dnssec` | `allow-downgrade` | DNSSEC mode: `true`, `false`, `allow-downgrade` |
| `dns_over_tls` | `opportunistic` | DoT mode: `true`, `opportunistic`, `false` |
| `dns_llmnr` | `false` | LLMNR (disabled for security) |
| `dns_validate_ips` | `true` | Validate IP addresses before deployment |

### Caching

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_cache` | `true` | Enable DNS caching |
| `dns_stale_retention_sec` | `0` | Serve stale records when upstream fails [systemd 246+] |
| `dns_cache_from_localhost` | `true` | Cache responses from 127.0.0.0/8 [systemd 248+] |

### Stub Resolver

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_stub_listener` | `true` | Enable stub listener on 127.0.0.53 |
| `dns_stub_port` | `53` | Stub listener port |
| `dns_stub_listener_extra` | `[]` | Additional listener addresses [systemd 243+] |

### mDNS and DNS-SD

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_mdns` | `false` | Enable mDNS: `true`, `resolve`, `false` |
| `dns_mdns_interfaces` | `[]` | Interfaces to enable mDNS on |
| `dns_sd_enabled` | `false` | Enable DNS Service Discovery |
| `dns_sd_services` | `[]` | Services to advertise via DNS-SD |
| `dns_sd_cleanup` | `true` | Remove unmanaged .dnssd files |

### Hosts Management

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_hosts_entries` | `[]` | Custom /etc/hosts entries |
| `dns_nsswitch_hosts` | `files mymachines resolve [!UNAVAIL=return] myhostname` | NSSwitch hosts line |
| `dns_read_etc_hosts` | `true` | Read /etc/hosts before DNS queries |

### Advanced

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_cleanup_old_configs` | `true` | Remove orphaned config files |
| `dns_custom_settings` | `{}` | Additional resolved.conf settings |

## DNS Service Discovery (DNS-SD)

Advertise services on the local network via mDNS. Clients can discover services without knowing IP addresses.

### Configuration

```yaml
dns_mdns: true                    # Enable mDNS globally
dns_mdns_interfaces:              # Enable per-interface
  - eth0
  - br0
dns_sd_enabled: true              # Enable DNS-SD
dns_sd_services:
  - name: "%H"                    # %H = hostname
    type: "_http._tcp"            # Service type
    port: 80
    txt:                          # Optional TXT records
      - "path=/"
      - "version=1.0"
```

### Common Service Types

| Type | Description |
|------|-------------|
| `_http._tcp` | Web servers |
| `_https._tcp` | Secure web servers |
| `_ssh._tcp` | SSH servers |
| `_prometheus._tcp` | Prometheus metrics |
| `_grafana._tcp` | Grafana dashboards |
| `_home-assistant._tcp` | Home Assistant |
| `_hap._tcp` | HomeKit Accessory Protocol |
| `_mqtt._tcp` | MQTT brokers |

### Discovery

```bash
# From another machine on the network
resolvectl service _http._tcp.local
resolvectl service _prometheus._tcp.local
```

## Example Playbooks

### Strict Security

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_dnssec: true
        dns_over_tls: true
        dns_servers:
          - 1.1.1.1#cloudflare-dns.com
          - 9.9.9.9#dns.quad9.net
```

### NextDNS (Direct)

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_over_tls: true
        dns_servers:
          - PROFILE_ID.dns.nextdns.io
        dns_fallback_servers:
          - 45.90.28.0#PROFILE_ID.dns.nextdns.io
          - 45.90.30.0#PROFILE_ID.dns.nextdns.io
```

### Custom Hosts

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_hosts_entries:
          - ip: 10.0.0.1
            names:
              - gateway
              - gateway.example.com
          - ip: 10.0.0.10
            names:
              - nas
              - nas.example.com
```

### Smart Home with DNS-SD

```yaml
- hosts: homelab
  roles:
    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
        dns_mdns: true
        dns_mdns_interfaces:
          - eth0
        dns_sd_enabled: true
        dns_sd_services:
          - name: "Home Assistant on %H"
            type: "_home-assistant._tcp"
            port: 8123
            txt:
              - "base_url=https://homeassistant.local:8123"
          - name: "%H Grafana"
            type: "_grafana._tcp"
            port: 3000
          - name: "%H Prometheus"
            type: "_prometheus._tcp"
            port: 9090
```

## Handlers

| Handler | Description |
|---------|-------------|
| `Restart systemd-resolved` | Restart the service (used after config changes) |
| `Reload systemd-networkd` | Reload networkd (used after mDNS interface changes) |
| `Flush DNS cache` | Clear the DNS cache |

## Files

| Path | Description |
|------|-------------|
| `/etc/systemd/resolved.conf.d/*.conf` | Drop-in configurations |
| `/etc/systemd/dnssd/*.dnssd` | DNS-SD service definitions |
| `/etc/systemd/network/*.network.d/50-mdns.conf` | Per-interface mDNS config |
| `/etc/nsswitch.conf` | Name service switch configuration |
| `/etc/hosts` | Static host entries (managed block) |
| `/etc/resolv.conf` | Symlink to stub resolver |
| `/var/lib/node_exporter/textfile_collector/dns.prom` | Prometheus metrics |

## Troubleshooting

### Check Service Status

```bash
systemctl status systemd-resolved
resolvectl status
```

### Test DNS Resolution

```bash
# Query via systemd-resolved
resolvectl query example.com

# Check DNSSEC validation
resolvectl query --validate example.com

# Traditional tools (use stub resolver)
dig +short example.com
host example.com
```

### View Statistics

```bash
resolvectl statistics
```

### Flush Cache

```bash
resolvectl flush-caches
```

### List DNS-SD Services

```bash
# List configured services
ls -la /etc/systemd/dnssd/

# Query for services on network
resolvectl service _http._tcp.local
```

### Common Issues

#### "systemd-resolved is masked"

```bash
systemctl unmask systemd-resolved
systemctl enable --now systemd-resolved
```

#### DNS not working after role run

1. Check symlink: `ls -la /etc/resolv.conf`
2. Should point to: `/run/systemd/resolve/stub-resolv.conf`
3. If not, re-run role or manually fix:

```bash
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

#### DNSSEC failures

Some domains have misconfigured DNSSEC. Use `allow-downgrade` (default) or `false`:

```yaml
dns_dnssec: allow-downgrade
```

Debug with:

```bash
resolvectl query --validate failing-domain.com
```

#### DNS-SD services not discoverable

1. Verify mDNS is enabled: `resolvectl status | grep MulticastDNS`
2. Check interface mDNS: `networkctl status eth0 | grep MDNS`
3. Ensure firewall allows mDNS (port 5353/udp)
4. Verify Avahi is not running: `systemctl status avahi-daemon`

## Version Compatibility

| Feature | Minimum systemd |
|---------|-----------------|
| Basic DNS, DNSSEC, DoT | 237 |
| DNSStubListenerExtra | 243 |
| StaleRetentionSec | 246 |
| CacheFromLocalhost | 248 |
| DNS-SD (.dnssd files) | 237 |

## See Also

- [systemd-resolved(8)](https://www.freedesktop.org/software/systemd/man/systemd-resolved.html)
- [resolved.conf(5)](https://www.freedesktop.org/software/systemd/man/resolved.conf.html)
- [systemd.dnssd(5)](https://www.freedesktop.org/software/systemd/man/systemd.dnssd.html)
