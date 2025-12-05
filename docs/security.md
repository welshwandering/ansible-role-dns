# Security Guide

How to use this role for privacy and security, including preventing ISP DNS snooping.

## Threat Models

### ISP DNS Surveillance

**Threat:** Your ISP can see every DNS query you make when using unencrypted DNS.

**Why this happens:** Traditional DNS uses unencrypted UDP on port 53. When your ISP provides DNS (the default via DHCP), they can log every query.[^1] Even when using third-party DNS, unencrypted queries traverse ISP infrastructure and can be inspected.

```
Your Device ──► ISP Network (unencrypted UDP/53) ──► DNS Server
                     │
                     └── Queries visible in transit
```

### DNS Spoofing/Hijacking

**Threat:** Without validation, DNS responses can be modified in transit or at the resolver.[^2]

### LLMNR/mDNS Credential Harvesting

**Threat:** LLMNR and mDNS can be exploited for credential theft on local networks using tools like Responder.[^3]

## High-Security Configuration

### Maximum Privacy (Encrypted DNS)

```yaml
- hosts: all
  roles:
    - role: dns
      vars:
        # Require encrypted DNS
        dns_over_tls: true

        # Privacy-focused providers with DoT
        dns_servers:
          - 1.1.1.1#cloudflare-dns.com
          - 9.9.9.9#dns.quad9.net

        # Fallback also encrypted
        dns_fallback_servers:
          - 1.0.0.1#cloudflare-dns.com
          - 149.112.112.112#dns.quad9.net

        # Validate responses
        dns_dnssec: true

        # Disable insecure discovery protocols
        dns_llmnr: false
        dns_mdns: false
```

**What this achieves:**

| Layer | Protection | Setting |
|-------|------------|---------|
| Transport | Queries encrypted via TLS | `dns_over_tls: true` |
| Authentication | Server certificate validated | `#hostname` suffix |
| Integrity | DNSSEC signature validation | `dns_dnssec: true` |
| Local network | No broadcast queries | `dns_llmnr: false`, `dns_mdns: false` |

### How DNS-over-TLS Works

```
┌─────────────────────────────────────────────────────────────────────┐
│                          YOUR DEVICE                                 │
│                                                                      │
│    Application ──► systemd-resolved ──► TLS connection (port 853)   │
│                                              │                       │
└──────────────────────────────────────────────┼───────────────────────┘
                                               │ Encrypted (TLS 1.2/1.3)
                                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       NETWORK PATH                                   │
│                                                                      │
│    Observer sees: TLS connection to 1.1.1.1:853                     │
│    Observer cannot see: Query contents                              │
│                                                                      │
└──────────────────────────────────────────────┼───────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      DNS PROVIDER                                    │
│                                                                      │
│    Decrypts query, resolves, encrypts response                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

DNS-over-TLS is specified in RFC 7858.[^4]

### Provider Privacy Policies

Review each provider's privacy policy before choosing:

| Provider | DoT Address | DoT Hostname | Privacy Policy |
|----------|-------------|--------------|----------------|
| Cloudflare | 1.1.1.1 | cloudflare-dns.com | [Privacy Policy][^5] |
| Quad9 | 9.9.9.9 | dns.quad9.net | [Privacy Policy][^6] |
| Google | 8.8.8.8 | dns.google | [Privacy Policy][^7] |
| Mullvad | 194.242.2.2 | dns.mullvad.net | [Privacy Policy][^8] |
| NextDNS | 45.90.28.0 | {profile}.dns.nextdns.io | [Privacy Policy][^9] |

**Note:** Each provider has different data retention and logging practices. Read their policies to understand what data they collect and retain.

### The `#hostname` Syntax Explained

```yaml
dns_servers:
  - 1.1.1.1#cloudflare-dns.com
```

The `#hostname` suffix provides:

1. **Server Name Indication (SNI):** Indicates which certificate the server should present
2. **Certificate Validation:** systemd-resolved verifies the certificate matches the hostname[^10]

Without it, systemd-resolved cannot validate the server's identity:
```yaml
dns_servers:
  - 1.1.1.1  # No certificate hostname validation
```

## Limitations

### DoT Port Blocking

Some networks block port 853 (DoT). If this occurs:

1. **Detect:** Check logs for TLS connection failures
   ```bash
   journalctl -u systemd-resolved | grep -i tls
   ```

2. **Options:**
   - Use `opportunistic` mode to fall back to unencrypted
   - Use DNS-over-HTTPS via a local proxy (cloudflared, dnscrypt-proxy)
   - Use a VPN

### IP Address Visibility

Encrypted DNS hides query contents but not destination IPs:
```
DNS query for "example.com" → Encrypted (hidden)
HTTPS connection to 93.184.216.34 → Visible
```

The destination IP often reveals the site. A VPN hides this from your ISP.

### DNS Provider Trust

You must trust your chosen DNS provider. Encryption protects queries in transit but the provider sees them.

### Metadata Analysis

Query timing and patterns may be observable even when encrypted.

## Security Hardening Checklist

### Default Security (Role Defaults)

```yaml
dns_dnssec: allow-downgrade  # Validates when available
dns_over_tls: opportunistic  # Encrypts when available
dns_llmnr: false             # Disabled
dns_mdns: false              # Disabled
```

### Enhanced Security

```yaml
dns_dnssec: true             # Strict validation (may break some domains)
dns_over_tls: true           # Require encryption (fails if blocked)
dns_servers:
  - 1.1.1.1#cloudflare-dns.com  # With hostname validation
```

## Testing Your Configuration

### Verify DoT is Active

```bash
resolvectl status | grep DNSOverTLS
# Should show: DNSOverTLS=yes
```

### Verify DNSSEC is Working

```bash
# This domain has valid DNSSEC
resolvectl query --validate sigok.verteiltesysteme.net

# This domain has intentionally broken DNSSEC (should fail)
resolvectl query --validate sigfail.verteiltesysteme.net
```

### Check Configured DNS Servers

```bash
resolvectl status
# Verify only your intended servers are listed
```

### External Verification

These sites can verify your DNS configuration:
- Cloudflare's diagnostic: https://1.1.1.1/help
- DNS leak tests (search for "DNS leak test")

## Security vs. Usability Trade-offs

| Setting | Security Benefit | Usability Impact |
|---------|------------------|------------------|
| `dns_dnssec: true` | Validates all responses | Some misconfigured domains fail |
| `dns_dnssec: allow-downgrade` | Validates when possible | Works with all domains |
| `dns_over_tls: true` | Requires encryption | Fails if port 853 blocked |
| `dns_over_tls: opportunistic` | Encrypts when possible | Falls back to unencrypted |

**Recommendation:** Start with `opportunistic` and `allow-downgrade`, test in your environment, then consider stricter settings.

## References

[^1]: EFF, "The Case for Banning ISPs from Selling Your Internet Browsing History," https://www.eff.org/deeplinks/2017/03/case-banning-isps-selling-your-internet-browsing-history - Documents ISP DNS logging practices

[^2]: RFC 4033, "DNS Security Introduction and Requirements," https://datatracker.ietf.org/doc/html/rfc4033 - DNSSEC specification addressing DNS spoofing

[^3]: MITRE ATT&CK T1557.001, "LLMNR/NBT-NS Poisoning and SMB Relay," https://attack.mitre.org/techniques/T1557/001/ - Documents LLMNR attack techniques

[^4]: RFC 7858, "Specification for DNS over Transport Layer Security (TLS)," https://datatracker.ietf.org/doc/html/rfc7858

[^5]: Cloudflare 1.1.1.1 Privacy Policy, https://developers.cloudflare.com/1.1.1.1/privacy/public-dns-resolver/

[^6]: Quad9 Privacy Policy, https://www.quad9.net/privacy/policy/

[^7]: Google Public DNS Privacy Policy, https://developers.google.com/speed/public-dns/privacy

[^8]: Mullvad DNS, https://mullvad.net/en/help/dns-over-https-and-dns-over-tls/

[^9]: NextDNS Privacy Policy, https://nextdns.io/privacy

[^10]: systemd-resolved documentation on DNS server specification, https://www.freedesktop.org/software/systemd/man/resolved.conf.html - See DNS= option for hostname syntax
