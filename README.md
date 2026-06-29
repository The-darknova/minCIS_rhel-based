---

This project provides a production-grade, highly modular, and idempotent Ansible playbook designed to automatically harden RHEL-based Linux systems. It drastically reduces the attack surface, implements mandatory access controls, hardens the kernel and filesystem, and enforces strict authentication policies based on industry-standard security guidelines.

## 🚨 Caution(s) 🚨
Executing this playbook will apply deep system modifications that may interfere with existing services or standard system behaviors. Please note that this is exclusively a **remediation tool**, designed to be deployed only *after* a proper security audit has been performed.

- **Thoroughly Test Before Deployment:** It is strongly recommended to apply these changes in a sterile staging or testing environment prior to any production roll-out.
- **Assumes a Clean Installation:** These roles were specifically engineered and tested against a fresh installation of RHEL. If you are applying them to a pre-existing environment, manually review the configurations beforehand to accommodate any site-specific operational requirements.
- **Not for EOL Systems:** This procedure is not valid for End-Of-Life (EOL) systems.

---

## 🏗 Architecture
The playbook follows a modular Role-Based architecture, making it easy to turn specific hardening features on or off as needed.

### Directory Structure
```text
minCIS_rhel-based/
├── Rhel based hardening [v3].md # Comprehensive manual hardening guide
├── inventory.ini                # Server inventory targeting your environments
├── site.yml                     # Master playbook defining role execution order
├── group_vars/
│   └── all.yml                  # Global feature toggles for all hardening metrics
└── roles/                       # Modular hardening roles
    ├── access_control/          # SELinux enforcement, bad pkgs removal, Banners
    ├── auth_ssh/                # SSH, PAM policies (authselect), Sudo restrictions
    ├── common_handlers/         # Global handlers (Reboot, SSH restart, Firewalld)
    ├── crypto_policies/         # System-wide cryptography enforcement
    ├── filesystem/              # Secure mount options and FS blacklisting
    ├── kernel_boot/             # Kernel sysctl, coredumps, GRUB permissions, Auditd
    ├── network_firewall/        # Network sysctl, Firewalld, exotic protocols removal
    ├── package_mgmt/            # DNF security and legacy service purging
    └── time_sync/               # Time synchronization (Chrony)
```

### Execution Flow (`site.yml`)
The main playbook targets the `test_nodes` group and executes the following roles sequentially:

1. **`common_handlers`**: Consolidates global handlers (e.g., `Restart SSH`, `Reboot System`, `Reload firewalld`) invoked by multiple roles.

2. **`filesystem`**:
   - Blacklists obsolete and high-risk filesystem modules (cramfs, jffs2, hfs, usb-storage, etc.).
   - Enforces strict mount options (`nodev`, `nosuid`, `noexec`) on sensitive partitions (`/tmp`, `/var`, `/home`, etc.).

3. **`package_mgmt`**:
   - Disables weak DNF dependencies (`install_weak_deps=False`).
   - Secures DNF repository file permissions.
   - Purges legacy network services, graphical interfaces (GDM3/X11), and unnecessary daemons.

4. **`time_sync`**:
   - Installs and configures **Chrony** for accurate network time synchronization.

5. **`access_control`**:
   - Forces **SELinux** policies to `enforcing` and `targeted`.
   - Removes vulnerable SELinux utilities (`mcstrans`, `setroubleshoot`).
   - Deploys legal warning banners (`/etc/issue`, `/etc/motd`).

6. **`kernel_boot`**:
   - Secures `/boot/grub2/grub.cfg` permissions to prevent local tampering.
   - Mitigates SELinux bypasses via `grubby` parameters (`selinux=0`, `enforcing=0`).
   - Applies strict Kernel Sysctl hardening (enables ASLR, restricts dmesg, protects symlinks/hardlinks).
   - Disables core dumps and installs/configures **Auditd** and **Journald**.

7. **`crypto_policies`**:
   - Modernizes system cryptography by applying `DEFAULT:NO-SHA1:NO-SSHCBC`.

8. **`network_firewall`**:
   - Blacklists exotic networking protocols (dccp, tipc, rds, sctp).
   - Secures the TCP/IP stack via Sysctl (disables IP forwarding, enables SYN cookies, logs martians).
   - Installs and configures **Firewalld** with a strict default deny-all (in/out/routed) policy, explicitly allowing only inbound SSH and outbound Web/DNS traffic via rich rules.

9. **`auth_ssh`**:
   - Heavily restricts SSH daemon access (`PermitRootLogin no`, `X11Forwarding no`, secure ciphers).
   - Implements strict **PAM** password complexity, history, and explicit lockout profiles using `authselect` (sssd, faillock, pwhistory).
   - Enforces strict file permissions on critical system authentication files (`shadow`, `gshadow`, `shells`, `opasswd`).
   - Enables detailed Sudo auditing and secures user profiles (TMOUT).

---

## ⚙️ Configuration & Toggles
Configuration is managed globally through `group_vars/all.yml` and can be overridden per host group in `inventory.ini`. Every hardening metric corresponds to a toggle variable, allowing highly granular control:

```yaml
# Examples of toggles in group_vars/all.yml:
hardening_fs_blacklist: true
hardening_selinux: true
hardening_firewalld: true
hardening_ssh: true
hardening_pam_authselect: true
hardening_time_sync: false
hardening_auditd: false
hardening_crypto_policies: false
```

If a configuration breaks an application, you can simply change its corresponding toggle to `false` and re-run the playbook to bypass that specific restriction without disabling the entire role.

---

## 🚀 Setup and Execution Instructions
> [!NOTE]
> For more in-depth information, architectural concepts, and manual step-by-step equivalents, please refer to the comprehensive [Rhel based hardening [v3].md](Rhel%20based%20hardening%20[v3].md) guide included in this repository.

### 1. Environment and Policy Customization
Before running the playbook, you must align the configuration with your specific environment and company policies:
- **Inventory Settings:** Edit the `inventory.ini` file to specify your target nodes. You can separate servers into logical groups (`[test_nodes]`, `[production_nodes]`) and apply variable overrides.
- **Legal Warning Banner:** Edit the `motd` and `issue` banner text within `roles/access_control/tasks/main.yml` to match your company's official legal login warning policy.

### 2. Pre-flight Security Verification
> [!WARNING]
> **SSH Keys Recommended!** The `auth_ssh` role heavily restricts root login (`PermitRootLogin no`) but leaves `PasswordAuthentication yes` intentionally to respect the baseline RHEL auth configuration. **You MUST ensure you have a standard user with `sudo` privileges configured or an SSH Key registered before running this playbook.** If you do not, you risk locking yourself out permanently if password attempts fail.
> 
> **Test Environment First!** These changes drastically limit system capabilities. Certain non-standard applications and legacy protocols will break. **Always apply the playbook to `[test_nodes]` first** so you can tweak the toggles in `group_vars/all.yml` before touching production.

### 3. Securing Credentials (Ansible Vault)
Your `inventory.ini` shouldn't store plain-text credentials (`ansible_ssh_pass`, `ansible_become_pass`). Use Ansible Vault to encrypt them.
Create an encrypted file to hold your sensitive variables:

```bash
ansible-vault create group_vars/all_secrets.yml
```

Inside, add your passwords:

```yaml
ansible_ssh_pass: "your_secure_password"
ansible_become_pass: "your_secure_password"
```

Remove the plain-text passwords from `inventory.ini` if you had them there.

### 4. Running the Playbook
To deploy the hardening configuration across all RHEL servers using your Vault password:

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

### 5. Post-Execution
Certain kernel parameters, GRUB changes, Crypto policies, and filesystem mount configurations require a full reboot. After a successful Ansible run, it is highly recommended to restart the target servers. The playbook may attempt to orchestrate a reboot via handlers automatically if deep system modifications were detected.

---

***Authored by [The-darknova]***
