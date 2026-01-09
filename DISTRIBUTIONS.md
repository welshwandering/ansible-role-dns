# Supported Distributions

The following distributions are supported and regularly tested:

| Distribution | Version | Details | Test Status |
|--------------|---------|---------|-------------|
| **Debian** | 14 (Forky) | Testing/Unstable | ✅ Verified |
| **Debian** | 13 (Trixie) | Stable | ✅ Verified |
| **Debian** | 12 (Bookworm) | Oldstable | ✅ Verified |
| **Debian** | 11 (Bullseye) | LTS | ✅ Verified |
| **Ubuntu** | 25.10 (Plucky) | Interim Release | ⚠️ Pending Image |
| **Ubuntu** | 24.04 (Noble) | LTS | ✅ Verified |
| **Ubuntu** | 22.04 (Jammy) | LTS | ✅ Verified |
| **Rocky Linux** | 10 | Current | ✅ Verified |
| **Rocky Linux** | 9 | Maintenance Support | ✅ Verified |
| **Rocky Linux** | 8 | Maintenance Support | ✅ Verified |
| **Fedora** | 43 | Current | ✅ Verified |

## Architecture Support
- **amd64** (x86_64): Primary support
- **arm64** (aarch64): Secondary support (via emulation)

## Testing Strategy
All distributions are tested using Molecule with Docker containers.
- **Debian**: Official images or `geerlingguy` derivatives.
- **Ubuntu**: Official images or `geerlingguy` derivatives.
- **Rocky/Fedora**: Official images or `geerlingguy` derivatives.
