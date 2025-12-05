# Usage Guide

Practical examples from simple to complex. Each scenario builds on the previous.

## Simple Scenarios

### 1. Just Make DNS Work

Accept all defaults. DHCP-provided DNS with DNSSEC validation and opportunistic DoT.

```yaml
- hosts: servers
  roles:
    - role: dns
```

**What you get:**
- systemd-resolved enabled and running
- DHCP-provided DNS servers used
- Cloudflare + Google as fallback
- DNSSEC: validate when possible
- DoT: encrypt when possible
- LLMNR/mDNS: disabled (security)

### 2. Use Specific DNS Servers

Override DHCP DNS with your preferred servers.

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
          - 1.0.0.1
```

### 3. Add Search Domains

Make single-label names work (e.g., `ssh server1` becomes `server1.example.com`).

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
        dns_domains:
          - example.com
          - internal.example.com
```

### 4. Add Static Hosts

Local name resolution without DNS.

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
        dns_hosts_entries:
          - ip: 10.0.0.1
            names:
              - gateway
              - router
          - ip: 10.0.0.10
            names:
              - nas
              - storage.home.lan
```

## Intermediate Scenarios

### 5. Privacy-Focused DNS

Use DNS-over-TLS with encrypted providers.

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_over_tls: true
        dns_servers:
          # Server hostname for certificate validation
          - 1.1.1.1#cloudflare-dns.com
          - 9.9.9.9#dns.quad9.net
        dns_fallback_servers:
          - 45.90.28.0#dns.nextdns.io
```

**Note:** With `dns_over_tls: true`, DNS fails if servers don't support DoT. Use `opportunistic` for graceful fallback.

### 6. Security-Hardened DNS

Maximum validation and encryption.

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_dnssec: true         # Strict DNSSEC
        dns_over_tls: true       # Require DoT
        dns_llmnr: false         # Already default
        dns_mdns: false          # Already default
        dns_servers:
          - 1.1.1.1#cloudflare-dns.com
          - 9.9.9.9#dns.quad9.net
```

**Warning:** Strict DNSSEC may break domains with misconfigured DNSSEC. Test thoroughly.

### 7. NextDNS Integration

Direct integration with NextDNS for filtering and analytics.

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_over_tls: true
        dns_servers:
          # Replace YOUR_PROFILE_ID with your NextDNS config ID
          - 45.90.28.0#YOUR_PROFILE_ID.dns.nextdns.io
          - 45.90.30.0#YOUR_PROFILE_ID.dns.nextdns.io
        dns_fallback_servers:
          - 2a07:a8c0::#YOUR_PROFILE_ID.dns.nextdns.io
          - 2a07:a8c1::#YOUR_PROFILE_ID.dns.nextdns.io
```

### 8. Quad9 with Malware Blocking

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_over_tls: opportunistic
        dns_dnssec: true
        dns_servers:
          - 9.9.9.9#dns.quad9.net
          - 149.112.112.112#dns.quad9.net
        dns_fallback_servers:
          - 9.9.9.10  # Quad9 unfiltered fallback
```

## Complex Scenarios

### 9. VPN Split-Tunnel DNS

Route corporate domains through VPN, everything else through normal DNS.

```yaml
- hosts: workstations
  roles:
    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
        dns_link_dns:
          - interface: wg0  # WireGuard VPN
            servers:
              - 10.100.0.1   # Corporate DNS
              - 10.100.0.2
            domains:
              - "~corp.example.com"    # Route these via VPN
              - "~internal.example.com"
              - "~10.in-addr.arpa"     # Reverse DNS for 10.x.x.x
```

**How it works:**
- Queries for `server.corp.example.com` → VPN DNS (10.100.0.1)
- Queries for `google.com` → Normal DNS (1.1.1.1)
- Reverse lookups for 10.x.x.x → VPN DNS

**Important:** `dns_link_dns` is runtime only (survives until reboot). For persistence, configure in `.network` files.

### 10. Multi-Homed Host

Server with multiple networks, each with its own DNS.

```yaml
- hosts: multi-homed
  roles:
    - role: dns
      vars:
        dns_servers: []  # No global servers
        dns_link_dns:
          - interface: eth0  # Public network
            servers:
              - 1.1.1.1
            domains:
              - "~."  # Default route for all domains

          - interface: eth1  # Management network
            servers:
              - 192.168.100.1
            domains:
              - "~mgmt.local"
              - "~168.192.in-addr.arpa"

          - interface: eth2  # Storage network
            servers:
              - 10.20.0.1
            domains:
              - "~storage.local"
              - "~20.10.in-addr.arpa"
```

### 11. Docker Host with Container DNS

Provide DNS to containers via additional stub listener.

```yaml
- hosts: docker-hosts
  roles:
    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
        dns_stub_listener_extra:
          - "172.17.0.1"     # Docker default bridge
          - "172.18.0.1"     # Custom bridge network
```

Configure Docker to use host DNS:

```json
// /etc/docker/daemon.json
{
  "dns": ["172.17.0.1"]
}
```

Or per-container:

```yaml
# docker-compose.yml
services:
  myapp:
    dns:
      - 172.17.0.1
```

### 12. Local DNS Server Integration

Use Pi-hole or Unbound running locally.

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers:
          - 127.0.0.1#5353  # Local Pi-hole on port 5353
        dns_fallback_servers:
          - 1.1.1.1  # Fallback if Pi-hole is down
        dns_cache_from_localhost: false  # Don't double-cache
```

**Why port 5353?** Pi-hole's default (port 53) conflicts with systemd-resolved's stub listener. Either:
- Run Pi-hole on 5353 (shown above)
- Disable stub listener: `dns_stub_listener: false`

### 13. Kubernetes Node Configuration

Configure DNS for k8s nodes that need both cluster DNS and external resolution.

```yaml
- hosts: k8s-nodes
  roles:
    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1
        dns_link_dns:
          - interface: cni0  # Kubernetes CNI bridge
            servers:
              - 10.96.0.10  # CoreDNS service IP
            domains:
              - "~cluster.local"
              - "~svc.cluster.local"
              - "~pod.cluster.local"
        dns_stub_listener_extra:
          - "10.244.0.1"  # Node's pod network IP
```

## Inventory-Based Configuration

### 14. Host Groups with Different DNS

```yaml
# inventory/group_vars/all.yml
dns_servers:
  - 1.1.1.1
dns_dnssec: allow-downgrade
dns_over_tls: opportunistic

# inventory/group_vars/corporate.yml
dns_servers:
  - 10.0.0.53
  - 10.0.0.54
dns_domains:
  - corp.example.com
dns_dnssec: false  # Internal DNS doesn't support DNSSEC

# inventory/group_vars/dmz.yml
dns_servers:
  - 1.1.1.1
  - 9.9.9.9
dns_dnssec: true
dns_over_tls: true

# inventory/host_vars/special-host.yml
dns_enabled: false  # This host manages its own DNS
```

### 15. Conditional DNS Based on Facts

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers: "{{
          ['10.0.0.53'] if ansible_default_ipv4.network == '10.0.0.0'
          else ['1.1.1.1']
        }}"
```

## Playbook Integration

### 16. With systemd-networkd

systemd-networkd `.network` files can configure persistent per-link DNS. Use both for complete coverage:

```yaml
- hosts: servers
  tasks:
    # Configure network interface with persistent DNS
    - name: Configure WireGuard VPN interface
      ansible.builtin.copy:
        dest: /etc/systemd/network/50-wg0.network
        content: |
          [Match]
          Name=wg0

          [Network]
          Address=10.100.0.5/24
          DNS=10.100.0.1
          Domains=~corp.example.com
        mode: '0644'
      notify: restart networkd

  roles:
    # DNS role configures systemd-resolved
    - role: dns
      vars:
        dns_servers:
          - 1.1.1.1

  handlers:
    - name: restart networkd
      ansible.builtin.systemd:
        name: systemd-networkd
        state: restarted
```

### 17. With Firewall Rules

If exposing stub listener beyond localhost, ensure firewall allows DNS traffic:

```yaml
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_stub_listener_extra:
          - "192.168.1.1"

  tasks:
    # Example using ufw
    - name: Allow DNS from LAN
      community.general.ufw:
        rule: allow
        port: '53'
        proto: udp
        from_ip: 192.168.1.0/24
        comment: "DNS for LAN clients"

    # Or using nftables
    - name: Allow DNS from LAN (nftables)
      ansible.builtin.shell: |
        nft add rule inet filter input ip saddr 192.168.1.0/24 udp dport 53 accept
      when: use_nftables | default(false)

## Testing Your Configuration

After applying the role:

```bash
# Check service status
systemctl status systemd-resolved

# View current configuration
resolvectl status

# Test resolution
resolvectl query google.com
resolvectl query --validate cloudflare.com  # Test DNSSEC

# Check per-link DNS
resolvectl dns
resolvectl domain

# View statistics
resolvectl statistics

# Flush cache (if testing changes)
resolvectl flush-caches
```
