# SEC-LAb Ansible Project - Complete File Documentation

**Author:** Professional Ansible Documentation  
**Date:** April 15, 2026  
**Purpose:** This document explains every file in your Ansible security automation lab in a simple, professional way.

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Root Configuration Files](#root-configuration-files)
3. [Inventory Files](#inventory-files)
4. [Group Variables](#group-variables)
5. [Playbooks](#playbooks)
6. [Roles](#roles)
7. [Documentation Files](#documentation-files)

---

## 🎯 Project Overview

**What is this project?**  
This is a **Security Operations Lab** built with Ansible. It automatically scans, hardens, and validates the security of multiple systems:
- **Linux servers** (Ubuntu)
- **Windows machines** (Windows 10, Windows Server 2019)
- **Android phones** (via Termux)

**Why do we need this?**  
Instead of manually configuring security on each machine, Ansible automates everything. You run one command, and it secures all your devices according to industry standards (CIS, ANSSI).

---

## 🔧 Root Configuration Files

### 1. **ansible.cfg**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/ansible.cfg`

**What it does:**  
This is Ansible's main configuration file. It controls how Ansible behaves globally.

**Why we create it:**
- Sets default locations (inventory, roles, vault password)
- Configures SSH settings for better performance
- Disables strict SSH key checking (useful in lab environments)
- Enables SSH pipelining to make connections faster
- Points to the vault password file for automatic decryption

**Key Settings:**
- `inventory = ./inventory.ini` → Where to find your hosts
- `vault_password_file = ./.vault_pass` → Automatic password decryption
- `host_key_checking = False` → Don't ask to verify SSH fingerprints
- `pipelining = True` → Speed up SSH connections

---

### 2. **.vault_pass**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/.vault_pass`

**What it does:**  
Stores the password (`12345678`) used to encrypt/decrypt Ansible Vault files.

**Why we create it:**
- Ansible Vault encrypts sensitive data (passwords, credentials)
- This file lets Ansible automatically decrypt vault files without asking you
- **Security Note:** Never commit this file to Git! Keep it local only.

**Contains:**
```
12345678
```

---

## 📊 Inventory Files

### 3. **inventory.ini**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/inventory.ini`

**What it does:**  
Lists all the computers/devices you want to manage with Ansible.

**Why we create it:**
- Organizes devices into groups (phones, linux_servers, windows_machines)
- Defines the IP address for each device
- Ansible knows which machines to connect to

**Structure:**
```ini
[phones]           ← Group name
mine ansible_host=192.168.1.31   ← Device name and IP

[linux_servers]
ubuntu_srv ansible_host=192.168.8.112

[windows_machines]
win_test ansible_host=192.168.1.194
win_srv_2019 ansible_host=192.168.8.194
win_client_10 ansible_host=192.168.8.180
```

**Simple explanation:** Think of this as your "address book" for Ansible.

---

## 🔐 Group Variables

Group variables store settings that apply to specific groups of hosts.

### 4. **group_vars/all/vars.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/group_vars/all/vars.yml`

**What it does:**  
Defines default settings for **ALL** hosts in the inventory.

**Why we create it:**
- Avoids repeating the same settings for each device
- Sets the default username (`ansible`)
- Sets the default connection type (`ssh`)
- References the encrypted password from vault.yml

**Contains:**
- `ansible_user: ansible` → Login username
- `ansible_password: "{{ vault_ansible_password }}"` → Password from vault
- `ansible_connection: ssh` → Use SSH to connect

---

### 5. **group_vars/all/vault.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/group_vars/all/vault.yml`

**What it does:**  
Stores the **encrypted** password for the ansible user.

**Why we create it:**
- Passwords should never be stored in plain text
- Ansible Vault encrypts this file
- Only people with the vault password can read it

**Contains (when decrypted):**
```yaml
vault_ansible_password: "azerty@123"
```

**File appears encrypted:**
```
$ANSIBLE_VAULT;1.1;AES256
31633831316237623036323039666162666535663734633135356332626631...
```

---

### 6. **group_vars/linux_servers/vars.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/group_vars/linux_servers/vars.yml`

**What it does:**  
Settings specific to Linux servers only.

**Why we create it:**
- Linux servers need special settings (sudo access, python path)
- These settings override the defaults from `group_vars/all/`

**Key Settings:**
- `ansible_become: yes` → Enable sudo/root access
- `ansible_become_password` → Password for sudo (from vault)
- `ansible_python_interpreter: /usr/bin/python3` → Use Python 3

---

### 7. **group_vars/linux_servers/vault.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/group_vars/linux_servers/vault.yml`

**What it does:**  
Stores the **encrypted** sudo password for Linux servers.

**Why we create it:**
- Sudo password (root access) must be protected
- Encrypted for security

**Contains (when decrypted):**
```yaml
vault_become_password: "azerty@123"
```

---

### 8. **group_vars/phones/vars.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/group_vars/phones/vars.yml`

**What it does:**  
Settings specific to Android phones running Termux.

**Why we create it:**
- Termux uses a different SSH port (8022 instead of 22)
- Termux has a unique username (`u0_a338`)
- Python is in a different location on Android

**Key Settings:**
- `ansible_port: 8022` → Termux SSH port
- `ansible_user: u0_a338` → Termux user
- `ansible_python_interpreter: /data/data/com.termux/files/usr/bin/python3`

---

### 9. **group_vars/phones/vault.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/group_vars/phones/vault.yml`

**What it does:**  
Stores the **encrypted** password for phone access.

**Contains (when decrypted):**
```yaml
vault_phone_password: "azerty@123"
```

---

### 10. **group_vars/windows_machines.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/group_vars/windows_machines.yml`

**What it does:**  
Settings specific to Windows machines.

**Why we create it:**
- Windows uses Win32-OpenSSH (SSH on Windows)
- Windows needs PowerShell settings
- Different SSH port and shell type

**Key Settings:**
- `ansible_port: 22` → Windows OpenSSH uses port 22
- `ansible_shell_type: powershell` → Use PowerShell
- `ansible_connection: ssh` → Connect via SSH

---

## 📖 Playbooks

Playbooks are Ansible's "instruction manuals" - they tell Ansible what to do.

### 11. **playbooks/full_pipeline.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/full_pipeline.yml`

**What it does:**  
Runs the **complete security pipeline** from start to finish:
1. Check connectivity
2. Run baseline security scans
3. Apply hardening
4. Re-scan to verify improvements

**Why we create it:**
- One command runs everything (Jalon 1 → 2 → 3)
- Automates the entire security workflow
- Works on Linux and Windows

**Run command:**
```bash
ansible-playbook playbooks/full_pipeline.yml
```

---

### 12. **playbooks/baseline_scan.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/baseline_scan.yml`

**What it does:**  
Runs initial security scans **before** hardening.

**Why we create it:**
- Measures current security level (baseline)
- Creates HTML reports showing vulnerabilities
- Linux: Uses OpenSCAP scanner
- Windows: Uses HardeningKitty scanner

**Tools used:**
- **OpenSCAP** (Linux) → Industry-standard CIS/ANSSI checks
- **HardeningKitty** (Windows) → CIS Windows benchmarks

---

### 13. **playbooks/rescan.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/rescan.yml`

**What it does:**  
Runs security scans **after** hardening to verify improvements.

**Why we create it:**
- Proves that hardening worked
- Target: >85% compliance
- Compares before/after results

---

### 14. **playbooks/harden.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/harden.yml`

**What it does:**  
Applies security hardening to all systems.

**Why we create it:**
- Fixes security weaknesses found in baseline scans
- Applies CIS/ANSSI security standards
- Configures firewalls, disables unused services, strengthens passwords

---

### 15. **playbooks/linux_check.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/linux_check.yml`

**What it does:**  
Quick connectivity test for Linux servers only.

**Why we create it:**
- Tests SSH connection to Linux servers
- Verifies credentials work
- Fast preliminary check

---

### 16. **playbooks/linux_scan.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/linux_scan.yml`

**What it does:**  
Runs OpenSCAP security scans on Linux only.

---

### 17. **playbooks/linux_harden.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/linux_harden.yml`

**What it does:**  
Applies hardening to Linux servers only.

---

### 18. **playbooks/windows_check.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/windows_check.yml`

**What it does:**  
Quick connectivity test for Windows machines only.

---

### 19. **playbooks/windows_scan.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/windows_scan.yml`

**What it does:**  
Runs HardeningKitty security scans on Windows only.

---

### 20. **playbooks/windows_harden.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/windows_harden.yml`

**What it does:**  
Applies hardening to Windows machines only.

---

### 21. **playbooks/phone_check.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/phone_check.yml`

**What it does:**  
Quick connectivity test for Android phones only.

---

### 22. **playbooks/site.yml**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/playbooks/site.yml`

**What it does:**  
Master playbook that can run tasks across the entire site.

**Why we create it:**
- Convention in Ansible projects
- One playbook to manage everything
- Usually includes all other playbooks

---

## 🎭 Roles

Roles are reusable Ansible components that perform specific tasks.

### 23. **roles/common/**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/roles/common/`

**What it does:**  
Tests SSH connectivity to all hosts.

**Why we create it:**
- First step in any automation
- Confirms we can reach machines
- Uses `ansible.builtin.ping` module
- Works on Linux, Windows, and Android

**Files:**
- `tasks/main.yml` → Ping test and fallback raw command

---

### 24. **roles/linux_baseline/**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/roles/linux_baseline/`

**What it does:**  
Verifies Linux server connectivity and basic setup.

**Why we create it:**
- Linux-specific preliminary checks
- Ensures Python and SSH work correctly

---

### 25. **roles/linux_hardening/**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/roles/linux_hardening/`

**What it does:**  
Applies comprehensive security hardening to Linux servers.

**Why we create it:**
- Implements CIS and ANSSI security benchmarks
- Modifies system configurations for better security

**Components:**
- `tasks/main.yml` → Main task file
- `tasks/auth.yml` → Password policies, account security
- `tasks/banners.yml` → Login warnings
- `tasks/filesystem.yml` → Partition security, permissions
- `tasks/logging.yml` → Audit logging setup
- `tasks/network.yml` → Network security settings
- `tasks/permissions.yml` → File permission hardening
- `tasks/services.yml` → Disable unnecessary services
- `tasks/ssh.yml` → Secure SSH configuration
- `tasks/sysctl.yml` → Kernel security parameters
- `defaults/main.yml` → Default variables
- `handlers/main.yml` → Service restart handlers

---

### 26. **roles/openscap_scan/**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/roles/openscap_scan/`

**What it does:**  
Runs OpenSCAP security scanner on Linux (baseline scan).

**Why we create it:**
- Industry-standard security compliance tool
- Generates HTML reports
- Checks CIS and ANSSI benchmarks

---

### 27. **roles/openscap_rescan/**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/roles/openscap_rescan/`

**What it does:**  
Runs OpenSCAP scanner again after hardening (verification scan).

**Why we create it:**
- Proves hardening was effective
- Compares before/after security scores

---

### 28. **roles/windows_baseline/**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/roles/windows_baseline/`

**What it does:**  
Verifies Windows machine connectivity and basic setup.

---

### 29. **roles/windows_hardening/**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/roles/windows_hardening/`

**What it does:**  
Applies security hardening to Windows machines.

**Components:**
- `tasks/main.yml` → Main task file
- `tasks/account_policy.yml` → Password policies
- `tasks/audit_policy.yml` → Security auditing
- `tasks/firewall.yml` → Windows Firewall rules
- `tasks/network.yml` → Network security
- `tasks/registry.yml` → Registry security settings
- `tasks/services.yml` → Service hardening
- `tasks/user_rights.yml` → User privilege assignments

---

### 30. **roles/hardeningkitty_scan/**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/roles/hardeningkitty_scan/`

**What it does:**  
Runs HardeningKitty security scanner on Windows (baseline scan).

**Why we create it:**
- PowerShell-based security scanner
- CIS Windows benchmarks
- Generates compliance reports

---

### 31. **roles/hardeningkitty_rescan/**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/roles/hardeningkitty_rescan/`

**What it does:**  
Runs HardeningKitty scanner after hardening (verification scan).

---

### 32. **roles/phone_check/**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/roles/phone_check/`

**What it does:**  
Verifies Android phone connectivity via Termux SSH.

**Why we create it:**
- Android/Termux has unique requirements
- Tests SSH on non-standard port (8022)

---

## 📚 Documentation Files

### 33. **docs/PROJECT.md**
**Location:** `/home/el_aoutmanie/Desktop/SEC-LAb/docs/PROJECT.md`

**What it does:**  
Project documentation, architecture, and credentials reference.

**Why we create it:**
- Documents the project structure
- Contains inventory information
- Lists credentials (for lab environment)
- Explains the security pipeline

---

## 🔑 Key Concepts Explained Simply

### What is Ansible Vault?
Think of it as a safe for your passwords. Instead of storing passwords in plain text, Ansible Vault encrypts them. You need one master password (`.vault_pass`) to unlock all the vaults.

### What are Group Variables?
Different types of computers need different settings:
- Linux needs sudo passwords
- Windows uses PowerShell
- Android phones use port 8022

Group variables let you organize these settings by group instead of repeating them for each machine.

### What is a Playbook?
A playbook is a step-by-step instruction manual for Ansible. It says:
1. Connect to these computers
2. Run these tasks
3. In this order

### What is a Role?
A role is a reusable package of tasks. Instead of writing the same hardening tasks in every playbook, you create one `linux_hardening` role and reuse it everywhere.

### Why Separate Scan and Hardening?
**Scan → Harden → Rescan**

1. **Baseline scan** shows current security (e.g., 45% compliant)
2. **Hardening** fixes the problems
3. **Rescan** proves it worked (e.g., 92% compliant)

This proves the value of your security work!

---

## 🚀 Quick Start Guide

### Test Connectivity
```bash
ansible-playbook playbooks/linux_check.yml
ansible-playbook playbooks/windows_check.yml
ansible-playbook playbooks/phone_check.yml
```

### Run Baseline Scans
```bash
ansible-playbook playbooks/baseline_scan.yml
```

### Apply Hardening
```bash
ansible-playbook playbooks/harden.yml
```

### Verify Results
```bash
ansible-playbook playbooks/rescan.yml
```

### Run Everything
```bash
ansible-playbook playbooks/full_pipeline.yml
```

---

## 📁 File Organization Summary

```
SEC-LAb/
├── ansible.cfg                    # Ansible configuration
├── .vault_pass                    # Vault password file
├── inventory.ini                  # List of all machines
├── group_vars/                    # Variables by group
│   ├── all/                       # Settings for ALL machines
│   │   ├── vars.yml              # Common variables
│   │   └── vault.yml             # Encrypted password
│   ├── linux_servers/            # Settings for Linux
│   │   ├── vars.yml
│   │   └── vault.yml
│   ├── phones/                   # Settings for Android
│   │   ├── vars.yml
│   │   └── vault.yml
│   └── windows_machines.yml      # Settings for Windows
├── playbooks/                    # Ansible instructions
│   ├── full_pipeline.yml        # Run everything
│   ├── baseline_scan.yml         # Initial scans
│   ├── harden.yml               # Apply hardening
│   ├── rescan.yml               # Verify results
│   └── ... (specific playbooks)
├── roles/                        # Reusable components
│   ├── common/                   # Connectivity test
│   ├── linux_hardening/         # Linux security
│   ├── windows_hardening/       # Windows security
│   ├── openscap_scan/           # Linux scanner
│   └── hardeningkitty_scan/     # Windows scanner
└── docs/
    └── PROJECT.md               # Project documentation
```

---

## ✅ Security Best Practices

1. **Never commit `.vault_pass` to Git**  
   Add it to `.gitignore`

2. **Use strong vault passwords**  
   Current: `12345678` (acceptable for lab, not production)

3. **Keep vaults encrypted**  
   Always re-encrypt after editing:
   ```bash
   ansible-vault encrypt group_vars/all/vault.yml
   ```

4. **Test before production**  
   Always test playbooks on non-critical systems first

5. **Backup before hardening**  
   Hardening changes system configurations - backup first!

---

## 🎓 Learning Resources

- **Ansible Documentation:** https://docs.ansible.com/
- **CIS Benchmarks:** https://www.cisecurity.org/cis-benchmarks/
- **OpenSCAP:** https://www.open-scap.org/
- **HardeningKitty:** https://github.com/scipag/HardeningKitty

---

## 📞 Need Help?

Each file has comments explaining its purpose. Look for:
```yaml
# ============================================================
# [Explanation of what this file does]
# ============================================================
```

---

**Document Version:** 1.0  
**Last Updated:** April 15, 2026  
**Project:** SEC-Ops Lab Security Automation

---

*This documentation is designed to be beginner-friendly while maintaining professional standards. Each file serves a specific purpose in the automated security pipeline.*
