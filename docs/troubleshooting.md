# Troubleshooting

Diagnostic commands, common issues, and solutions.

## Diagnostic Commands

### Quick Health Check

```bash
# Is systemd-resolved running?
systemctl is-active systemd-resolved

# Full service status
systemctl status systemd-resolved

# Current DNS configuration
resolvectl status

# DNS statistics
resolvectl statistics
```

### Detailed Diagnostics

```bash
# View resolved logs
journalctl -u systemd-resolved -f

# Check resolv.conf symlink
ls -la /etc/resolv.conf

# View configuration files
cat /etc/systemd/resolved.conf.d/*.conf

# Check NSSwitch
grep ^hosts /etc/nsswitch.conf

# List per-link DNS
resolvectl dns
resolvectl domain
```

### Test DNS Resolution

```bash
# Basic query via systemd-resolved
resolvectl query google.com

# Query specific record type
resolvectl query -t AAAA google.com

# Test DNSSEC validation
resolvectl query --validate cloudflare.com

# Query via traditional tools (uses stub resolver)
dig +short google.com
host google.com
nslookup google.com
```

## Common Issues

### DNS Not Working After Role Run

**Symptoms:** No DNS resolution, `host: not found` errors

**Check resolv.conf symlink:**
```bash
ls -la /etc/resolv.conf
```

**Expected:**
```
/etc/resolv.conf -> /run/systemd/resolve/stub-resolv.conf
```

**If wrong or not a symlink:**
```bash
# Manual fix
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

**If stub-resolv.conf doesn't exist:**
```bash
# systemd-resolved not running
sudo systemctl start systemd-resolved
```

---

### "systemd-resolved is masked"

**Symptoms:** Role fails with "systemd-resolved is masked" error

**Fix:**
```bash
sudo systemctl unmask systemd-resolved
sudo systemctl enable --now systemd-resolved
```

**Then re-run the role.**

---

### DNSSEC Validation Failures

**Symptoms:** Some domains fail to resolve, SERVFAIL errors

**Diagnose:**
```bash
# Check if DNSSEC is the problem
resolvectl query --validate failing-domain.com
```

**If DNSSEC validation fails:**
```yaml
# Option 1: Use allow-downgrade (recommended)
dns_dnssec: allow-downgrade

# Option 2: Disable DNSSEC (less secure)
dns_dnssec: false
```

**Common causes:**
- Domain has misconfigured DNSSEC (expired signatures, wrong DS records)
- Upstream DNS server strips DNSSEC records
- Network middlebox interfering

---

### DNS-over-TLS Failures

**Symptoms:** DNS works with `dns_over_tls: false` but fails with `true`

**Diagnose:**
```bash
# Check if DoT connection works
resolvectl query google.com
journalctl -u systemd-resolved | grep -i tls
```

**Common causes:**

1. **Network blocks port 853:**
   ```yaml
   dns_over_tls: opportunistic  # Fall back to port 53
   ```

2. **Server doesn't support DoT:**
   ```yaml
   # Use known DoT servers
   dns_servers:
     - 1.1.1.1#cloudflare-dns.com
     - 9.9.9.9#dns.quad9.net
   ```

3. **Certificate validation fails:**
   ```yaml
   # Include server hostname for SNI
   dns_servers:
     - 1.1.1.1#cloudflare-dns.com
   ```

---

### Per-Link DNS Not Working

**Symptoms:** Domain routing doesn't work, queries go to wrong server

**Check current per-link config:**
```bash
resolvectl dns
resolvectl domain
resolvectl status
```

**Common issues:**

1. **Interface doesn't exist:**
   ```bash
   ip link show
   # Verify interface name matches dns_link_dns configuration
   ```

2. **Routing domain missing `~` prefix:**
   ```yaml
   # Wrong: becomes search domain only
   domains:
     - corp.example.com

   # Correct: routing domain
   domains:
     - "~corp.example.com"
   ```

3. **Configuration didn't survive reboot:**
   ```
   Per-link DNS via resolvectl is runtime only.
   For persistence, use systemd-networkd .network files.
   ```

---

### Stub Listener Port Conflict

**Symptoms:** systemd-resolved fails to start, port 53 in use

**Diagnose:**
```bash
sudo ss -tlnp | grep ':53'
```

**Common culprits:**
- dnsmasq (often from libvirt)
- Pi-hole
- BIND
- Unbound

**Solutions:**

1. **Disable the conflicting service:**
   ```bash
   sudo systemctl stop dnsmasq
   sudo systemctl disable dnsmasq
   ```

2. **Use different port for other service:**
   ```yaml
   # Keep systemd-resolved on 53, move other service
   ```

3. **Disable stub listener (use resolved directly):**
   ```yaml
   dns_stub_listener: false
   ```
   Then update other services to query resolved directly at `/run/systemd/resolve/resolv.conf`.

---

### NSSwitch Configuration Issues

**Symptoms:** Some name resolution methods don't work, `/etc/hosts` ignored

**Check NSSwitch:**
```bash
grep ^hosts /etc/nsswitch.conf
```

**Expected:**
```
hosts:          files mymachines resolve [!UNAVAIL=return] myhostname
```

**Common issues:**

1. **`resolve` missing:**
   - Re-run the role
   - Or manually add `resolve` to hosts line

2. **`files` not first:**
   - `/etc/hosts` won't be checked first
   - Reorder to: `files ... resolve ...`

3. **Missing `[!UNAVAIL=return]`:**
   - May fall through to other methods unexpectedly

---

### Slow DNS Resolution

**Symptoms:** DNS queries take several seconds

**Diagnose:**
```bash
# Time a query
time resolvectl query google.com

# Check for timeout patterns
journalctl -u systemd-resolved | grep -i timeout
```

**Common causes:**

1. **IPv6 queries timing out (no IPv6 connectivity):**
   ```yaml
   # Add to dns_custom_settings if no IPv6
   dns_custom_settings:
     ResolveUnicastSingleLabel: "no"
   ```

2. **Slow upstream DNS server:**
   ```yaml
   # Switch to faster DNS
   dns_servers:
     - 1.1.1.1  # Cloudflare is typically fastest
   ```

3. **DoT connection overhead:**
   ```bash
   # First query is slow (TLS handshake), subsequent queries should be fast
   # If all queries are slow, check for DoT issues
   ```

---

### Docker Container DNS Issues

**Symptoms:** Containers can't resolve DNS

**Check Docker DNS config:**
```bash
docker run --rm alpine cat /etc/resolv.conf
```

**If pointing to 127.0.0.53:**

Containers can't reach host's 127.0.0.53 (different network namespace).

**Solution 1: Expose stub listener on Docker bridge:**
```yaml
dns_stub_listener_extra:
  - "172.17.0.1"  # Docker default bridge
```

**Solution 2: Configure Docker to use external DNS:**
```json
// /etc/docker/daemon.json
{
  "dns": ["1.1.1.1", "8.8.8.8"]
}
```

---

### Cache Issues

**Symptoms:** Stale DNS records, changes not reflecting

**Flush the cache:**
```bash
resolvectl flush-caches
```

**Check cache statistics:**
```bash
resolvectl statistics
```

**For persistent issues:**
```yaml
# Reduce cache effectiveness (not recommended)
dns_cache: false
```

---

### Role Validation Failures

**"Invalid DNS server address":**
```bash
# Check your IP addresses for typos
dns_servers:
  - 1.1.1.1    # Valid
  - 1.1.11     # Invalid (missing octet)
  - 1.1.1.256  # Invalid (octet > 255)
```

**"systemd-resolved is not available":**
```bash
# Install systemd-resolved
sudo apt install systemd-resolved

# Or on minimal systems
sudo apt install systemd
```

## Debug Mode

Run Ansible with increased verbosity:

```bash
ansible-playbook -vvv playbooks/platform.yml -l hostname --tags dns
```

Check task-by-task:

```bash
ansible-playbook playbooks/platform.yml -l hostname --tags dns --step
```

## Collecting Diagnostics

For bug reports, collect:

```bash
# System info
uname -a
cat /etc/os-release
systemctl --version

# DNS status
resolvectl status
resolvectl statistics
cat /etc/systemd/resolved.conf.d/*.conf
ls -la /etc/resolv.conf
grep ^hosts /etc/nsswitch.conf

# Logs
journalctl -u systemd-resolved --since "1 hour ago"
```

## Recovery Procedures

### Full Reset

If DNS is completely broken:

```bash
# Stop resolved
sudo systemctl stop systemd-resolved

# Restore resolv.conf temporarily
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf

# Now you have DNS to troubleshoot

# Clean up resolved configs
sudo rm -f /etc/systemd/resolved.conf.d/*.conf

# Restart resolved
sudo systemctl start systemd-resolved

# Restore symlink
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

### Rollback to DHCP DNS

```yaml
# Minimal config - let DHCP handle everything
- hosts: servers
  roles:
    - role: dns
      vars:
        dns_servers: []
        dns_fallback_servers:
          - 1.1.1.1
```

## References

For authoritative documentation on systemd-resolved behavior:

- [systemd-resolved(8)](https://www.freedesktop.org/software/systemd/man/systemd-resolved.html) - Service behavior and stub resolver
- [resolved.conf(5)](https://www.freedesktop.org/software/systemd/man/resolved.conf.html) - Configuration options
- [resolvectl(1)](https://www.freedesktop.org/software/systemd/man/resolvectl.html) - Query and control commands
- [nss-resolve(8)](https://www.freedesktop.org/software/systemd/man/nss-resolve.html) - NSS module behavior
