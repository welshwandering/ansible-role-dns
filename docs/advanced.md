# Advanced Usage

Niche scenarios, integration patterns, and esoteric configurations.

## DNS Server Configurations

### Running Local DNS Server

When running Pi-hole, Unbound, or another local DNS server:

```yaml
# Pi-hole on port 5353 (avoids port 53 conflict)
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers:
          - 127.0.0.1#5353
        dns_cache_from_localhost: false  # Pi-hole caches
        dns_dnssec: false  # Pi-hole handles DNSSEC
```

```yaml
# Disable stub listener, use Pi-hole directly
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_stub_listener: false
        dns_servers:
          - 127.0.0.1
```

Then update `/etc/resolv.conf` to point directly to Pi-hole (not managed by this role when stub listener is disabled).

### Recursive Resolver (Unbound)

```yaml
# Unbound as local recursive resolver
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers:
          - 127.0.0.1#5335  # Unbound default port
        dns_dnssec: false  # Unbound validates
        dns_cache: false   # Unbound caches
        dns_over_tls: false  # Unbound handles DoT upstream
```

### DNS-over-HTTPS via Proxy

systemd-resolved doesn't support DoH natively.[^1] Use a local proxy:

```yaml
# cloudflared or dnscrypt-proxy on port 5053
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers:
          - 127.0.0.1#5053
        dns_over_tls: false  # DoH proxy handles encryption
        dns_cache_from_localhost: false
```

## Split-Horizon DNS Patterns

### Corporate VPN with Multiple Zones

```yaml
dns_link_dns:
  - interface: tun0
    servers:
      - 10.0.0.53
    domains:
      # Production
      - "~prod.corp.com"
      - "~aws.corp.com"
      # Development
      - "~dev.corp.com"
      - "~staging.corp.com"
      # Internal tools
      - "~jira.corp.com"
      - "~confluence.corp.com"
      # Reverse DNS for internal ranges
      - "~10.in-addr.arpa"
      - "~172.16.in-addr.arpa"
```

### Multi-VPN Setup

```yaml
dns_link_dns:
  # Work VPN
  - interface: wg-work
    servers:
      - 10.100.0.1
    domains:
      - "~work.example.com"

  # Lab VPN
  - interface: wg-lab
    servers:
      - 10.200.0.1
    domains:
      - "~lab.example.com"

  # Personal VPN (default route for everything else)
  - interface: wg-personal
    servers:
      - 10.8.0.1
    domains:
      - "~."  # Catch-all
```

### Reverse DNS Zones

Route reverse lookups to appropriate DNS servers:

```yaml
dns_link_dns:
  - interface: eth1
    servers:
      - 192.168.1.1
    domains:
      # Forward zone
      - "~internal.local"
      # Reverse zones (RFC 1918)
      - "~168.192.in-addr.arpa"    # 192.168.x.x
      - "~16.172.in-addr.arpa"     # 172.16.x.x
      - "~10.in-addr.arpa"         # 10.x.x.x
      # IPv6 reverse zone
      - "~8.b.d.0.1.0.0.2.ip6.arpa"
```

## NSSwitch Customization

### Without Container Support

```yaml
# Skip mymachines if not running containers
dns_nsswitch_hosts: "files resolve [!UNAVAIL=return] myhostname"
```

### With mDNS for .local

```yaml
# Enable mDNS in NSSwitch (also needs dns_mdns: true)
dns_mdns: true
dns_nsswitch_hosts: "files mymachines mdns4_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] myhostname"
```

### Legacy DNS Fallback

```yaml
# Fall back to legacy DNS if resolved fails
dns_nsswitch_hosts: "files mymachines resolve dns myhostname"
```

### LDAP/NIS Name Resolution

```yaml
# Include LDAP for user/group resolution
dns_nsswitch_hosts: "files mymachines resolve [!UNAVAIL=return] ldap myhostname"
```

## Stub Listener Advanced Configurations

### Multiple Stub Listeners

Expose DNS on multiple addresses for different networks:

```yaml
dns_stub_listener_extra:
  - "127.0.0.54"            # Alternative loopback
  - "192.168.1.1"           # LAN interface
  - "172.17.0.1"            # Docker bridge
  - "10.244.0.1"            # Kubernetes node IP
  - "192.168.1.1:5353"      # Custom port for specific client
```

### Stub Listener for LXC/LXD Containers

```yaml
dns_stub_listener_extra:
  - "10.0.3.1"   # lxdbr0 default
```

In container profile:
```yaml
config:
  raw.lxc: |
    lxc.net.0.flags = up
    lxc.net.0.ipv4.address = 10.0.3.0/24
    lxc.net.0.ipv4.gateway = 10.0.3.1
devices:
  eth0:
    nictype: bridged
    parent: lxdbr0
    type: nic
```

## Tailscale Integration

### MagicDNS Coexistence

Tailscale's MagicDNS configures per-link DNS automatically. This role coexists:

```yaml
- hosts: servers
  roles:
    - role: tailscale
      vars:
        tailscale_args: "--accept-dns=true"

    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
        # Don't configure tailscale0 - Tailscale manages it
```

### Override Tailscale DNS

```yaml
- hosts: servers
  roles:
    - role: tailscale
      vars:
        tailscale_args: "--accept-dns=false"

    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
        dns_link_dns:
          - interface: tailscale0
            servers:
              - 100.100.100.100  # Tailscale DNS
            domains:
              - "~ts.net"
              - "~tail1234.ts.net"  # Your tailnet
```

## Kubernetes Integration

### Node DNS for Pods

Configure nodes to provide DNS to pods:

```yaml
dns_servers:
  - 1.1.1.1  # External DNS
dns_link_dns:
  - interface: cni0
    servers:
      - 10.96.0.10  # kube-dns/CoreDNS ClusterIP
    domains:
      - "~cluster.local"
      - "~svc.cluster.local"
      - "~pod.cluster.local"
dns_stub_listener_extra:
  - "{{ ansible_default_ipv4.address }}"  # Node IP for pods
```

### With Calico CNI

```yaml
dns_stub_listener_extra:
  - "{{ calico_node_ip }}"  # Node's Calico IP
```

## Prometheus Metrics

The role exports metrics to `/var/lib/node_exporter/textfile_collector/dns.prom`:

```promql
# Alert on DNSSEC disabled
alert: DNSSECDisabled
expr: dns_dnssec_strict == 0 and dns_dnssec_allow_downgrade == 0
for: 5m
labels:
  severity: warning
annotations:
  summary: "DNSSEC disabled on {{ $labels.instance }}"

# Alert on LLMNR enabled (security risk)[^2]
alert: LLMNREnabled
expr: dns_llmnr_enabled == 1
for: 0m
labels:
  severity: warning
annotations:
  summary: "LLMNR enabled on {{ $labels.instance }} - security risk"

# Track systemd version
dns_systemd_version{instance="server1"} 252
```

## Custom resolved.conf Options

For options not exposed as variables:

```yaml
dns_custom_settings:
  # Resolve single-label names via unicast DNS
  ResolveUnicastSingleLabel: "yes"

  # Maximum number of DNS transactions
  DNSStubListener: "udp"  # UDP only, no TCP

  # Custom LLMNR/mDNS per-link (usually via resolvectl)
  # These apply globally
```

See [resolved.conf(5)](https://www.freedesktop.org/software/systemd/man/resolved.conf.html) for all options.

## Testing and Validation

### Molecule Testing

The role includes Molecule tests. Run locally:

```bash
cd ansible/roles/dns
molecule test
```

Test specific scenario:

```bash
molecule test -s disabled
```

### Manual Validation

```bash
# Verify idempotence
ansible-playbook your-playbook.yml -l host --tags dns
ansible-playbook your-playbook.yml -l host --tags dns
# Second run should show 0 changed
```

### DNS Benchmark

```bash
# Install dnsperf
apt install dnsperf

# Benchmark
echo "google.com A" > queries.txt
dnsperf -s 127.0.0.53 -d queries.txt -l 30
```

## Migration Scenarios

### From NetworkManager DNS

```bash
# 1. Disable NetworkManager DNS management
# /etc/NetworkManager/NetworkManager.conf
[main]
dns=none

# 2. Restart NetworkManager
systemctl restart NetworkManager

# 3. Run this role
ansible-playbook your-playbook.yml -l host --tags dns
```

### From resolvconf

```bash
# 1. Remove resolvconf
apt remove resolvconf

# 2. Run this role
ansible-playbook your-playbook.yml -l host --tags dns
```

### From Direct resolv.conf

The role handles this automatically:
1. Backs up existing `/etc/resolv.conf`
2. Creates symlink to stub-resolv.conf

## Coexistence Patterns

### With cloud-init

cloud-init may manage DNS. Options:

```yaml
# Option 1: Disable cloud-init DNS
# /etc/cloud/cloud.cfg.d/99-disable-dns.cfg
manage_resolv_conf: false

# Option 2: Let this role override (cleanup enabled by default)
dns_cleanup_old_configs: true  # Removes cloud-init's resolved.conf
```

### With systemd-networkd

systemd-networkd can configure per-link DNS persistently. This role configures global settings; networkd handles per-link:

```ini
# /etc/systemd/network/10-eth0.network
[Network]
DNS=192.168.1.1
Domains=~internal.local
```

This role's `dns_link_dns` is runtime only; networkd's is persistent.

### With DHCP

By default (`dns_servers: []`), DHCP-provided DNS is used. To override:

```yaml
# Ignore DHCP DNS, use these instead
dns_servers:
  - 1.1.1.1
```

## Debugging Internals

### View Resolved's View

```bash
# What resolved thinks the config is
busctl introspect org.freedesktop.resolve1 /org/freedesktop/resolve1
busctl get-property org.freedesktop.resolve1 /org/freedesktop/resolve1 org.freedesktop.resolve1.Manager DNS
busctl get-property org.freedesktop.resolve1 /org/freedesktop/resolve1 org.freedesktop.resolve1.Manager DNSOverTLS
```

### D-Bus Debugging

```bash
# Monitor resolved D-Bus traffic
dbus-monitor --system "interface='org.freedesktop.resolve1.Manager'"
```

### Trace DNS Queries

```bash
# Using resolvectl
resolvectl monitor

# Using systemd-resolved debug logging
systemctl edit systemd-resolved
# Add:
# [Service]
# Environment=SYSTEMD_LOG_LEVEL=debug

systemctl restart systemd-resolved
journalctl -u systemd-resolved -f
```

## Performance Tuning

### High-Volume DNS

```yaml
dns_cache: true
dns_stale_retention_sec: 300  # 5 minutes
dns_stub_listener_extra:
  - "127.0.0.54"  # Additional listener for load distribution
```

### Low-Latency DNS

```yaml
dns_servers:
  - 1.1.1.1  # Choose geographically close, fast DNS
dns_over_tls: false  # Skip TLS handshake latency
dns_cache: true
```

### Memory-Constrained Systems

```yaml
dns_cache: false  # Disable caching
dns_stale_retention_sec: 0
```

## References

[^1]: systemd-resolved feature discussion, https://github.com/systemd/systemd/issues/8639 - DoH support has been requested but is not implemented as of systemd 256

[^2]: MITRE ATT&CK T1557.001, "LLMNR/NBT-NS Poisoning and SMB Relay," https://attack.mitre.org/techniques/T1557/001/

[^3]: resolved.conf(5), https://www.freedesktop.org/software/systemd/man/resolved.conf.html - All configuration options

[^4]: Tailscale DNS documentation, https://tailscale.com/kb/1054/dns/ - MagicDNS and split DNS behavior
