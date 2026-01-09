# Agent Documentation

This file provides context and instructions for AI agents (and humans) working on the `ansible-role-dns` repository.

## Repository Overview
- **Role**: `welshwandering.dns`
- **Purpose**: Configure `systemd-resolved` with DNSSEC, DoT, and split-horizon DNS.
- **Key Philosophy**: Secure defaults, systemd-native, comprehensive distribution support.

## Supported Platforms
Refer to [DISTRIBUTIONS.md](DISTRIBUTIONS.md) for the authoritative list of supported versions.
- **Debian**: 11, 12, 13, 14
- **Ubuntu**: 22.04, 24.04, 25.10
- **Rocky Linux**: 8, 9, 10
- **Fedora**: 43

## Development Standards
1.  **Idempotence**: All tasks must be idempotent.
2.  **Linting**: Code must pass `ansible-lint` without errors.
3.  **Testing**: New features must be tested via Molecule.
    - Docker images are provided by `geerlingguy`.
    - RHEL-based tests require explicitly installing Python (handled in `converge.yml`).
4.  **NetworkManager**: On RHEL systems, `NetworkManager` is the primary renderer. We configure it to use `systemd-resolved` as the backend.

## Release Process
1.  Update `CHANGELOG.md`.
2.  Update `meta/main.yml` platforms.
3.  Update `README.md` version matrix.
4.  Tag the release on GitHub.
