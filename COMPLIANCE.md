# Security and Compliance

## Overview
This role is designed to meet high security standards for DNS resolution on Linux systems. It leverages `systemd-resolved` to implement modern cryptographic protections and attack surface reduction techniques, mapping directly to major compliance frameworks.

## Compliance Control Mapping

| Framework | Control | Requirement | Role Feature |
| :--- | :--- | :--- | :--- |
| **NIST 800-53 Rev 5** | **SC-20**, **SC-21** | Secure Name / Address Resolution Service | **DNSSEC**: Validates integrity and authenticity of DNS records. |
| **NIST 800-53 Rev 5** | **SC-8** | Transmission Confidentiality and Integrity | **DNS-over-TLS (DoT)**: Encrypts DNS queries to prevent eavesdropping and manipulation. |
| **NIST 800-53 Rev 5** | **CM-7** | Least Functionality | **Attack Surface Reduction**: Disables legacy protocols (LLMNR) and mDNS on non-required interfaces. |
| **NIST 800-53 Rev 5** | **AC-4** | Information Flow Enforcement | **Split-Horizon DNS**: Routes internal queries to internal servers and external queries to public resolvers via per-link routing domains. |
| **PCI DSS 4.0** | **4.2.1** | Strong cryptography for data in transit | **DNS-over-TLS (DoT)**: Uses TLS 1.2/1.3 for DNS traffic encryption. |
| **PCI DSS 4.0** | **1.3.2** | Limit inbound/outbound traffic to what is necessary | **Attack Surface Reduction**: Disables unnecessary listening services (LLMNR/mDNS) by default. |
| **CIS Linux** | **3.x** | Network Configuration | **systemd-networkd / NetworkManager**: Integrates with system-level network managers for consistent configuration. |

## Security Features

### 1. DNSSEC (Data Integrity)
- **Default**: `allow-downgrade` (Balanced)
- **Strict Mode**: Set `dns_dnssec: "yes"` to drop non-signed responses (may break some domains).
- **Protection**: Ensures that DNS responses utilize the "Chain of Trust" to verify they have not been tampered with.

### 2. DNS-over-TLS (Privacy)
- **Default**: `opportunistic` (Balanced)
- **Strict Mode**: Set `dns_over_tls: "yes"` to require TLS for all queries.
- **Protection**: Encrypts the entire DNS transaction, preventing ISPs or attackers on the wire from seeing which domains are being requested.

### 3. Attack Surface Reduction
- **LLMNR**: Disabled globally by default (`dns_llmnr: "no"`). Legacy protocol vulnerable to poisoning.
- **mDNS**: Disabled globally by default (`dns_mdns: "no"`). Only enabled on specific trusted interfaces if explicitly configured.

### 4. Split-Horizon DNS
- **Function**: Allows specific domains to be routed to private DNS servers (e.g., VPN links) while the rest of the traffic goes to public resolvers.
- **Benefit**: Prevents internal domain names from leaking to public DNS resolvers.

## Verification
To verify the security posture:
```bash
# Check DNSSEC and DoT status
resolvectl status

# Verify listening ports (should be minimal/none for stub listener on direct interfaces)
ss -tulpn | grep 5355  # LLMNR (Should be empty)
ss -tulpn | grep 5353  # mDNS (Should be empty unless enabled)
```
