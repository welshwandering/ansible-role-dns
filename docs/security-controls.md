# Security Controls Reference

This document maps the DNS role's security features to industry frameworks, helping security teams understand what threats are mitigated and what remains out of scope.

## MITRE ATT&CK Coverage

### Techniques Mitigated

| ATT&CK ID | Technique | Role Setting | How It Helps | Coverage |
|-----------|-----------|--------------|--------------|----------|
| [T1557.001](https://attack.mitre.org/techniques/T1557/001/) | LLMNR/NBT-NS Poisoning | `dns_llmnr: false` (default) | Disables LLMNR, preventing poisoning attacks that capture credentials | Full |
| [T1557](https://attack.mitre.org/techniques/T1557/) | Adversary-in-the-Middle | `dns_over_tls: true` | Encrypts DNS queries, preventing interception/modification in transit | Partial[^1] |
| [T1584.002](https://attack.mitre.org/techniques/T1584/002/) | Compromise Infrastructure: DNS Server | `dns_dnssec: true` | Validates DNS responses, detecting tampered records | Partial[^2] |
| [T1565.002](https://attack.mitre.org/techniques/T1565/002/) | Data Manipulation: Transmitted Data | `dns_over_tls: true` + `dns_dnssec: true` | Prevents modification of DNS responses in transit | Full (for DNS) |

[^1]: Partial because MITM at the DNS provider or via compromised CA is still possible. DoT protects the transport, not the endpoint.

[^2]: Partial because DNSSEC depends on the domain being signed. Many domains lack DNSSEC. `allow-downgrade` accepts unsigned responses.

### Techniques NOT Addressed

These require controls outside this role's scope:

| ATT&CK ID | Technique | Why Not Addressed | What To Do Instead |
|-----------|-----------|-------------------|-------------------|
| [T1071.004](https://attack.mitre.org/techniques/T1071/004/) | Application Layer Protocol: DNS | DNS tunneling/exfiltration requires content inspection | Use DNS firewall, Pi-hole, NextDNS with threat feeds |
| [T1568](https://attack.mitre.org/techniques/T1568/) | Dynamic Resolution (DGA) | Detecting algorithmically-generated domains requires threat intelligence | Use threat-blocking DNS (Quad9, NextDNS) |
| [T1583.001](https://attack.mitre.org/techniques/T1583/001/) | Acquire Infrastructure: Domains | Malicious domain registration is upstream | Use DNS filtering services |
| [T1048.003](https://attack.mitre.org/techniques/T1048/003/) | Exfiltration Over Alternative Protocol | DNS as exfil channel | Monitor DNS query patterns, use DNS firewall |

## CIS Controls Mapping

Mapping to [CIS Controls v8](https://www.cisecurity.org/controls):

| CIS Control | Sub-Control | Role Implementation | Notes |
|-------------|-------------|---------------------|-------|
| 4.2 | Establish and Maintain a Secure Configuration Process | Drop-in config, idempotent application | Role enforces consistent DNS config |
| 9.2 | Use DNS Filtering Services | `dns_servers` variable | Configure threat-blocking DNS providers |
| 12.1 | Ensure Network Infrastructure is Up-to-Date | Version detection, feature compatibility | Role adapts to systemd version |
| 13.6 | Collect DNS Query Audit Logs | Prometheus metrics export | Exports configuration state; query logging requires additional setup[^3] |

[^3]: systemd-resolved doesn't log individual queries by default. For query logging, use a local DNS server (Pi-hole, Unbound) or network-level capture.

## NIST 800-53 Mapping

Relevant controls from [NIST SP 800-53 Rev. 5](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final):

| Control ID | Control Name | Role Implementation |
|------------|--------------|---------------------|
| SC-8 | Transmission Confidentiality | `dns_over_tls: true` encrypts DNS queries |
| SC-8(1) | Cryptographic Protection | DoT uses TLS 1.2/1.3; DNSSEC uses cryptographic signatures |
| SC-20 | Secure Name/Address Resolution Service | DNSSEC validation, authenticated DNS providers |
| SC-21 | Secure Name/Address Resolution Service (Recursive) | Configures systemd-resolved as validating resolver |
| SC-22 | Architecture for Name Resolution | Centralizes DNS through systemd-resolved, consistent policy |
| CM-6 | Configuration Settings | Declarative Ansible configuration, idempotent application |
| CM-7 | Least Functionality | Disables LLMNR, mDNS by default |

## NSA/CISA Encrypted DNS Guidance

The role aligns with [NSA/CISA guidance on Encrypted DNS](https://media.defense.gov/2021/Jan/14/2002564889/-1/-1/0/CSI_ADOPTING_ENCRYPTED_DNS_U_OO_102904_21.PDF):

| Recommendation | Role Implementation | Default |
|----------------|---------------------|---------|
| Use encrypted DNS (DoT or DoH) | `dns_over_tls` variable | `opportunistic` |
| Use DNSSEC validation | `dns_dnssec` variable | `allow-downgrade` |
| Disable multicast DNS protocols | `dns_llmnr`, `dns_mdns` | Both `false` |
| Use enterprise DNS or trusted public resolvers | `dns_servers` variable | Configurable |
| Ensure DNS configuration is centrally managed | Ansible role deployment | Yes |

### Recommended Hardening (Per NSA/CISA)

```yaml
# NSA/CISA aligned configuration
dns_over_tls: true           # Require encrypted DNS
dns_dnssec: true             # Require DNSSEC validation
dns_llmnr: false             # Disable LLMNR (default)
dns_mdns: false              # Disable mDNS (default)
dns_servers:
  - 9.9.9.9#dns.quad9.net    # Threat-blocking, privacy-respecting
  - 1.1.1.1#cloudflare-dns.com
```

## Defense in Depth Position

This role operates at the **host resolver layer**. Understanding its position helps identify what other controls are needed:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NETWORK PERIMETER                                 │
│                 (Firewall, IDS/IPS, DNS Firewall)                   │
│                 ┌─────────────────────────────────────────────────┐ │
│                 │              NETWORK LAYER                       │ │
│                 │           (Encrypted transport)                  │ │
│                 │  ┌───────────────────────────────────────────┐  │ │
│                 │  │          HOST RESOLVER ◄── THIS ROLE      │  │ │
│                 │  │      (DNSSEC, DoT, policy)                 │  │ │
│                 │  │  ┌─────────────────────────────────────┐  │  │ │
│                 │  │  │         APPLICATION                 │  │  │ │
│                 │  │  │    (Browser, curl, etc.)            │  │  │ │
│                 │  │  └─────────────────────────────────────┘  │  │ │
│                 │  └───────────────────────────────────────────┘  │ │
│                 └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

| Layer | This Role's Contribution | Complementary Controls |
|-------|--------------------------|------------------------|
| Perimeter | None | DNS firewall, threat feeds, egress filtering |
| Network | Encrypted transport (DoT) | Network monitoring, flow analysis |
| Host Resolver | **DNSSEC validation, policy enforcement, secure defaults** | Endpoint detection, audit logging |
| Application | Consistent DNS behavior | Application-layer security |

## Threat Model Summary

### What This Role Protects Against

| Threat | Protection Mechanism | Confidence |
|--------|---------------------|------------|
| ISP/network DNS interception | DNS-over-TLS encryption | High (when `dns_over_tls: true`) |
| DNS response spoofing | DNSSEC validation | Medium (depends on domain signing) |
| LLMNR credential harvesting | Protocol disabled | High (default) |
| DNS cache poisoning | DNSSEC + systemd-resolved protections | High |
| Configuration drift | Idempotent Ansible deployment | High |

### What This Role Does NOT Protect Against

| Threat | Why | Mitigation |
|--------|-----|------------|
| Malicious DNS provider | Trust relationship, not technical control | Choose reputable providers, consider self-hosting |
| DNS tunneling/exfiltration | No content inspection | Use DNS firewall with DPI |
| Compromised CA (for DoT) | PKI trust model limitation | Certificate pinning (not supported by systemd-resolved) |
| Application-level DNS bypass | Apps can implement their own DNS | Network-level DNS enforcement |
| Queries to malicious domains | No threat intelligence | Use threat-blocking DNS (Quad9, NextDNS) |

## Security Configuration Profiles

### Minimal Security (Not Recommended)

```yaml
dns_dnssec: false
dns_over_tls: false
dns_llmnr: true
dns_mdns: true
```

**ATT&CK Coverage:** None

### Default Security (Role Defaults)

```yaml
dns_dnssec: allow-downgrade
dns_over_tls: opportunistic
dns_llmnr: false
dns_mdns: false
```

**ATT&CK Coverage:** T1557.001 (full), T1557 (partial), T1584.002 (partial)

### Enhanced Security

```yaml
dns_dnssec: true
dns_over_tls: true
dns_servers:
  - 1.1.1.1#cloudflare-dns.com
```

**ATT&CK Coverage:** T1557.001 (full), T1557 (full for DNS), T1584.002 (full for signed domains), T1565.002 (full for DNS)

### Maximum Security + Threat Protection

```yaml
dns_dnssec: true
dns_over_tls: true
dns_servers:
  - 9.9.9.9#dns.quad9.net      # Malware/phishing blocking
dns_llmnr: false
dns_mdns: false
```

Plus network-level controls:
- Block outbound UDP/53, TCP/53 (force DoT)
- Allow only TCP/853 to approved DNS servers
- Monitor for DNS-over-HTTPS bypass attempts

**ATT&CK Coverage:** T1557.001 (full), T1557 (full), T1584.002 (full), T1565.002 (full), T1568 (partial via threat feeds)

## Compliance Framework Applicability

This section details how the DNS role's controls map to specific compliance certifications and audit requirements. Use this to understand what compliance bars this role helps you meet.

### SOC 2 (Type I/II)

Relevant Trust Service Criteria:

| TSC | Criteria | Role Contribution | Evidence |
|-----|----------|-------------------|----------|
| CC6.1 | Logical access controls | Centralized DNS policy enforcement | Ansible configuration as code |
| CC6.6 | Transmission security | DNS-over-TLS encryption | `dns_over_tls` setting, DoT traffic in logs |
| CC6.7 | Protection during transmission | DNSSEC validation | `dns_dnssec` setting, validation metrics |
| CC7.1 | System configuration | Idempotent, auditable configuration | Git history, Molecule tests |

**Compliance bar met:** This role provides **supporting controls** for SOC 2. DNS encryption and validation contribute to the overall security posture but are not sufficient alone. SOC 2 requires organizational controls, access management, and monitoring beyond DNS configuration.

### ISO 27001:2022

Relevant Annex A controls:

| Control | Description | Role Contribution |
|---------|-------------|-------------------|
| A.5.15 | Access control | Disabling LLMNR prevents credential harvesting |
| A.8.20 | Networks security | Encrypted DNS transport (DoT) |
| A.8.21 | Security of network services | Secure DNS resolver configuration |
| A.8.24 | Use of cryptography | DNSSEC cryptographic validation |

**Compliance bar met:** Provides **technical implementation evidence** for these controls. ISO 27001 certification requires documented policies and procedures around these technical controls.

### FedRAMP (Moderate/High)

Based on NIST 800-53 controls (see above section for detailed mapping):

| Impact Level | Relevant Controls | Role Coverage |
|--------------|-------------------|---------------|
| Low | SC-8, SC-20 | Full |
| Moderate | SC-8, SC-8(1), SC-20, SC-21, SC-22, CM-6, CM-7 | Full |
| High | All Moderate + enhanced monitoring | Partial (requires additional logging) |

**Compliance bar met:** Role configuration addresses **technical control requirements** for FedRAMP Moderate. High baseline requires additional query logging and monitoring not included by default.

### PCI DSS v4.0

| Requirement | Description | Role Contribution |
|-------------|-------------|-------------------|
| 1.3.1 | Restrict inbound/outbound traffic | Configure `dns_servers` to approved providers only |
| 2.2.1 | Secure configuration standards | Ansible role provides consistent, auditable config |
| 4.2.1 | Strong cryptography for transmission | DNS-over-TLS encrypts queries |

**Compliance bar met:** Provides **defense-in-depth controls**. PCI DSS focuses on cardholder data; DNS security is a supporting control, not a primary requirement. Useful for demonstrating secure configuration practices.

### HIPAA Security Rule

| Standard | Implementation Spec | Role Contribution |
|----------|--------------------|--------------------|
| § 164.312(e)(1) | Transmission security | DNS-over-TLS encryption |
| § 164.312(e)(2)(ii) | Encryption | DoT uses TLS 1.2/1.3 |
| § 164.312(b) | Audit controls | Prometheus metrics, configuration as code |

**Compliance bar met:** **Addressable implementation specification** for transmission security. HIPAA requires risk assessment to determine if DNS encryption is appropriate for your environment. This role provides the technical capability.

### GDPR (Technical Measures)

| Article | Requirement | Role Contribution |
|---------|-------------|-------------------|
| Art. 32(1)(a) | Encryption of personal data in transit | DNS-over-TLS prevents ISP from logging queries |
| Art. 32(1)(b) | Confidentiality of processing systems | Encrypted DNS protects user browsing patterns |
| Art. 25 | Data protection by design | Privacy-preserving defaults (LLMNR disabled) |

**Compliance bar met:** Supports **privacy-by-design** requirements. Using DoT with privacy-respecting DNS providers (Cloudflare 1.1.1.1, Quad9) prevents third-party query logging. This is a technical measure supporting GDPR compliance.

### CMMC 2.0 (Cybersecurity Maturity Model Certification)

| Level | Practice | Role Contribution |
|-------|----------|-------------------|
| Level 1 | AC.L1-3.1.1 (Authorized access) | Disabling LLMNR prevents unauthorized credential access |
| Level 2 | SC.L2-3.13.8 (Cryptography) | DNS-over-TLS for encrypted transport |
| Level 2 | CM.L2-3.4.2 (Security config) | Ansible-managed baseline configuration |

**Compliance bar met:** Supports **Level 2 practices** related to cryptographic protection and secure configuration.

### Compliance Readiness Summary

| Framework | Role's Contribution | Additional Requirements |
|-----------|--------------------|-----------------------|
| **SOC 2** | Supporting technical controls | Policies, monitoring, access controls |
| **ISO 27001** | Implementation evidence for A.5.15, A.8.20-24 | ISMS documentation, risk assessment |
| **FedRAMP Moderate** | SC-8, SC-20, SC-21, SC-22, CM-6, CM-7 | POA&M, continuous monitoring |
| **FedRAMP High** | Same as Moderate | Enhanced logging, real-time monitoring |
| **PCI DSS** | Defense-in-depth, Req 2.2.1, 4.2.1 | CDE segmentation, full DSS implementation |
| **HIPAA** | Addressable transmission security | Risk assessment, BAAs |
| **GDPR** | Technical measure for Art. 32 | DPO, DPIA, organizational measures |
| **CMMC L2** | SC.L2-3.13.8, CM.L2-3.4.2 | Full CMMC assessment |

### Using This for Audit Evidence

When preparing for audits, document:

1. **Configuration baseline**: Git commit hash of Ansible configuration
2. **Test evidence**: Molecule test results showing controls are applied
3. **Runtime verification**: `resolvectl status` output showing active settings
4. **Metrics**: Prometheus exports showing control state over time

Example audit evidence package:
```bash
# Generate audit evidence
mkdir -p audit-evidence/dns
resolvectl status > audit-evidence/dns/resolver-status.txt
cat /etc/systemd/resolved.conf.d/*.conf > audit-evidence/dns/config.txt
git log -1 --format="%H %ai %s" -- ansible/roles/dns/ > audit-evidence/dns/config-version.txt
curl -s localhost:9100/metrics | grep ^dns_ > audit-evidence/dns/metrics.txt
```

## Audit Checklist

For security auditors reviewing systems configured with this role:

- [ ] Verify `dns_llmnr: false` (mitigates T1557.001)
- [ ] Verify `dns_over_tls` is `opportunistic` or `true` (mitigates T1557)
- [ ] Verify `dns_dnssec` is `allow-downgrade` or `true` (mitigates T1584.002)
- [ ] Verify DNS servers are reputable/approved providers
- [ ] Verify `/etc/resolv.conf` points to stub resolver
- [ ] Verify systemd-resolved is running and enabled
- [ ] Review Prometheus metrics for configuration state
- [ ] Check for unauthorized DNS bypass (applications using DoH directly)

```bash
# Quick audit commands
resolvectl status | grep -E "DNS|DNSSEC|DNSOverTLS"
grep ^hosts /etc/nsswitch.conf
ss -tlnp | grep ':53'  # Check for rogue DNS listeners
```

## References

- [MITRE ATT&CK](https://attack.mitre.org/) - Adversarial tactics and techniques
- [CIS Controls v8](https://www.cisecurity.org/controls) - Security best practices
- [NIST SP 800-53 Rev. 5](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final) - Security and privacy controls
- [NSA/CISA Encrypted DNS Guidance](https://media.defense.gov/2021/Jan/14/2002564889/-1/-1/0/CSI_ADOPTING_ENCRYPTED_DNS_U_OO_102904_21.PDF) - Adopting Encrypted DNS in Enterprise Environments
- [RFC 7858](https://datatracker.ietf.org/doc/html/rfc7858) - DNS over TLS specification
- [RFC 4033](https://datatracker.ietf.org/doc/html/rfc4033) - DNSSEC introduction
