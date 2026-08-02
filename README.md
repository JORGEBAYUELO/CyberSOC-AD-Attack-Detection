# CyberSOC-AD-Attack-Detection

![HeroBanner](/images/AD-Attack-Detection.jpg)

> This project demonstrates how Active Directory reconnaissance and authentication activity can be detected using Sysmon and Wazuh in a Windows domain environment.

## Project Overview

The objective of this investigation was to simulate common attacker reconnaissance techniques from a Kali Linux system against an Active Directory environment while collecting endpoint telemetry with Sysmon and analyzing detections in Wazuh.

This project focuses on:

- Active Directory enumeration
- Network discovery
- SMB authentication
- Windows Security Events
- Wazuh alert analysis
- Initial SOC investigation workflow

---

# Lab Architecture

| Component | Purpose |
|-----------|----------|
| Cyber-DC01 | Windows Server 2022 Domain Controller |
| WIN11-CLIENT01 | Windows 11 Enterprise Endpoint |
| Kali-Attacker01 | Attacker workstation |
| Wazuh Server | SIEM and Detection Platform |

---

# Network

| Host | IP Address |
|--------|------------|
| Cyber-DC01 | 192.168.100.10 |
| WIN11-CLIENT01 | 192.168.100.20 |
| Wazuh-Server01 | 192.168.100.30 |
| Kali-Attacker01 | 192.168.100.40 |

---

# Detection Stack

- Windows Security Logging
- Sysmon
- Wazuh Agent
- Wazuh Server
- MITRE ATT&CK Mapping

---

# Investigation Timeline

---

## Stage 1 - Baseline Collection

Before performing any Active Directory activity, baseline information was collected from both the attacker and the monitored Windows workstation.

### Kali Baseline

Commands executed:

```bash
hostname
whoami
ip addr
ip route
arp -a
```

Baseline output was saved locally on the Kali VM under:

```
~/cybersoc-investigation-001/
```

No files were copied into this repository.

---

### Windows Baseline

Commands executed:

```powershell
hostname

whoami

ipconfig /all

arp -a

netstat -ano
```

The objective was to establish a known-good baseline before beginning reconnaissance.

---

📷 Screenshot Placeholder

```
Insert Windows baseline screenshots here
```

---

## Stage 2 - Host Discovery

The attacker identified active systems within the lab network.

Command executed:

```bash
sudo nmap -sn 192.168.100.0/24
```

Results identified:

- Cyber-DC01
- WIN11-CLIENT01
- Wazuh-Server01
- Kali-Attacker01

Output was saved locally on the Kali VM.

---

📷 Screenshot Placeholder

```
Insert Nmap Host Discovery screenshot
```

---

## Stage 3 - Service Discovery

The monitored workstation was scanned to identify exposed services.

Commands executed:

```bash
sudo nmap -sV 192.168.100.20

sudo nmap -sS -Pn 192.168.100.20
```

Observed Services

- TCP 135
- TCP 139
- TCP 445

These services indicated SMB and RPC availability.

Output remains stored inside:

```
~/cybersoc-investigation-001/
```

---

Screenshot Placeholder

```
Insert Service Discovery screenshot
```

---

## Stage 4 - SMB Enumeration

Additional SMB enumeration was performed from Kali.

Tools used:

- enum4linux-ng
- crackmapexec (NetExec)

Unauthenticated enumeration confirmed:

- SMB Signing Enabled
- SMBv3 Enabled
- Domain Name
- Computer Name
- Operating System

Anonymous RPC and SMB sessions were denied.

This behavior is expected in a properly configured Active Directory environment.

---

Screenshot Placeholder

```
Insert enum4linux screenshot
```

---

Screenshot Placeholder

```
Insert NetExec SMB enumeration screenshot
```

---

## Stage 5 - Valid Domain Authentication

A legitimate domain user account was used to authenticate against the Domain Controller.

Command executed:

```bash
netexec smb 192.168.100.10 \
-u emma.davis \
-p Password123!
```

Authentication completed successfully.

No exploitation techniques were used.

The objective was to generate authentication telemetry for investigation.

---

Screenshot Placeholder

```
Insert successful NetExec authentication screenshot
```

---

# Wazuh Investigation

After authentication, the Wazuh dashboard was reviewed.

Filtering was performed using:

```
Event ID 4624
```

Observed events included:

- Successful Remote Logon
- NTLM Authentication
- Anonymous Logon
- Network Logon

The originating source IP was correctly identified as:

```
192.168.100.40
```

which corresponds to:

```
Kali-Attacker01
```

---

Screenshot Placeholder

```
Insert Wazuh Dashboard overview
```

---

Screenshot Placeholder

```
Insert Events tab
```

---

Screenshot Placeholder

```
Insert Event Details
```

---

# Investigation Findings

## Detection 1

Successful Remote Logon

User

```
emma.davis
```

Authentication Package

```
NTLM
```

Logon Type

```
3
```

Source IP

```
192.168.100.40
```

Destination

```
CYBER-DC01
```

---

## Detection 2

Anonymous Logon

Observed immediately before the authenticated session.

This behavior is expected during SMB negotiation and IPC$ communication.

---

## Wazuh Alert

Wazuh classified the authentication as:

```
Successful Remote Logon Detected

Possible Pass-the-Hash Attack
```

During investigation, it was confirmed that:

- Valid credentials were supplied.
- No NTLM hashes were used.
- No credential dumping occurred.
- No Pass-the-Hash technique was executed.

This alert represents a heuristic detection rather than confirmation of malicious activity.

---

# MITRE ATT&CK Mapping

| Technique | ID |
|------------|----|
| Network Service Discovery | T1046 |
| System Network Configuration Discovery | T1016 |
| System Owner User Discovery | T1033 |
| Remote Services | T1021 |
| SMB/Windows Admin Shares | T1021.002 |
| Valid Accounts | T1078 |

---

# Lessons Learned

- Active Directory reconnaissance generates valuable telemetry even without exploitation.
- Wazuh successfully correlated Windows Security Event 4624 into high level authentication alerts.
- NTLM network logons can trigger heuristic detections that resemble Pass-the-Hash activity.
- Analysts must validate alerts before concluding malicious behavior.
- Proper investigation requires correlating the originating host, authentication method, and Windows event data.

---

# Future Improvements

- Deploy a Wazuh Agent on Cyber-DC01.
- Collect Domain Controller Security Events.
- Monitor Kerberos authentication events.
- Monitor NTLM authentication events.
- Detect failed authentication attempts.
- Create custom Wazuh detection rules.
- Expand investigation to include Kerberoasting and AS-REP Roasting detections.

---

# Conclusion

This project demonstrates a complete SOC investigation workflow beginning with reconnaissance from an attacker workstation and ending with detection, analysis, and validation inside Wazuh.

Rather than relying solely on SIEM alerts, the investigation correlated authentication events, source IP addresses, Windows Security logs, and attacker activity to determine that the observed alert represented legitimate authentication rather than an actual Pass-the-Hash attack.

The resulting workflow closely resembles the triage process performed by SOC analysts when investigating authentication-related alerts in enterprise Active Directory environments.
