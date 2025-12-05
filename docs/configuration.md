# Configuration Reference

Complete reference for all DNS role variables. See [defaults/main.yml](../defaults/main.yml) for the canonical source.

## Role Control

### dns_enabled

Enable or disable the entire role.

| Property | Value |
|----------|-------|
| Type | boolean |
| Default | `true` |

When `false`, the role exits immediately without making any changes. Useful for conditional role application based on host groups or facts.

```yaml
# Disable DNS role on this host
dns_enabled: false
```

### dns_resolved_enabled

Enable or disable systemd-resolved service management.

| Property | Value |
|----------|-------|
| Type | boolean |
| Default | `true` |

When `false`:
- Configuration files are not deployed
- Service is stopped and disabled
- `/etc/resolv.conf` symlink is not managed

```yaml
# Don't manage systemd-resolved (use for hosts with other DNS resolvers)
dns_resolved_enabled: false
```

## DNS Servers

### dns_servers

Primary DNS servers for resolution.

| Property | Value |
|----------|-------|
| Type | list of strings |
| Default | `[]` (empty = DHCP/network defaults) |

Supports:
- IPv4 addresses: `1.1.1.1`
- IPv6 addresses: `2606:4700:4700::1111`
- With port: `1.1.1.1#53`
- With hostname (for DoT SNI): `1.1.1.1#cloudflare-dns.com`

```yaml
# Cloudflare DNS
dns_servers:
  - 1.1.1.1
  - 1.0.0.1

# With DoT server name (for certificate validation)
dns_servers:
  - 1.1.1.1#cloudflare-dns.com
  - 9.9.9.9#dns.quad9.net

# Empty = use DHCP-provided DNS
dns_servers: []
```

### dns_fallback_servers

Servers used when primary servers are unreachable.

| Property | Value |
|----------|-------|
| Type | list of strings |
| Default | `['1.1.1.1', '8.8.8.8']` |

Same format as `dns_servers`. Used only when:
- No primary servers configured and DHCP fails
- All primary servers timeout
- Network is completely misconfigured

```yaml
dns_fallback_servers:
  - 9.9.9.9      # Quad9
  - 208.67.222.222  # OpenDNS
```

### dns_domains

Search domains appended to single-label names.

| Property | Value |
|----------|-------|
| Type | list of strings |
| Default | `[]` |

When you query `server1`, systemd-resolved tries:
1. `server1` (single-label lookup)
2. `server1.example.com` (first search domain)
3. `server1.internal.corp` (second search domain)

```yaml
dns_domains:
  - example.com
  - internal.corp
```

## Per-Link DNS

### dns_link_dns

Interface-specific DNS servers and routing domains.

| Property | Value |
|----------|-------|
| Type | list of objects |
| Default | `[]` |

Each object:
- `interface`: Network interface name (required)
- `servers`: List of DNS servers for this interface
- `domains`: List of routing domains (prefix with `~` for routing-only)

**Important**: This is runtime configuration via `resolvectl`. It does NOT survive reboots. For persistent per-link DNS, use systemd-networkd `.network` files.

```yaml
dns_link_dns:
  # VPN: route corp domains through VPN DNS
  - interface: wg0
    servers:
      - 10.100.0.1
    domains:
      - "~corp.example.com"
      - "~internal.corp"

  # Direct interface to internal network
  - interface: eth1
    servers:
      - 192.168.1.1
    domains:
      - "~lan.home"
```

**Routing domains (`~` prefix):**
- `~corp.example.com`: Route queries for this domain to this interface's DNS
- NOT added to search path
- Enables split-horizon DNS

**Search domains (no prefix):**
- `internal.corp`: Both route queries AND add to search path
- Single-label queries try this domain

## Security

### dns_dnssec

DNSSEC validation mode.

| Property | Value |
|----------|-------|
| Type | string or boolean |
| Default | `allow-downgrade` |
| Values | `true`, `false`, `allow-downgrade` |

| Mode | Behavior |
|------|----------|
| `true` | Strict validation. Fails if DNSSEC validation fails. |
| `false` | No validation. Accepts all responses. |
| `allow-downgrade` | Validates if available, accepts unsigned responses. |

```yaml
# Recommended: validate when possible, don't break unsigned domains
dns_dnssec: allow-downgrade

# Strict: some domains will fail (misconfigured DNSSEC is common)
dns_dnssec: true
```

### dns_over_tls

DNS-over-TLS (DoT) mode.

| Property | Value |
|----------|-------|
| Type | string or boolean |
| Default | `opportunistic` |
| Values | `true`, `opportunistic`, `false` |

| Mode | Behavior |
|------|----------|
| `true` | Require DoT. Fail if server doesn't support it. |
| `opportunistic` | Use DoT if available, fall back to plaintext. |
| `false` | Never use DoT. Plaintext only. |

```yaml
# Recommended: encrypt when possible
dns_over_tls: opportunistic

# Strict: requires DoT-capable servers
dns_over_tls: true
dns_servers:
  - 1.1.1.1#cloudflare-dns.com  # Server name for certificate validation
```

### dns_llmnr

Link-Local Multicast Name Resolution.

| Property | Value |
|----------|-------|
| Type | boolean |
| Default | `false` |

**Security warning**: LLMNR is vulnerable to credential harvesting attacks.[^1] Keep disabled unless you have a specific need.

```yaml
# Disabled by default for security
dns_llmnr: false

# Enable only if you understand the risks
dns_llmnr: true
```

### dns_mdns

Multicast DNS for `.local` domains.

| Property | Value |
|----------|-------|
| Type | boolean |
| Default | `false` |

Enables Bonjour/Avahi-style `.local` resolution. Useful for:
- Discovering printers
- Apple devices
- IoT devices using mDNS

```yaml
dns_mdns: true
```

### dns_validate_ips

Validate IP addresses before deployment.

| Property | Value |
|----------|-------|
| Type | boolean |
| Default | `true` |

Catches typos like `1.1.11` or `192.168.1.256` before they cause silent DNS failures.

```yaml
# Disable if using hostnames (for DoT SNI)
dns_validate_ips: false
```

## Caching

### dns_cache

Enable DNS response caching.

| Property | Value |
|----------|-------|
| Type | boolean |
| Default | `true` |

Caching significantly improves performance and reduces upstream load.

```yaml
dns_cache: true
```

### dns_stale_retention_sec

Serve stale cached records when upstream is unreachable.

| Property | Value |
|----------|-------|
| Type | integer |
| Default | `0` (disabled) |
| Requires | systemd 246+ |

When upstream DNS is temporarily unreachable, serve expired cache entries for this duration while retrying.

```yaml
# Serve stale records for up to 5 minutes during outages
dns_stale_retention_sec: 300
```

### dns_cache_from_localhost

Cache responses from localhost (127.0.0.0/8).

| Property | Value |
|----------|-------|
| Type | boolean |
| Default | `true` |
| Requires | systemd 248+ |

Relevant when running a local DNS server (Pi-hole, Unbound) that resolved queries.

```yaml
# Don't cache responses from local DNS server
dns_cache_from_localhost: false
```

## Stub Resolver

### dns_stub_listener

Enable stub listener on 127.0.0.53.

| Property | Value |
|----------|-------|
| Type | boolean |
| Default | `true` |

The stub listener allows traditional tools (dig, nslookup, host) to work via `/etc/resolv.conf`.

```yaml
dns_stub_listener: true
```

### dns_stub_port

Port for stub listener.

| Property | Value |
|----------|-------|
| Type | integer |
| Default | `53` |

Usually 53. Change only if port 53 conflicts with another service.

```yaml
dns_stub_port: 53
```

### dns_stub_listener_extra

Additional stub listener addresses.

| Property | Value |
|----------|-------|
| Type | list of strings |
| Default | `[]` |
| Requires | systemd 243+ |

Useful for:
- Containers that can't reach 127.0.0.53
- Bridge networks needing DNS on host IP
- Multi-homed hosts

```yaml
dns_stub_listener_extra:
  - "127.0.0.54"           # Another loopback
  - "192.168.1.1:5353"     # Host IP, custom port
  - "172.17.0.1"           # Docker bridge
```

## Hosts Management

### dns_hosts_entries

Custom /etc/hosts entries.

| Property | Value |
|----------|-------|
| Type | list of objects |
| Default | `[]` |

Each object:
- `ip`: IP address (required)
- `names`: List of hostnames (required)

Entries are managed in a block with markers. Existing entries outside the block are preserved.

```yaml
dns_hosts_entries:
  - ip: 10.0.0.1
    names:
      - gateway
      - gateway.home.lan
  - ip: 10.0.0.10
    names:
      - nas
      - nas.home.lan
      - storage
```

### dns_nsswitch_hosts

NSSwitch hosts resolution order.

| Property | Value |
|----------|-------|
| Type | string |
| Default | `files mymachines resolve [!UNAVAIL=return] myhostname` |

Controls name resolution order. See [design.md](design.md) for detailed explanation.

```yaml
# Default (recommended)
dns_nsswitch_hosts: "files mymachines resolve [!UNAVAIL=return] myhostname"

# Without container support
dns_nsswitch_hosts: "files resolve [!UNAVAIL=return] myhostname"
```

### dns_read_etc_hosts

Read /etc/hosts before DNS queries.

| Property | Value |
|----------|-------|
| Type | boolean |
| Default | `true` |

When `true`, systemd-resolved reads `/etc/hosts` and can answer queries from it directly.

```yaml
dns_read_etc_hosts: true
```

## Advanced Options

### dns_config_filename

Configuration filename in resolved.conf.d.

| Property | Value |
|----------|-------|
| Type | string |
| Default | `dns.conf` |

Deployed to `/etc/systemd/resolved.conf.d/{dns_config_filename}`.

```yaml
dns_config_filename: "dns.conf"
```

### dns_cleanup_old_configs

Remove orphaned config files.

| Property | Value |
|----------|-------|
| Type | boolean |
| Default | `true` |

When enabled, removes any `*.conf` files in `resolved.conf.d/` that don't match `dns_config_filename`.

```yaml
# Keep other configs (e.g., from cloud-init)
dns_cleanup_old_configs: false
```

### dns_custom_settings

Additional resolved.conf settings.

| Property | Value |
|----------|-------|
| Type | dictionary |
| Default | `{}` |

Key-value pairs added verbatim to `[Resolve]` section. For settings not exposed as dedicated variables.

```yaml
dns_custom_settings:
  ResolveUnicastSingleLabel: "yes"  # Resolve single-label names via DNS
  DNSStubListenerExtra: "172.17.0.1"  # Another way to add stub listener
```

See [resolved.conf(5)](https://www.freedesktop.org/software/systemd/man/resolved.conf.html) for all options.

## Feature Availability by systemd Version

Version information derived from the systemd NEWS file.[^2]

| Feature | Variable | Min Version |
|---------|----------|-------------|
| Basic DNS | all | 237 |
| DNSSEC, DoT | `dns_dnssec`, `dns_over_tls` | 237 |
| Extra stub listeners | `dns_stub_listener_extra` | 243 |
| Stale retention | `dns_stale_retention_sec` | 246 |
| Cache from localhost | `dns_cache_from_localhost` | 248 |

The role auto-detects systemd version and skips unsupported features with a warning.

## References

[^1]: MITRE ATT&CK T1557.001, "LLMNR/NBT-NS Poisoning and SMB Relay," https://attack.mitre.org/techniques/T1557/001/

[^2]: systemd NEWS file, https://github.com/systemd/systemd/blob/main/NEWS - Authoritative source for feature availability by version

[^3]: resolved.conf(5), https://www.freedesktop.org/software/systemd/man/resolved.conf.html - All configuration options
