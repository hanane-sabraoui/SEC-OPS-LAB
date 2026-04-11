# SEC-OPS LAB — Project Documentation

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Lab Infrastructure](#2-lab-infrastructure)
3. [Directory Structure](#3-directory-structure)
4. [File-by-File Explanation](#4-file-by-file-explanation)
5. [How SSH Connectivity Works](#5-how-ssh-connectivity-works)
6. [Running the Playbooks](#6-running-the-playbooks)
7. [Jalon 2 — Baseline Compliance Scanning](#7-jalon-2--baseline-compliance-scanning)
8. [Jalon Roadmap](#8-jalon-roadmap)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Project Overview

**SEC-OPS LAB** is a 4-week technical project that automates the **hardening and auditing** of a mixed IT infrastructure (Linux + Windows + Android) using Ansible, targeting a minimum **85% compliance score** against **ANSSI** and **CIS** security benchmarks.

| Tool | Purpose |
|------|---------|
| **Ansible** | Configuration management & automation engine |
| **OpenSCAP** | Linux compliance scanning (ANSSI/CIS profiles) |
| **HardeningKitty** | Windows compliance scanning (CIS benchmarks) |
| **SSH** | Sole transport protocol for all target machines |

---

## 2. Lab Infrastructure

All machines are in an **isolated, air-gapped** network.

| Host | OS | IP Placeholder | SSH Port | User | Notes |
|------|----|-----------------|----------|------|-------|
| `ubuntu_srv` | Ubuntu Server 22.04 | `<UBUNTU_IP>` | 22 | ansible | sudo enabled |
| `win_srv_2019` | Windows Server 2019 | `<WIN_2019_IP>` | 22 | ansible | Win32-OpenSSH |
| `win_client_10` | Windows 10 | `<WIN_10_IP>` | 22 | hanane | Win32-OpenSSH |
| `phone` | Android (Termux) | 192.168.1.31 | 8022 | u0_a338 | Termux SSH server |

**Common credentials:** `ansible` / `12345678` (except where overridden above).

---

## 3. Directory Structure

```
SEC-LAb/
├── ansible.cfg                     # Global Ansible configuration
├── inventory.ini                   # Host inventory (all groups)
│
├── group_vars/                     # Per-group variables (best practice)
│   ├── all.yml                     # Shared: credentials, connection type
│   ├── linux_servers.yml           # Linux: sudo, python path
│   ├── windows_machines.yml        # Windows: powershell shell type
│   └── phones.yml                  # Phones: port 8022, Termux paths
│
├── roles/                          # Modular, reusable task bundles
│   ├── common/                     # [J1] Universal SSH connectivity check
│   │   └── tasks/main.yml
│   ├── linux_baseline/             # [J1] Ubuntu system info collection
│   │   └── tasks/main.yml
│   ├── windows_baseline/           # [J1] Windows system info via PowerShell
│   │   └── tasks/main.yml
│   ├── phone_check/                # [J1] Termux/Android device info
│   │   └── tasks/main.yml
│   ├── openscap_scan/              # [J2] OpenSCAP CIS+ANSSI baseline audit
│   │   └── tasks/main.yml
│   ├── hardeningkitty_scan/        # [J2] HardeningKitty CIS baseline audit
│   │   └── tasks/main.yml
│   ├── linux_hardening/            # [J3] CIS/ANSSI hardening for Ubuntu
│   │   ├── tasks/
│   │   │   ├── main.yml            # Orchestrator — includes all sub-tasks
│   │   │   ├── filesystem.yml      # Mount options, USB, core dumps
│   │   │   ├── sysctl.yml          # Kernel parameters (network, ASLR, etc.)
│   │   │   ├── ssh.yml             # SSHD configuration hardening
│   │   │   ├── auth.yml            # Password policy, lockout, PAM
│   │   │   ├── services.yml        # Disable unnecessary services
│   │   │   ├── network.yml         # UFW firewall, IPv6, protocol disabling
│   │   │   ├── logging.yml         # auditd rules, rsyslog
│   │   │   ├── permissions.yml     # Critical file permissions
│   │   │   └── banners.yml         # Legal warning banners
│   │   ├── handlers/main.yml       # Service restart handlers
│   │   └── defaults/main.yml       # Tunable variables
│   ├── windows_hardening/          # [J3] CIS hardening for Windows
│   │   └── tasks/
│   │       ├── main.yml            # Orchestrator
│   │       ├── account_policy.yml  # Password/lockout via secedit
│   │       ├── audit_policy.yml    # Advanced audit policy (auditpol)
│   │       ├── registry.yml        # Registry keys (UAC, SMB, LLMNR, etc.)
│   │       ├── services.yml        # Disable unnecessary Windows services
│   │       ├── firewall.yml        # Windows Firewall hardening
│   │       ├── network.yml         # NetBIOS, WPAD, SMBv1, encryption
│   │       └── user_rights.yml     # Privilege assignments, PS v2 disable
│   ├── openscap_rescan/            # [J3] Post-hardening Linux rescan
│   │   └── tasks/main.yml
│   └── hardeningkitty_rescan/      # [J3] Post-hardening Windows rescan
│       └── tasks/main.yml
│
├── playbooks/                      # Executable playbooks
│   ├── site.yml                    # [J1] Master: all groups connectivity
│   ├── linux_check.yml             # [J1] Linux servers only
│   ├── windows_check.yml           # [J1] Windows machines only
│   ├── phone_check.yml             # [J1] Phones only
│   ├── baseline_scan.yml           # [J2] OpenSCAP + HardeningKitty
│   ├── linux_scan.yml              # [J2] Linux OpenSCAP only
│   ├── windows_scan.yml            # [J2] Windows HardeningKitty only
│   ├── harden.yml                  # [J3] Hardening (both OS)
│   ├── linux_harden.yml            # [J3] Linux hardening only
│   ├── windows_harden.yml          # [J3] Windows hardening only
│   ├── rescan.yml                  # [J3] Post-hardening rescan (both OS)
│   └── full_pipeline.yml           # [ALL] Complete J1→J2→J3 pipeline
│
├── reports/                        # Auto-populated by scan playbooks
│   ├── linux/                      # OpenSCAP HTML + XML reports
│   └── windows/                    # HardeningKitty HTML + CSV reports
│
└── docs/
    └── PROJECT.md                  # This documentation file
```

---

## 4. File-by-File Explanation

### `ansible.cfg`
Global configuration for the Ansible control node:
- **`host_key_checking = False`** — Suppresses SSH fingerprint prompts. Required in a lab where hosts are frequently rebuilt.
- **`pipelining = True`** — Reduces SSH overhead by executing multiple commands in a single connection. Improves performance (~2x for large plays).
- **`ssh_args`** — Uses `ControlMaster` for SSH connection multiplexing, `StrictHostKeyChecking=no` to avoid interactive prompts.
- **`stdout_callback = ansible.builtin.default`** + **`result_format = yaml`** — Human-readable YAML output (replaces deprecated `community.general.yaml` callback).

### `inventory.ini`
Defines three host groups:
- **`[phones]`** — Android devices reachable via Termux SSH (port 8022).
- **`[linux_servers]`** — Ubuntu 22.04 servers (standard SSH on port 22).
- **`[windows_machines]`** — Windows machines with Win32-OpenSSH enabled.

Host-specific overrides (like `ansible_user=hanane` on `win_client_10`) are inline.  
Group-wide variables have been moved to `group_vars/` files.

### `group_vars/all.yml`
Variables applied to **every host** in the inventory:
- **`ansible_user`** / **`ansible_password`** — Shared lab credentials.
- **`ansible_connection: ssh`** — Enforces SSH as the sole transport (no WinRM, no local).

### `group_vars/linux_servers.yml`
Linux-specific settings:
- **`ansible_become: yes`** — Enables privilege escalation for hardening tasks.
- **`ansible_become_method: sudo`** — Uses sudo (default on Ubuntu).
- **`ansible_python_interpreter`** — Explicitly points to `/usr/bin/python3` to avoid ambiguity.

### `group_vars/windows_machines.yml`
Windows-over-SSH settings:
- **`ansible_shell_type: powershell`** — Critical setting. Tells Ansible to wrap all commands in PowerShell syntax. Without this, Ansible sends `/bin/sh` commands which fail on Windows.
- No `become` — The `ansible` user must be a local Administrator. SSH on Windows does not support sudo/runas via Ansible become plugins reliably.

### `group_vars/phones.yml`
Termux-specific settings:
- **`ansible_port: 8022`** — Termux's SSHd listens on 8022 (non-privileged port, no root needed).
- **`ansible_user: u0_a338`** — Termux assigns a Linux UID-based username.
- **`ansible_python_interpreter`** — Points to Termux's prefix path.

### `roles/common/tasks/main.yml`
Universal connectivity role applied to **all** groups:
1. Tries `ansible.builtin.ping` (verifies SSH + Python).
2. Falls back to `raw: echo "SSH OK"` if ping fails (e.g., Windows without Python, Termux with incomplete setup).
3. Prints a clear CONNECTED / UNREACHABLE status.

### `roles/linux_baseline/tasks/main.yml`
Gathers Ubuntu "before" state:
- OS version, kernel, IP via `setup` module.
- SSH daemon status via `systemd` module.
- Open TCP ports via `ss -tlnp`.
- Installed package count via `dpkg`.

### `roles/windows_baseline/tasks/main.yml`
Gathers Windows "before" state using **raw PowerShell** (not win_* modules, which require WinRM):
- PowerShell version, OS name/version/build.
- SSH service status.
- Listening TCP ports (`Get-NetTCPConnection`).
- Firewall profile status.
- Antivirus detection (SecurityCenter2 — client SKUs only).

### `roles/phone_check/tasks/main.yml`
Termux device inspection:
- SSH connectivity check.
- Device info (user, kernel, arch, uptime).
- Disk space on `/data`.
- Termux package count.
- Python availability.

### `roles/openscap_scan/tasks/main.yml` *(Jalon 2)*
OpenSCAP compliance scanning for Ubuntu 22.04:
- **Installs** `libopenscap8` + `ssg-debderived` (SCAP Security Guide with CIS/ANSSI profiles).
- **Auto-detects** the SSG datastream file (handles version differences).
- **Lists** all available XCCDF profiles for visibility.
- **Runs two scans**: CIS Level 1 Server + ANSSI BP28 Enhanced.
- **Generates** HTML + XML reports on the target.
- **Fetches** all reports to `reports/linux/` on the control node.
- WHY CIS: Center for Internet Security benchmarks are the global standard for system hardening.
- WHY ANSSI: Agence nationale de la sécurité des systèmes d'information — France's national cybersecurity authority publishes BP28 guidelines.

### `roles/hardeningkitty_scan/tasks/main.yml` *(Jalon 2)*
HardeningKitty CIS audit for Windows via SSH/PowerShell:
- **Downloads** HardeningKitty from GitHub (idempotent — skips if present).
- **Auto-detects** the OS (Server 2019, Server 2022, Win 10, Win 11) and selects the matching CIS finding list.
- **Runs audit** in non-destructive `Audit` mode — reads registry keys, security policies, firewall rules, services, but changes nothing.
- **Generates** a CSV report + converts it to a styled HTML report with pass/fail/warning color coding and compliance score.
- **Fetches** reports to `reports/windows/` on the control node.
- WHY raw PowerShell: Win32-OpenSSH doesn't support WinRM, so all commands go through `ansible.builtin.raw` with PowerShell.

---

## 5. How SSH Connectivity Works

### Linux (Ubuntu 22.04)
Standard OpenSSH server. Ansible uses `ansible.builtin.ping` which:
1. Opens an SSH session.
2. Copies a tiny Python script to `/tmp/.ansible/`.
3. Executes it and reads the JSON response `{"ping": "pong"}`.

### Windows (Server 2019 / Win 10)
Win32-OpenSSH (Microsoft's port). Key difference:
- **`ansible_shell_type: powershell`** tells Ansible to encode commands as PowerShell.
- We use `ansible.builtin.raw` (not `win_shell`) because `raw` sends commands directly over SSH without needing WinRM or a Python interpreter on Windows.
- Win32-OpenSSH must be installed and the `sshd` service running.

### Phones (Android / Termux)
Termux provides a minimal Linux userland:
- SSH server on port **8022** (unprivileged, no root).
- Limited Python support — `gather_facts: no` avoids failures.
- `raw` module used for maximum compatibility.

---

## 6. Running the Playbooks

### Prerequisites
```bash
# Verify Ansible is installed
ansible --version

# Replace placeholder IPs in inventory.ini
nano inventory.ini   # Change <UBUNTU_IP>, <WIN_2019_IP>, <WIN_10_IP>
```

### Commands

| Command | Scope | Purpose |
|---------|-------|---------|
| `ansible-playbook playbooks/site.yml` | All hosts | Full connectivity + baseline check |
| `ansible-playbook playbooks/linux_check.yml` | Linux only | Ubuntu SSH + system info |
| `ansible-playbook playbooks/windows_check.yml` | Windows only | Windows SSH + system info |
| `ansible-playbook playbooks/phone_check.yml` | Phones only | Termux SSH + device info |

### Jalon 2 — Compliance Scanning Commands

| Command | Scope | Purpose |
|---------|-------|--------|
| `ansible-playbook playbooks/baseline_scan.yml` | Linux + Windows | Full baseline compliance scan (both tools) |
| `ansible-playbook playbooks/linux_scan.yml` | Linux only | OpenSCAP CIS + ANSSI scan → HTML reports |
| `ansible-playbook playbooks/windows_scan.yml` | Windows only | HardeningKitty CIS audit → HTML + CSV reports |

Reports are automatically fetched to:
- `reports/linux/` — OpenSCAP HTML + XML (one per profile per host)
- `reports/windows/` — HardeningKitty HTML + CSV (one per host)

### Quick connectivity test (no playbook needed)
```bash
# Ping all hosts
ansible all -m ping

# Ping specific group
ansible linux_servers -m ping
ansible windows_machines -m raw -a "echo OK"
ansible phones -m raw -a "echo OK"

# List inventory to verify parsing
ansible-inventory --list
```

### Verbose mode for debugging
```bash
ansible-playbook playbooks/site.yml -vvv
```

---

## 7. Jalon 2 — Baseline Compliance Scanning

### What happens in this phase

```
┌────────────────────┐     ┌──────────────────────────────────┐
│  Control Node      │────▶│  Ubuntu Server 22.04             │
│  (ansible-playbook)│     │  ├── apt install openscap + ssg  │
│                    │     │  ├── oscap eval --profile cis     │
│                    │     │  ├── oscap eval --profile anssi   │
│                    │◀────│  └── fetch HTML+XML reports       │
│                    │     └──────────────────────────────────┘
│                    │
│                    │     ┌──────────────────────────────────┐
│                    │────▶│  Windows Server 2019 / Win 10    │
│                    │     │  ├── Download HardeningKitty     │
│                    │     │  ├── Invoke-HardeningKitty Audit │
│                    │◀────│  └── fetch HTML+CSV reports       │
└────────────────────┘     └──────────────────────────────────┘
```

### OpenSCAP Profiles Used

| Profile ID | Description | Standard |
|-----------|-------------|----------|
| `xccdf_org.ssgproject.content_profile_cis_level1_server` | CIS Benchmark Level 1 for servers | CIS |
| `xccdf_org.ssgproject.content_profile_anssi_bp28_enhanced` | ANSSI BP28 Enhanced hardening | ANSSI |

### HardeningKitty Auto-Detection

The role automatically selects the correct CIS finding list based on the OS:

| OS Caption Match | Finding List Pattern |
|-----------------|---------------------|
| `Server 2019` | `finding_list_cis_microsoft_windows_server_2019*` |
| `Server 2022` | `finding_list_cis_microsoft_windows_server_2022*` |
| `Windows 10` | `finding_list_cis_microsoft_windows_10*` |
| `Windows 11` | `finding_list_cis_microsoft_windows_11*` |

### Understanding the Reports

**OpenSCAP HTML Report** contains:
- Overall compliance score (percentage)
- Per-rule pass/fail/notapplicable/error breakdown
- Rule descriptions with ANSSI/CIS references
- Remediation suggestions for each failed rule

**HardeningKitty HTML Report** contains:
- Compliance score (percentage)
- Color-coded table: green=passed, red=failed, yellow=warning
- Expected vs. current value for each check
- Severity rating per rule

### Expected Baseline Scores

On **unhardened** systems, expect:
- **Linux (CIS):** ~30-50% (Ubuntu defaults are reasonable but not CIS-compliant)
- **Linux (ANSSI):** ~25-45% (ANSSI BP28 Enhanced is stricter)
- **Windows (CIS):** ~20-40% (default Windows installs are permissive)

These scores are the "before" picture. Jalon 3 will bring them above 85%.

---

## 8. Jalon 3 — CIS/ANSSI Hardening

### Linux Hardening Role (`linux_hardening`)

The role is split into **9 modular task files**, each addressing a CIS/ANSSI control family:

| Task File | CIS Controls | ANSSI Controls | What It Does |
|-----------|-------------|----------------|--------------|
| `filesystem.yml` | 1.1.x | R14, R28, R67 | Mount options (noexec/nosuid/nodev), disable USB, disable uncommon FS, disable core dumps |
| `sysctl.yml` | 3.3.x | R12, R13 | Kernel network hardening (anti-spoofing, SYN cookies, ICMP), ASLR, restrict dmesg/ptrace |
| `ssh.yml` | 5.2.x | R33-R37 | Disable root login, strong ciphers only, session timeout, X11/TCP/agent forwarding off |
| `auth.yml` | 5.3-5.4 | R30-R31, R51 | Password complexity (pwquality), SHA-512 hashing, account lockout, shell TMOUT, restrict su |
| `services.yml` | 2.1-2.2 | R16-R18 | Disable avahi/cups/rpcbind/snapd, remove telnet/rsh/talk, local-only MTA |
| `network.yml` | 3.4-3.5 | R20-R22 | UFW default deny + SSH allow, disable IPv6/DCCP/SCTP/RDS/TIPC |
| `logging.yml` | 4.1-4.2 | R40-R42 | Install auditd, 40+ audit rules (time, identity, login, permissions, modules), rsyslog |
| `permissions.yml` | 6.1-6.2 | R54-R57 | Lock down /etc/passwd, /etc/shadow, crontab, cron dirs, /boot |
| `banners.yml` | 1.7.x | R20 | Legal warning banners on /etc/issue, /etc/issue.net, /etc/motd |

All variables are tuneable via `roles/linux_hardening/defaults/main.yml`.

### Windows Hardening Role (`windows_hardening`)

The role uses 7 modular task files, all executing raw PowerShell over SSH:

| Task File | CIS Controls | What It Does |
|-----------|-------------|--------------|
| `account_policy.yml` | 1.1-1.2 | Password history (24), min length (14), complexity, lockout (5 tries/30 min), disable Guest, rename Admin |
| `audit_policy.yml` | 17.x, 18.9.27 | 30+ advanced audit subcategories enabled, command-line logging, event log sizes (Security=192MB) |
| `registry.yml` | 2.3.x, 18.x | 30+ registry keys: UAC, SMB signing, NTLMv2 only, LLMNR off, LSA protection, Credential Guard, SEHOP, autoplay off, WSH off |
| `services.yml` | 5.x | Disable Print Spooler (PrintNightmare), RDP, Remote Registry, SNMP, Xbox, Bluetooth, etc. |
| `firewall.yml` | 9.x | All profiles enabled, default block inbound, 16MB logging, SSH whitelisted, RDP/WinRM disabled |
| `network.yml` | 18.5-18.6 | Disable NetBIOS, WPAD, SMBv1 (EternalBlue), enable SMB encryption |
| `user_rights.yml` | 2.2.x | Restrict network/local logon, deny Guest, disable PowerShell v2, enable ScriptBlock logging |

### Running Jalon 3

```bash
# Harden both Linux + Windows
ansible-playbook playbooks/harden.yml

# Or individually
ansible-playbook playbooks/linux_harden.yml
ansible-playbook playbooks/windows_harden.yml

# Post-hardening compliance rescan
ansible-playbook playbooks/rescan.yml

# Or run the complete pipeline (J1 → J2 → J3 + rescan)
ansible-playbook playbooks/full_pipeline.yml
```

### Report Comparison

After running `rescan.yml`, compare files in `reports/`:

| File | Type | When |
|------|------|------|
| `*_cis_baseline.html` | OpenSCAP CIS | Before hardening |
| `*_anssi_baseline.html` | OpenSCAP ANSSI | Before hardening |
| `*_cis_post_hardening.html` | OpenSCAP CIS | After hardening |
| `*_anssi_post_hardening.html` | OpenSCAP ANSSI | After hardening |
| `*_cis_baseline.html` | HardeningKitty | Before hardening |
| `*_cis_post_hardening.html` | HardeningKitty | After hardening |

---

## 9. Jalon Roadmap

| Jalon | Status | Description |
|-------|--------|-------------|
| **Jalon 1** | Complete | Validate SSH connectivity, build inventory & roles structure |
| **Jalon 2** | Complete | Blank compliance scan: OpenSCAP + HardeningKitty → HTML baseline reports |
| **Jalon 3** | Complete | CIS/ANSSI hardening roles → target >85% compliance |

---

## 10. Troubleshooting

### Common Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| `Permission denied (publickey,password)` | SSH password auth disabled | Enable `PasswordAuthentication yes` in `/etc/ssh/sshd_config` (Linux) or OpenSSH config (Windows) |
| `Connection timed out` | Wrong IP or firewall | Verify IP, check `ufw` / Windows Firewall allows port 22/8022 |
| `MODULE FAILURE` on Windows | Missing `shell_type` | Ensure `ansible_shell_type: powershell` in `group_vars/windows_machines.yml` |
| `No python interpreter found` on Phone | Termux has no Python | Install: `pkg install python` in Termux |
| `/bin/sh: powershell: not found` | Ansible treating Windows as Linux | Check `ansible_shell_type` is set correctly |
| `unreachable` on Phones | Wrong port | Confirm `ansible_port: 8022` in `group_vars/phones.yml` |
| `community.general.yaml callback removed` | Ansible collection v12 | Use `stdout_callback = ansible.builtin.default` + `result_format = yaml` in ansible.cfg |
| OpenSCAP `exit code 2` | Some rules failed | **Expected** on unhardened systems — exit 2 = scan ran, some checks failed |
| OpenSCAP `exit code 1` | Scanner error | Check SSG content: `ls /usr/share/xml/scap/ssg/content/` |
| HardeningKitty download fails | No internet / TLS error | Manually copy HardeningKitty ZIP to `C:\SEC-OPS\` on Windows target |
| HardeningKitty `No CIS finding list` | Missing list for OS | Check `C:\SEC-OPS\HardeningKitty\lists\` — may need newer HardeningKitty version |
| **UFW locks you out** | SSH rule not added before default deny | Role adds SSH allow first; if locked out, access console and run `ufw allow 22/tcp` |
| **auditd rules don't apply** | Immutable flag (`-e 2`) active | Requires a **full reboot** to load new rules after changes |
| **secedit fails on non-domain PC** | Template references domain-only settings | Expected on standalone — some policies only apply to domain-joined machines |
| **Windows service won't stop** | Running as dependency | Stop dependent services first, or set StartupType to Disabled and reboot |
| **Password policy not enforced** | Local policy overridden by GPO | On domain-joined machines, domain GPO takes precedence over local secedit |

### Enabling Win32-OpenSSH (Windows)

**Windows Server 2019:**
```powershell
# Install OpenSSH Server (if not already)
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# Start and enable the service
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic

# Ensure firewall rule exists
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

**Windows 10:**
```powershell
# Via Settings → Apps → Optional Features → Add OpenSSH Server
# Or via PowerShell:
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

### Enabling Termux SSH (Android)
```bash
# Inside Termux
pkg install openssh
sshd   # Starts on port 8022
passwd # Set password
```

---

> **Project status:** Jalons 1–3 are complete. Run the full pipeline with `ansible-playbook playbooks/full_pipeline.yml` to execute connectivity checks → baseline scans → hardening → compliance rescan in one shot. Compare before/after HTML reports in `reports/` to verify the >85% CIS/ANSSI compliance target.
