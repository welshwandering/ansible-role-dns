# Changelog

All notable changes to the DNS role will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.0.0] - 2024-12-05

### Added

- Initial public release
- **Core Features**
  - systemd-resolved configuration with drop-in files
  - DNSSEC validation (strict, allow-downgrade, disabled modes)
  - DNS-over-TLS (strict, opportunistic, disabled modes)
  - DNS caching with stale record support (systemd 246+)
  - Cache from localhost control (systemd 248+)
- **DNS Server Configuration**
  - Primary and fallback DNS servers
  - Search domains
  - Per-link DNS with routing domains (split-horizon DNS)
  - Runtime per-link configuration via resolvectl
- **Stub Resolver**
  - Standard stub listener on 127.0.0.53
  - Extra stub listeners for containers/multi-homed hosts (systemd 243+)
  - Configurable stub port
- **Security**
  - LLMNR disabled by default (credential harvesting mitigation)
  - mDNS disabled by default (attack surface reduction)
  - IP address validation before deployment
- **Name Resolution**
  - NSSwitch hosts line configuration
  - Custom /etc/hosts entries via blockinfile
  - resolv.conf symlink management
- **Observability**
  - Prometheus metrics export for all settings
  - systemd version exported as metric
  - Verification tasks with status output
- **Operational**
  - Enable/disable toggle for entire role
  - Old config cleanup (prevents accumulation)
  - systemd version detection with feature gating
  - Idempotent operation
- **Documentation**
  - Comprehensive README with quick start
  - Design document with architecture decisions
  - Configuration reference for all variables
  - Usage guide with 17 scenarios
  - Security documentation with citations
  - Security controls mapping (MITRE ATT&CK, CIS, NIST)
  - Advanced usage and integration patterns
  - Troubleshooting guide
  - Testing documentation
- **Testing**
  - Molecule test suite with default and disabled scenarios
  - 22 verification checks
  - Platform coverage: Debian 12, Ubuntu 24.04

### Platform Support

- Debian 11 (Bullseye) - systemd 247, limited features
- Debian 12 (Bookworm) - systemd 252, full features
- Debian 13 (Trixie) - systemd 256, full features
- Ubuntu 22.04 (Jammy) - systemd 249, limited features
- Ubuntu 24.04 (Noble) - systemd 255, full features

### Dependencies

- Ansible 2.15+
- `ansible.utils` collection

[Unreleased]: https://github.com/haven/ansible-role-dns/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/haven/ansible-role-dns/releases/tag/v1.0.0
