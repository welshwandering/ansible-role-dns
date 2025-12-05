# Testing

This role includes comprehensive Molecule tests covering multiple platforms and scenarios.

## Test Coverage Summary

| Category | Tests | Verified |
|----------|-------|----------|
| Service state | 3 | Running, enabled, resolvectl available |
| Symlink | 2 | Exists, points to stub-resolv.conf |
| Configuration | 6 | File exists, permissions, DNS/fallback/DNSSEC/DoT/custom settings |
| NSSwitch | 1 | Hosts line contains `resolve` |
| /etc/hosts | 3 | Custom entries, managed block markers |
| DNS resolution | 1 | localhost resolves |
| Metrics | 3 | File exists, content, systemd version |
| Disabled mode | 3 | No config created, no metrics, clean exit |

**Total: 22 verification checks**

## Test Scenarios

### Default Scenario

Tests full role functionality with representative configuration.

**Platforms tested:**
- Debian 11 (Bullseye) - systemd 247
- Debian 12 (Bookworm) - systemd 252
- Debian 13 (Trixie) - systemd 256
- Debian sid - latest
- Ubuntu 22.04 (Jammy) - systemd 249
- Ubuntu 24.04 (Noble) - systemd 255
- Ubuntu devel - latest

**Configuration tested:**
```yaml
dns_enabled: true
dns_resolved_enabled: true
dns_servers: [1.1.1.1, 8.8.8.8]
dns_fallback_servers: [9.9.9.9]
dns_dnssec: allow-downgrade
dns_over_tls: opportunistic
dns_cache: true
dns_stale_retention_sec: 60
dns_stub_listener_extra: ["127.0.0.54"]
dns_hosts_entries:
  - ip: 10.0.0.1
    names: [testgateway, testgateway.example.com]
  - ip: 10.0.0.2
    names: [testdns]
dns_custom_settings:
  ResolveUnicastSingleLabel: "yes"
```

**Verifications:**
1. systemd-resolved running and enabled
2. resolvectl command available
3. /etc/resolv.conf symlinked to stub-resolv.conf
4. Configuration file exists with correct permissions (0644)
5. DNS servers in config
6. Fallback DNS in config
7. DNSSEC setting correct
8. DNS-over-TLS setting correct
9. Custom settings applied
10. NSSwitch hosts line configured
11. Custom /etc/hosts entries present
12. Managed block markers in /etc/hosts
13. DNS resolution works (localhost)
14. Metrics file exists
15. Metrics content correct
16. Systemd version exported in metrics

### Disabled Scenario

Tests clean behavior when `dns_enabled: false`.

**Platform tested:**
- Debian 12

**Configuration tested:**
```yaml
dns_enabled: false
```

**Verifications:**
1. Role completes without error
2. No configuration file created
3. No metrics file created
4. System state unchanged

## Running Tests

### Prerequisites

```bash
# Install Molecule and Docker driver
pip install molecule molecule-plugins[docker]

# Verify Docker is running
docker ps
```

### Run All Tests

```bash
cd ansible/roles/dns

# Full test sequence (all scenarios)
molecule test --all

# Or run specific scenario
molecule test                    # default scenario
molecule test -s disabled        # disabled scenario
```

### Test Sequence

The default test sequence:
1. `dependency` - Install role dependencies
2. `cleanup` - Clean any previous test state
3. `destroy` - Remove any existing containers
4. `syntax` - Verify Ansible syntax
5. `create` - Create test containers
6. `prepare` - (if defined) Prepare containers
7. `converge` - Apply the role
8. `idempotence` - Apply again, verify no changes
9. `verify` - Run verification tests
10. `cleanup` - Clean up
11. `destroy` - Remove containers

### Development Workflow

```bash
# Create containers and apply role (for debugging)
molecule converge

# Run verification tests only
molecule verify

# Log into a container for manual inspection
molecule login -h debian-12

# View logs
molecule logs

# Destroy when done
molecule destroy
```

### Idempotence Testing

Molecule automatically tests idempotence. After `converge`, it runs the role again and fails if any tasks report changes.

To test idempotence manually:
```bash
molecule converge
molecule converge  # Should show 0 changed
```

## Platform-Specific Notes

### Older systemd Versions (Debian 11, Ubuntu 22.04)

These platforms have systemd < 250 and lack:
- `StaleRetentionSec`
- `CacheFromLocalhost`

The test configuration sets these to safe values:
```yaml
# molecule/default/molecule.yml
host_vars:
  debian-11:
    dns_stale_retention_sec: 0
    dns_cache_negative_ttl: 0
```

The role auto-detects and skips unsupported features.

### Privileged Containers

Tests require privileged Docker containers with systemd:
```yaml
platforms:
  - name: debian-12
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    command: /lib/systemd/systemd
```

This is necessary because:
- systemd-resolved requires systemd
- Managing services requires cgroup access
- /etc/resolv.conf symlink requires proper permissions

## Adding New Tests

### Adding a Verification Check

Edit `molecule/default/verify.yml`:

```yaml
- name: Verify new feature works
  ansible.builtin.shell: your_verification_command
  register: new_check
  changed_when: false
  failed_when: new_check.rc != 0
```

### Adding a New Scenario

1. Create directory: `molecule/new-scenario/`
2. Add `molecule.yml`:
   ```yaml
   driver:
     name: docker
   platforms:
     - name: test-platform
       image: debian:12
       # ... container config
   provisioner:
     name: ansible
   scenario:
     name: new-scenario
   ```
3. Add `converge.yml` with role configuration
4. Add `verify.yml` with verification tasks

### Testing New Features

1. Add feature to `converge.yml` configuration
2. Add verification in `verify.yml`
3. Run tests:
   ```bash
   molecule test
   ```

## CI Integration

This role supports two CI modes:

1. **Standalone mode**: When published as a separate repository, uses `.github/workflows/` in this role
2. **Monorepo mode**: When part of a larger repository, the parent repo's CI reads `.ci.yml` for configuration

### CI Configuration (.ci.yml)

The `.ci.yml` file at the role root defines test parameters:

```yaml
# .ci.yml
role: dns
version: 1.0.0

molecule:
  scenarios:
    - default
    - disabled
  platforms:
    - debian12
    - ubuntu2404
  parallel: true
  fail_fast: false

validators:
  ansible_lint: true
  yamllint: true
  jinja_lint: true

timeout: 30
```

This file serves as:
- Machine-readable test specification
- Input for monorepo CI orchestration
- Documentation of test coverage

### Standalone GitHub Actions

When published separately, the role includes complete workflows:

```
.github/workflows/
├── test.yml      # Lint + Molecule tests on push/PR
└── release.yml   # Version tagging and Galaxy publishing
```

**Test workflow features:**
- Linting (yamllint, ansible-lint)
- Matrix testing (scenarios × platforms)
- Security scanning (Trivy)
- Artifact collection on failure
- Automatic Galaxy publishing on version tags

### Monorepo Integration

When part of a larger repository:

1. **Parent repo discovers roles** by finding `.ci.yml` files:
   ```yaml
   # In parent CI workflow
   - name: Discover testable roles
     run: |
       find ansible/roles -name ".ci.yml" | \
         xargs -I{} dirname {} | \
         sort -u
   ```

2. **Reads test configuration** from each role's `.ci.yml`:
   ```yaml
   - name: Parse role CI config
     run: |
       yq -r '.molecule.scenarios[]' ansible/roles/dns/.ci.yml
   ```

3. **Runs tests selectively** based on changed files:
   ```yaml
   # Only test changed roles (PR)
   # Test all roles (main branch push or CI changes)
   ```

### GitHub Actions (Standalone)

```yaml
name: Molecule Test
on: [push, pull_request]

jobs:
  molecule:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        scenario: [default, disabled]
        distro: [debian12, ubuntu2404]
        exclude:
          - scenario: disabled
            distro: ubuntu2404
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install ansible molecule molecule-plugins[docker]
          ansible-galaxy collection install ansible.utils

      - name: Run Molecule
        run: molecule test -s ${{ matrix.scenario }}
        env:
          PY_COLORS: '1'
          ANSIBLE_FORCE_COLOR: '1'
          MOLECULE_DISTRO: ${{ matrix.distro }}
```

### GitLab CI Example

```yaml
molecule:
  image: python:3.11
  services:
    - docker:dind
  variables:
    DOCKER_HOST: tcp://docker:2375
  parallel:
    matrix:
      - SCENARIO: [default, disabled]
        DISTRO: [debian12, ubuntu2404]
  script:
    - pip install molecule molecule-plugins[docker]
    - ansible-galaxy collection install ansible.utils
    - molecule test -s $SCENARIO
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
```

### Release Workflow

The release workflow (`.github/workflows/release.yml`) handles:

1. **Version validation**
   - Checks CHANGELOG.md has entry for version
   - Validates meta/main.yml has required fields

2. **Full test suite**
   - Runs all scenarios on all platforms

3. **GitHub Release**
   - Creates release with changelog extract
   - Tags the commit

4. **Galaxy Publishing**
   - Publishes to Ansible Galaxy
   - Requires `GALAXY_API_KEY` secret

**Triggering a release:**
```bash
# Create and push version tag
git tag v1.0.0
git push origin v1.0.0

# Or use workflow dispatch with version input
```

## Troubleshooting Tests

### Container Won't Start

```bash
# Check Docker
docker ps
systemctl status docker

# Try creating manually
molecule create
molecule login -h debian-12
```

### Verification Failing

```bash
# Run converge to apply role
molecule converge

# Login and inspect
molecule login -h debian-12

# Inside container:
systemctl status systemd-resolved
cat /etc/systemd/resolved.conf.d/*.conf
ls -la /etc/resolv.conf
```

### Idempotence Failing

A task is changing on the second run. Common causes:
- Timestamp in template
- `changed_when: true` without condition
- Command without state check

```bash
# Run with diff to see what changed
molecule converge -- --diff
```

## Test File Reference

```
molecule/
├── default/
│   ├── molecule.yml      # Test configuration (platforms, driver)
│   ├── converge.yml      # Role application with test variables
│   └── verify.yml        # Verification tasks
└── disabled/
    ├── molecule.yml
    ├── converge.yml
    └── verify.yml
```
