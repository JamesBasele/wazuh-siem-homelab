# 🛡️ SOC Home Lab — Wazuh SIEM on VirtualBox

![Wazuh](https://img.shields.io/badge/Wazuh-4.7.5-blue)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-orange)
![Windows](https://img.shields.io/badge/Windows-10_Pro-0078D6)
![VirtualBox](https://img.shields.io/badge/VirtualBox-7.x-183A61)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

A fully functional, hands-on Security Operations Centre (SOC) home lab built using **Wazuh SIEM** deployed on **Ubuntu Server 22.04** inside **VirtualBox**, with a **Windows 10** endpoint as the monitored agent. This lab was built from scratch as a portfolio project to demonstrate practical skills in SIEM deployment, log ingestion, detection engineering, threat simulation, and incident analysis.

---

## 📋 Table of Contents

- [Lab Overview](#lab-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Wazuh Installation](#wazuh-installation)
- [Agent Deployment](#agent-deployment)
- [Custom Detection Rules](#custom-detection-rules)
- [Attack Simulations](#attack-simulations)
- [Dashboard & Visualizations](#dashboard--visualizations)
- [Challenges & Troubleshooting](#challenges--troubleshooting)
- [Key Takeaways](#key-takeaways)
- [Screenshots](#screenshots)
- [Skills Demonstrated](#skills-demonstrated)

---

## Lab Overview

| Component | Details |
|---|---|
| **SIEM Platform** | Wazuh 4.7.5 (All-in-One) |
| **Server OS** | Ubuntu Server 22.04 LTS |
| **Endpoint OS** | Windows 10 Pro (64-bit) |
| **Hypervisor** | Oracle VirtualBox 7.x |
| **Network** | VirtualBox Host-Only (192.168.56.0/24) |
| **Host RAM Allocated** | 6GB (Wazuh Server VM) |
| **Host vCPUs Allocated** | 2 vCPUs (Wazuh Server VM) |
| **Disk** | 50GB dynamically allocated VDI |

### What This Lab Covers

- Deploying a production-grade open-source SIEM from scratch
- Ingesting logs from a Windows endpoint via Wazuh agent
- Writing custom XML-based detection rules
- Simulating real-world attack scenarios (brute force, privilege escalation, port scanning, log tampering)
- Building SOC dashboards and visualizations
- Incident response and root cause analysis

---

## Architecture

```
Host PC (Windows)
│
├── VirtualBox Host-Only Network: 192.168.56.0/24
│
├── VM 1: Wazuh Server (Ubuntu 22.04)
│   IP: 192.168.56.101
│   ├── Wazuh Manager      ← Detection engine & rule processing
│   ├── Wazuh Indexer      ← OpenSearch data storage (JVM-based)
│   └── Wazuh Dashboard    ← Web UI (https://192.168.56.101)
│
└── VM 2: Windows 10 Endpoint
    IP: 192.168.56.102
    └── Wazuh Agent        ← Forwards logs/events to Wazuh Server
```

**Data Flow:**
```
Windows Event Logs → Wazuh Agent → Wazuh Manager → Rule Engine → Wazuh Indexer → Dashboard
```

---

## Prerequisites

### Software Downloads

| Tool | Version | Purpose |
|---|---|---|
| [VirtualBox](https://www.virtualbox.org/wiki/Downloads) | 7.x | Hypervisor |
| [Ubuntu Server ISO](https://ubuntu.com/download/server) | 22.04 LTS | Wazuh Server OS |
| [Windows 10 ISO](https://www.microsoft.com/software-download/windows10) | Latest | Monitored Endpoint |

### Minimum Host PC Specs

| Resource | Minimum | Recommended |
|---|---|---|
| RAM | 8GB | 16GB |
| CPU Cores | 4 | 8 |
| Free Disk | 80GB | 120GB |
| OS | Windows 10/11, macOS, Linux | — |

---

## Environment Setup

### VirtualBox Host-Only Network

Before creating VMs, a dedicated Host-Only network was configured to allow communication between the host PC and both VMs without exposing them to external networks.

**Steps:**
1. VirtualBox → File → Tools → Network Manager
2. Host-only Networks → Create
3. Network range: `192.168.56.0/24`

### VM Specifications

**Wazuh Server VM:**
- RAM: 6144 MB (6GB)
- vCPUs: 2
- Storage: 50GB VDI (Dynamically allocated)
- Network Adapter 1: NAT (internet access during install)
- Network Adapter 2: Host-only (192.168.56.x)

**Windows Endpoint VM:**
- RAM: 2048 MB (2GB)
- vCPUs: 1
- Storage: 40GB VDI (Dynamically allocated)
- Network Adapter 1: NAT
- Network Adapter 2: Host-only (192.168.56.x)

---

## Wazuh Installation

Wazuh was deployed using the official all-in-one installation script, which installs the Manager, Indexer, and Dashboard on a single node.

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Download and run the Wazuh install script
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a -i
```

> **Note:** The `-i` flag bypasses the OS compatibility check. Wazuh 4.7.5's install script throws a warning on Ubuntu 22.04 due to its supported OS list, but the installation works correctly with this flag.

### Post-Install Service Management

```bash
# Start services in correct order (indexer must come first)
sudo systemctl start wazuh-indexer
sleep 30
sudo systemctl start wazuh-manager
sleep 20
sudo systemctl start wazuh-dashboard

# Enable on boot
sudo systemctl enable wazuh-indexer wazuh-manager wazuh-dashboard
```

### Dashboard Access

After installation, the dashboard is accessible at:
```
https://192.168.56.101
Username: admin
Password: <auto-generated during install — save this immediately>
```

---

## Agent Deployment

The Wazuh agent was installed on the Windows 10 VM to forward security events to the Wazuh Manager.

### Windows Agent Installation

1. Download: `https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi`
2. During installation, set **Manager IP** to: `192.168.56.101`
3. Start the agent service:

```cmd
NET START WazuhSvc
```

### Verify Agent Connection

In the Wazuh Dashboard → **Agents** → Confirm the Windows endpoint shows status: **Active**

---

## Custom Detection Rules

Custom rules were written in XML and saved to `/var/ossec/etc/rules/local_rules.xml`.

```xml
<!-- Local rules -->
<!-- Copyright (C) 2015, Wazuh Inc. -->

<group name="local,">

  <!-- Rule: Detect new local user creation on Windows -->
  <rule id="100001" level="12">
    <if_group>windows</if_group>
    <match>net user</match>
    <description>Possible new user account created via command line</description>
    <group>account_creation,attack</group>
  </rule>

  <!-- Rule: Detect multiple failed logins (brute force) -->
  <rule id="100002" level="10" frequency="5" timeframe="120">
    <if_matched_sid>60122</if_matched_sid>
    <description>Brute force attack: 5+ failed logins in 2 minutes</description>
    <group>authentication_failures,attack</group>
  </rule>

  <!-- Rule: Detect suspicious file in Temp folder -->
  <rule id="100003" level="8">
    <if_group>syscheck</if_group>
    <match>C:\\Users\\.*\\AppData\\Local\\Temp</match>
    <description>File created in Temp directory - possible malware dropper</description>
    <group>suspicious_file,attack</group>
  </rule>

</group>
```

After saving, the manager was restarted to apply the rules:

```bash
sudo systemctl restart wazuh-manager
```

### Filtering Alerts by Custom Rules in Dashboard

```
# By rule ID
rule.id: 100001 OR rule.id: 100002 OR rule.id: 100003

# By shared group tag
rule.groups: attack

# By severity level
rule.level >= 8
```

---

## Attack Simulations

### Visualization 1 — Authentication Failures Over Time

**Attack 1: RDP/SMB Brute Force (CMD)**
```cmd
for /L %i in (1,1,20) do net use \\localhost\IPC$ /user:Administrator wrongpassword
```

**Attack 2: PowerShell Authentication Failures**
```powershell
1..15 | ForEach-Object {
    net use \\localhost\IPC$ /user:Administrator wrongpass
}
```

**Attack 3: SSH Brute Force (Ubuntu VM)**
```bash
# Install Hydra
sudo apt install hydra -y

# Create password list
cat > /tmp/passwords.txt << EOF
password
123456
admin
letmein
qwerty
wrongpass
EOF

# Run brute force
hydra -l wazuhuser -P /tmp/passwords.txt ssh://127.0.0.1 -t 4 -V
```

---

### Visualization 2 — Top Triggered Rules (Pie Chart)

**Attack 1: Malware File Drop Simulation**
```powershell
$locations = @(
    "C:\Users\labuser\AppData\Local\Temp\payload.exe",
    "C:\Users\labuser\AppData\Local\Temp\update.bat",
    "C:\Users\labuser\AppData\Local\Temp\svchost32.exe"
)
foreach ($path in $locations) {
    New-Item -Path $path -ItemType File -Force
    Add-Content -Path $path -Value "Simulated malware file"
    Start-Sleep -Seconds 2
}
```

**Attack 2: Registry Persistence Simulation**
```powershell
$regPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"
Set-ItemProperty -Path $regPath -Name "WindowsUpdate32" -Value "C:\Temp\malware.exe"
Set-ItemProperty -Path $regPath -Name "SvcHost64" -Value "C:\Windows\Temp\backdoor.exe"
Start-Sleep -Seconds 5

# Clean up
Remove-ItemProperty -Path $regPath -Name "WindowsUpdate32"
Remove-ItemProperty -Path $regPath -Name "SvcHost64"
```

**Attack 3: Log Tampering (Attacker Covering Tracks)**
```powershell
# Wazuh detects log clearing attempts
wevtutil cl System
wevtutil cl Application
wevtutil cl Security
```

**Attack 4: Port Scan (Ubuntu → Windows)**
```bash
sudo nmap -sS 192.168.56.102 -p 1-1000
sudo nmap -sV 192.168.56.102
sudo nmap -A 192.168.56.102
```

---

### Visualization 3 — Agents With Most Alerts (Bar Chart)

**Attack 1: Bulk User Creation & Deletion**
```cmd
:: Create 10 fake backdoor accounts
for /L %i in (1,1,10) do net user hacker%i Password123! /add
for /L %i in (1,1,5) do net localgroup administrators hacker%i /add
timeout /t 30
for /L %i in (1,1,10) do net user hacker%i /delete
```

**Attack 2: Scheduled Task Persistence**
```powershell
1..5 | ForEach-Object {
    schtasks /create /tn "WindowsUpdate$_" /tr "C:\Temp\malware$_.exe" /sc minute /mo 5 /f
    Start-Sleep -Seconds 3
}
# Clean up
1..5 | ForEach-Object { schtasks /delete /tn "WindowsUpdate$_" /f }
```

**Attack 3: File Integrity Monitoring — Mass File Modification (Ransomware Simulation)**
```powershell
New-Item -Path "C:\TestFiles" -ItemType Directory -Force
1..20 | ForEach-Object { Set-Content "C:\TestFiles\document$_.txt" -Value "Original $_" }
Start-Sleep -Seconds 10
# Simulate encryption
1..20 | ForEach-Object {
    Set-Content "C:\TestFiles\document$_.txt" -Value "ENCRYPTED_$([System.Guid]::NewGuid())"
    Start-Sleep -Milliseconds 500
}
```

---

## Dashboard & Visualizations

Three visualizations were built in the Wazuh Dashboard under a custom workbook named **"SOC Home Lab Overview"**:

| Panel | Index | Field/Filter | Chart Type |
|---|---|---|---|
| Auth Failures Over Time | `wazuh-alerts-*` | `rule.groups: authentication_failures` | Time chart |
| Top Triggered Rules | `wazuh-alerts-*` | `rule.description` | Pie chart |
| Agents With Most Alerts | `wazuh-alerts-*` | `agent.name` | Bar chart |

> All three visualizations use the **Wazuh Alerts** source/index — not Wazuh Monitoring or Wazuh Statistics, which track agent connectivity and manager performance metrics respectively.

---

## Challenges & Troubleshooting

This section documents every significant challenge encountered during the build, along with the root cause and resolution. These are real issues from a real build — not a clean-run walkthrough.

---

### Challenge 1: OS Compatibility Check Failure During Wazuh Install

**Error:**
```
ERROR: The recommended systems are: Red Hat Enterprise Linux 7, 8, 9;
CentOS 7, 8; Amazon Linux 2; Ubuntu 16...
The current system does not match this list.
```

**Root Cause:** Wazuh 4.7.5's install script performs an OS version check that doesn't include Ubuntu 22.04 in its validated list, even though the installation works correctly on it.

**Resolution:** Added the `-i` flag to bypass the check:
```bash
sudo bash wazuh-install.sh -a -i
```

---

### Challenge 2: Forgotten VM Terminal Password

**Situation:** Lost the Ubuntu VM login password mid-lab.

**Resolution:** Booted into Ubuntu Recovery Mode via GRUB, dropped to root shell, and reset the password:
```bash
mount -o remount,rw /
passwd wazuhuser
reboot
```

---

### Challenge 3: VirtualBox Guest Additions Not Installed (Drag & Drop / Clipboard)

**Error:** `DnD: Error: Drag and drop to guest not possible — VBOX_E_DND_ERROR`

**Root Cause:** VirtualBox Guest Additions were not installed on either VM, preventing shared clipboard and drag-and-drop functionality between host and guest.

**Resolution (Ubuntu VM):**
```bash
sudo apt install build-essential dkms linux-headers-$(uname -r) -y
sudo mount /dev/sr0 /mnt/cdrom
sudo bash /mnt/cdrom/VBoxLinuxAdditions.run
sudo reboot
```

**Resolution (Windows VM):** Inserted Guest Additions CD via VirtualBox menu → ran `VBoxWindowsAdditions-amd64.exe` inside the Windows VM → rebooted.

**Note:** The mount warning `WARNING: source write-protected, mounted read-only` is normal behaviour for a CD-ROM device and is not an error.

---

### Challenge 4: Wazuh Dashboard SSL Certificate Missing

**Error:**
```
errno: -2, syscall: 'open', code: 'ENOENT',
path: '/etc/wazuh-dashboard/certs/dashboard-key.pem'
```

**Root Cause:** The dashboard's SSL certificate files existed but with different names (`wazuh-dashboard-key.pem`) than what the config expected (`dashboard-key.pem`). This was caused by the install script being run twice (first attempt failed due to Challenge 1 above), resulting in mismatched cert references.

**Resolution:** Copied the existing certs to the expected filenames and fixed permissions:
```bash
sudo cp /etc/wazuh-dashboard/certs/wazuh-dashboard-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem
sudo cp /etc/wazuh-dashboard/certs/wazuh-dashboard.pem /etc/wazuh-dashboard/certs/dashboard.pem
sudo chown wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs/dashboard-key.pem /etc/wazuh-dashboard/certs/dashboard.pem
sudo chmod 400 /etc/wazuh-dashboard/certs/dashboard-key.pem /etc/wazuh-dashboard/certs/dashboard.pem
```

---

### Challenge 5: Wazuh Indexer Startup Timeout

**Error:**
```
wazuh-indexer.service: start operation timed out. Terminating.
wazuh-indexer.service: Failed with result 'timeout'.
Mem peak: 895.7M (swap: 741.3M)
```

**Root Cause:** The Wazuh Indexer (OpenSearch/JVM) was slow to initialize on a resource-constrained VM. systemd's default startup timeout was too short, causing it to kill the process before it finished starting. Heavy swap usage indicated RAM pressure.

**Resolution 1 — Increased systemd startup timeout:**
```bash
sudo systemctl edit wazuh-indexer
# Added:
[Service]
TimeoutStartSec=300
sudo systemctl daemon-reload
```

**Resolution 2 — Reduced JVM heap size:**
```bash
sudo nano /etc/wazuh-indexer/jvm.options
# Changed -Xms1g / -Xmx1g to:
-Xms512m
-Xmx512m
```

**Resolution 3 — Increased VM RAM** from 4GB to 6GB via VirtualBox settings.

---

### Challenge 6: Custom Rules File XML Errors

**Situation:** After adding custom detection rules, `sudo systemctl restart wazuh-manager` failed.

**Root Cause:** Multiple XML syntax errors in `local_rules.xml`:
- Duplicate rule ID (`100001` used twice)
- Missing `=` in attribute: `timeframe"120"` instead of `timeframe="120"`
- Spaces inside XML tags: `<descript ion>`, `</ if_group>`, `account_creat ion`
- Malformed XML comments: `<!-- comment -- >` (space before `>`)
- Unclosed example comment block

**Resolution:** Rewrote the entire rules file from scratch with correct XML syntax. Key lesson: XML is unforgiving — a single space inside a tag name breaks the entire file.

---

### Challenge 7: Wazuh Manager Startup Timeout (Same Pattern as Indexer)

**Error:**
```
wazuh-manager.service: start operation timed out.
Process: ExecStart (code=killed, signal=TE...)
```

**Root Cause:** Same timeout pattern as the indexer — the manager was taking longer than systemd's default timeout to initialize all its daemons (analysisd, modulesd, remoted, etc.), so systemd killed it as "failed" even though the processes were actually starting correctly.

**Resolution:** Applied the same override approach:
```bash
sudo systemctl edit wazuh-manager
# Added:
[Service]
TimeoutStartSec=300
sudo systemctl daemon-reload
```

---

### Challenge 8: CPU Soft Lockup — Java GC Thread Freezing the VM

**Error (on VM console):**
```
watchdog: BUG: soft lockup - CPU#0 stuck for 420s! [GC Thread#0:2042]
systemd[1]: systemd-udevd.service: Watchdog timeout (limit 3min)!
```

**Root Cause:** The Wazuh Indexer's JVM Garbage Collector (GC) was monopolizing the single vCPU allocated to the VM, causing a complete CPU freeze for 7 minutes. With only 1 vCPU, when the JVM GC runs a stop-the-world collection, no other processes — including the OS watchdog — get CPU time.

**Resolution:** Increased VM vCPUs from 1 to 2 (minimum) in VirtualBox → System → Processor settings. Also reduced JVM heap (see Challenge 5) to reduce GC frequency and duration.

**Lesson:** Never run a JVM-based application (OpenSearch, Elasticsearch, etc.) on a single-vCPU VM. This is a fundamental infrastructure requirement, not a preference.

---

### Challenge 9: Invalid Dashboard Config Key Causing Crash Loop

**Error:**
```
FATAL Error: Unknown configuration key(s): "wazuh.api.timeout".
Check for spelling errors and ensure that expected plugins are installed.
```

**Root Cause:** Attempted to add `wazuh.api.timeout: 60000` to `opensearch_dashboards.yml` to extend the API connection timeout. This key does not exist in Wazuh 4.7.5's dashboard — it's not a valid configuration option for this version, causing the dashboard to refuse to start entirely (exit code 64/USAGE).

**Resolution:** Removed the invalid key from the config file and restarted the dashboard. The correct approach for timeout issues in this version is to wait for the manager to fully initialize before checking the API connection, not to modify dashboard config.

**Lesson:** Always verify configuration keys against the correct version's documentation. A single invalid key in a YAML config can take down an entire service.

---

### Challenge 10: Virtual Disk Too Small (25GB Instead of 50GB)

**Situation:** Wazuh Indexer and Manager began generating storage warnings. The originally provisioned 25GB VDI was insufficient for the indexer's data storage plus OS and logs.

**Resolution:**

**Step 1 — Resize the VDI from host:**
```cmd
VBoxManage modifymedium "C:\...\Wazuh-Server.vdi" --resize 51200
```

**Step 2 — Extend the partition inside the VM:**
```bash
sudo apt install cloud-guest-utils -y
sudo growpart /dev/sda 3
sudo resize2fs /dev/sda3
# Or for LVM:
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

**Verification:**
```bash
df -h  # Confirm root partition shows new size
```

---

### Challenge 11: PowerShell vs CMD Syntax Confusion

**Error:**
```
for : The term 'for' is not recognized as the name of a cmdlet...
```

**Root Cause:** CMD-style loop syntax (`for /L %i in (1,1,15) do ...`) was pasted into a PowerShell terminal. PowerShell uses entirely different loop syntax.

**Resolution:** Used PowerShell-native loop syntax:
```powershell
# Instead of CMD: for /L %i in (1,1,15) do net use ...
1..15 | ForEach-Object {
    net use \\localhost\IPC$ /user:Administrator wrongpass
}
```

---

## Key Takeaways

### Technical Lessons

**Infrastructure sizing matters from day one.** The majority of issues in this lab (CPU soft lockup, startup timeouts, swap usage) traced back to the VM being under-resourced. For any JVM-based workload (Wazuh Indexer, Elasticsearch), the absolute minimums are 2 vCPUs and 4GB RAM — and that's the floor, not the recommended spec.

**Start services in dependency order.** The Indexer must be fully up before starting the Manager, and both must be up before the Dashboard. Ignoring startup order leads to cascading failures that look like separate bugs but have a single root cause.

**YAML and XML are unforgiving.** A single space inside an XML tag name or an invalid key in a YAML config file can take down an entire service. Always validate config changes before restarting services.

**Separate configuration errors from resource errors.** Several issues in this lab initially looked like configuration problems (dashboard not starting, manager failing) but were actually resource exhaustion (RAM, CPU, timeout). Checking `free -h`, `df -h`, and `dmesg` before diving into config files saves significant time.

**Version-specific documentation is critical.** The `wazuh.api.timeout` incident demonstrated that applying configuration advice without verifying it against your exact version can create new, harder-to-diagnose failures. Always check the Wazuh documentation for your specific version number.

### SOC/Security Lessons

**Detection rules are only as good as their syntax.** Rules with XML errors don't just fail — they can take the entire rule engine down. Test rules in a staging environment before applying to production.

**Alert fatigue is real even in a lab.** Running all six attacks simultaneously generates so much noise that the signal gets lost. In a real SOC, tuning alert thresholds and grouping related events into single incidents is a core part of the analyst's daily work.

**The attack chain matters more than individual alerts.** The most valuable insight from this lab wasn't any single alert — it was seeing the full attack pattern across visualizations: recon (port scan) → initial access (brute force) → persistence (registry/scheduled tasks) → covering tracks (log clearing). Real threat hunting connects these dots.

---

## Screenshots

> Screenshots to be added — take these from your working lab:

```
screenshots/
├── 01-wazuh-dashboard-login.png
├── 02-agents-active.png
├── 03-security-events-overview.png
├── 04-brute-force-alert.png
├── 05-user-creation-alert.png
├── 06-custom-rules-file.png
├── 07-auth-failures-timechart.png
├── 08-top-rules-piechart.png
├── 09-agents-barchart.png
└── 10-soc-dashboard-overview.png
```

---

## Skills Demonstrated

| Skill | Evidence |
|---|---|
| SIEM Deployment | Deployed Wazuh all-in-one on Ubuntu Server in a virtualised environment |
| Log Ingestion | Configured Wazuh agent on Windows 10 to forward security events to centralised SIEM |
| Detection Engineering | Authored custom XML detection rules for brute force, privilege escalation, and suspicious file activity |
| Threat Simulation | Simulated brute force, port scanning, persistence, log tampering, and mass file modification |
| Incident Analysis | Investigated and documented each attack scenario with timeline and findings |
| Linux Administration | Managed systemd services, edited config files, extended disk partitions, managed certificates |
| Troubleshooting | Diagnosed and resolved 11 distinct infrastructure and configuration issues |
| Network Configuration | Configured VirtualBox Host-Only networking for isolated lab communication |
| Dashboard Building | Built custom SOC visualizations for real-time alert monitoring |
| Documentation | Produced structured incident reports and technical write-ups |

---

## Repository Structure

```
wazuh-siem-homelab/
│
├── README.md                          ← This file
├── screenshots/                       ← Dashboard and alert screenshots
├── rules/
│   └── local_rules.xml                ← Custom detection rules
├── configs/
│   ├── opensearch_dashboards.yml      ← Dashboard config (sanitised)
│   └── jvm.options                    ← Indexer JVM tuning
└── incident-reports/
    ├── brute-force-incident.md        ← Sample incident report
    ├── privilege-escalation.md
    └── log-tampering.md
```

---

## References

- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Wazuh Installation Guide](https://documentation.wazuh.com/current/installation-guide/index.html)
- [Wazuh Rules Syntax](https://documentation.wazuh.com/current/user-manual/ruleset/rules/rules-syntax.html)
- [VirtualBox Guest Additions](https://www.virtualbox.org/manual/ch04.html)
- [TryHackMe SOC Level 1 Path](https://tryhackme.com/path/outline/soclevel1)

---

*Built as part of a cybersecurity portfolio project. All attack simulations were performed in an isolated lab environment with no connection to production systems or the public internet.*
