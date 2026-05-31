# Security Policy

## Supported Versions

| Version | Supported | Security Updates |
|---------|-----------|------------------|
| 4.6.x (Butterfly Edition) | ✅ | Full support |
| 4.x (Legacy Abyssal) | ❌ | No updates |
| < 4.0 | ❌ | End of life |

## Reporting a Vulnerability

**You can report security vulnerabilities directly through GitHub Issues.**

I will receive an email notification immediately.

👉 **To report:**
1. Go to the [Issues tab](https://github.com/effjy/krakken-disk-butterfly/issues)
2. Click **"New Issue"**
3. Select **"Report a security vulnerability"** (if available) or use the blank template
4. Label your issue with **`security`** tag

**Alternative contact:** `effjy@protonmail.com` (encrypted preferred for critical issues)

### What to Include in Your Report

For a clear and actionable report, please include:

- **Description** of the vulnerability
- **Steps to reproduce** (code, commands, or proof of concept)
- **Potential impact** (what could an attacker do?)
- **Affected versions** (if known)
- **Suggested fix** (optional, but appreciated)

### What to Expect

| Step | Timeline |
|------|----------|
| Acknowledgment | Within 24-48 hours |
| Initial assessment | Within 7 days |
| Fix development | 14-30 days (depending on severity) |
| Coordinated release | After fix is verified |

### Severity Classification

- **🔴 Critical** - Remote code execution, key extraction, full volume compromise
- **🟠 High** - Privilege escalation, data leakage across volumes
- **🟡 Medium** - Denial of service, timing side-channels
- **🟢 Low** - Minor information disclosure, hardening issues

## Responsible Disclosure Guidelines

**Please DO:**
- Report vulnerabilities privately first (no public exploits)
- Give reasonable time to fix before public disclosure
- Use the issue tracker (I'll get email notifications)

**Please DON'T:**
- Exploit vulnerabilities for malicious purposes
- Access data that doesn't belong to you
- Disclose details publicly without coordination

## Security Best Practices for Krakken-Disk Users

### ⚠️ Critical Warnings

- **Encrypted swap is REQUIRED** - Run `sudo swapoff -a` or encrypt swap with LUKS
- **Never use weak passphrases** - Argon2id uses 1 GB RAM, but a weak password is still weak
- **Memory locking needs privileges** - Run with `sudo` or configure `ulimit -l unlimited`

### ✅ Recommended Configuration

```bash
# Disable unencrypted swap
sudo swapoff -a

# Run Krakken-Disk with memory locking
sudo krakken-disk

# Or configure user limits
echo "session required pam_limits.so" | sudo tee -a /etc/pam.d/common-session
echo "effjy soft memlock unlimited" | sudo tee -a /etc/security/limits.conf
