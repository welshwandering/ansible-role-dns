# Design

This document explains how the DNS role works internally, the design decisions behind it, and the patterns we value.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        APPLICATIONS                                  │
│                  dig, curl, apt, docker, etc.                       │
├──────────────────────────┬──────────────────────────────────────────┤
│                          │                                          │
│    /etc/resolv.conf ─────┼──► 127.0.0.53:53 (stub listener)        │
│    (symlink)             │                                          │
│                          ▼                                          │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │                    systemd-resolved                           │ │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │ │
│   │  │  Cache   │  │  DNSSEC  │  │   DoT    │  │ Link Manager │ │ │
│   │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │ │
│   └──────────────────────────────────────────────────────────────┘ │
│                          │                                          │
│                          ▼                                          │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │                    Upstream DNS                               │ │
│   │         Primary ──► Fallback ──► Per-link (split DNS)        │ │
│   └──────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

## Task Execution Flow

The role executes tasks in a deliberate order:

```
1. validate.yml     Detect systemd version, verify prerequisites
         │
         ▼
2. configure.yml    Deploy resolved.conf.d drop-in, manage resolv.conf symlink
         │
         ▼
3. link-dns.yml     Apply per-interface DNS (runtime only)
         │
         ▼
4. nsswitch.yml     Configure name resolution order
         │
         ▼
5. hosts.yml        Manage /etc/hosts entries
         │
         ▼
6. service.yml      Enable/start systemd-resolved
         │
         ▼
7. verify.yml       Test DNS resolution, validate configuration
         │
         ▼
8. metrics.yml      Export Prometheus metrics
```

### Why This Order?

1. **Validate first**: Fail fast if systemd-resolved isn't available or is masked. No point configuring a broken system.

2. **Configure before service**: Drop-in configs are read on service start. If we start first, we'd need an extra restart.

3. **Per-link DNS after config**: `resolvectl` commands need the service running. We apply global config first, then per-interface.

4. **NSSwitch after resolved**: The `resolve` NSS module needs systemd-resolved running.

5. **Verify after everything**: Only meaningful after full configuration is applied.

6. **Metrics last**: Captures the final state, including verification results.

## Key Design Decisions

### 1. Drop-in Configuration Over Main File

We deploy to `/etc/systemd/resolved.conf.d/dns.conf` (configurable) rather than modifying `/etc/systemd/resolved.conf`.[^1]

**Why:**
- Non-destructive: preserves distribution defaults
- Clear ownership: easy to identify Ansible-managed config
- Clean removal: delete one file to undo everything
- Package upgrades: dpkg won't prompt about modified conffiles

### 2. Symlink Strategy for resolv.conf

We force `/etc/resolv.conf` to be a symlink to `/run/systemd/resolve/stub-resolv.conf`.[^2]

**Why stub-resolv.conf specifically:**
- Contains `nameserver 127.0.0.53` pointing to the stub listener
- Traditional tools (dig, nslookup, host) work normally
- Alternative `/run/systemd/resolve/resolv.conf` bypasses caching[^2]

**What about existing resolv.conf:**
- We back it up first (if it's a file, not symlink)
- We remove conflicting symlinks (e.g., to NetworkManager's version)
- This is aggressive but intentional—we own DNS now

### 3. Per-Link DNS is Runtime Only

The `dns_link_dns` configuration uses `resolvectl` commands, which don't survive reboots.[^8]

**Why not persistent:**
- Per-link DNS belongs in the network configuration (`.network` files)
- The `network` role handles persistent per-link DNS
- This role focuses on resolved configuration, not network topology
- Runtime config is useful for testing and one-off scenarios

**For persistence:** Use the `network` role or configure systemd-networkd `.network` files directly.

### 4. NSSwitch Comes With Opinions

Default: `files mymachines resolve [!UNAVAIL=return] myhostname`[^4]

**Breaking it down:**
- `files`: Check `/etc/hosts` first (localhost, custom entries)
- `mymachines`: Container/VM name resolution (systemd-machined)
- `resolve`: systemd-resolved for DNS[^3]
- `[!UNAVAIL=return]`: If resolved is running, stop here (don't fall through to other methods)[^4]
- `myhostname`: Fallback for the local hostname if nothing else works

**Why `[!UNAVAIL=return]`:**[^3]
- If systemd-resolved is running, it's authoritative
- Prevents double-lookups through other NSS modules
- Avoids weird edge cases with mDNS/LLMNR fallback

### 5. Secure Defaults, Not Secure Absolutism

We default to `allow-downgrade` for DNSSEC[^7] and `opportunistic` for DoT.[^6]

**The pragmatic choice:**
- Many domains have broken DNSSEC (misconfigured DS records, expired signatures)
- Many networks block DoT (corporate firewalls, captive portals)
- Strict modes break real-world usage
- Users who need strict can enable it

**The security-conscious can enable:**
```yaml
dns_dnssec: true
dns_over_tls: true
```

But they should understand the implications (some sites will fail).

### 6. Version Detection is Runtime

We detect systemd version at runtime rather than hardcoding per-distro.[^5]

**Why:**
- Distributions upgrade systemd independently of their release cycle
- Users may have backported packages
- Rolling releases (Debian sid, Ubuntu devel) change constantly
- Feature availability is a function of systemd version, not distro version[^5]

**The mechanism:**
```yaml
- name: Get systemd version
  command: systemctl --version
  register: systemd_version_output

- name: Parse systemd version
  set_fact:
    dns_systemd_version: "{{ systemd_version_output.stdout_lines[0] | regex_search('systemd ([0-9]+)', '\\1') | first | int }}"
```

### 7. Cleanup Old Configs by Default

When `dns_config_filename` changes (or on first run), we remove orphaned configs.

**Why:**
- Prevents config accumulation from experimentation
- Drop-in directory should have one authoritative config
- Makes role behavior predictable

**Disable if needed:** `dns_cleanup_old_configs: false`

### 8. IP Validation is Opt-Out

We validate IP addresses before deployment by default.

**Why:**
- Typos in DNS IPs cause silent failures
- Invalid IPs waste time debugging
- Validation is cheap, DNS outages are expensive

**Disable if needed:** `dns_validate_ips: false` (e.g., if using hostnames for DoT)

## Patterns We Value

### Idempotency

Every task can run multiple times with the same result:
- `lineinfile` for NSSwitch (replaces existing line)
- `blockinfile` for /etc/hosts (managed block)
- `template` for config (replaces file)
- Symlink creation is idempotent by definition

### Explicit Over Implicit

- Every default is documented in `defaults/main.yml`
- No hidden magic—what you see is what you get
- Warnings for version-incompatible features

### Fail Fast

- Validation runs first
- Clear error messages for common issues
- Pre-flight checks for masked services

### Observable

- Prometheus metrics for every boolean setting
- Systemd version exported as metric
- Verification tasks confirm actual state

### Minimal Disruption

- Backup before modifying system files
- Handler-based restarts (only when needed)
- No unnecessary service restarts

## File Ownership

Files managed by this role:

| Path | Ownership | Notes |
|------|-----------|-------|
| `/etc/systemd/resolved.conf.d/dns.conf` | Created | Main config (filename configurable) |
| `/etc/resolv.conf` | Symlink forced | Points to stub-resolv.conf |
| `/etc/nsswitch.conf` | Line modified | Only `hosts:` line |
| `/etc/hosts` | Block added | Managed block with markers |
| `/var/lib/node_exporter/textfile_collector/dns.prom` | Created | Metrics |

## What This Role Doesn't Do

- **Network configuration**: Use a network role (e.g., systemd-networkd) for interfaces, bonds, VLANs
- **Persistent per-link DNS**: Configure in `.network` files or via your network role
- **DNS server operation**: This configures the client/resolver, not a DNS server
- **DHCP/RA DNS configuration**: That's network stack territory
- **Container DNS**: Docker/Podman have their own DNS resolution

## References

[^1]: systemd resolved.conf(5), "Configuration file for DNS stub resolver," https://www.freedesktop.org/software/systemd/man/resolved.conf.html - Documents drop-in directory behavior and all configuration options

[^2]: systemd-resolved(8), "Network Name Resolution Manager," https://www.freedesktop.org/software/systemd/man/systemd-resolved.html - Describes stub resolver, caching, and DNS resolution behavior

[^3]: nss-resolve(8), "Hostname resolution via systemd-resolved," https://www.freedesktop.org/software/systemd/man/nss-resolve.html - Documents NSS module behavior and `[!UNAVAIL=return]` action

[^4]: nsswitch.conf(5), "System Databases and Name Service Switch configuration file," https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html - NSSwitch configuration format and action specifiers

[^5]: systemd NEWS file, https://github.com/systemd/systemd/blob/main/NEWS - Authoritative source for feature availability by version

[^6]: RFC 7858, "Specification for DNS over Transport Layer Security (TLS)," https://datatracker.ietf.org/doc/html/rfc7858 - DNS-over-TLS specification

[^7]: RFC 4033, "DNS Security Introduction and Requirements," https://datatracker.ietf.org/doc/html/rfc4033 - DNSSEC specification

[^8]: resolvectl(1), "Resolve domain names, IPaddresses, DNS resource records, and services," https://www.freedesktop.org/software/systemd/man/resolvectl.html - Per-link DNS configuration via resolvectl
