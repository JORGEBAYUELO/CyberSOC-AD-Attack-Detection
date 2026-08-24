# CyberSOC-AD-Attack-Detection

![HeroBanner](/images/AD-Attack-Detection.jpg)

> This project demonstrates how Active Directory reconnaissance and authentication activity can be detected using Sysmon and Wazuh in a Windows domain environment.

## Project Overview

This project demonstrates the simulation, detection, correlation, and investigation of authentication based attacks within a controlled Active Directory cybersecurity lab.

The lab uses a Windows Active Directory environment, a Kali Linux attacker system, and Wazuh as the SIEM and detection platform.

The project builds on the Active Directory and monitoring infrastructure established in the CyberSOC HomeLab Foundation project and focuses on the defensive workflow of identifying suspicious authentication activity.

The investigation follows a realistic security operations workflow:

**Reconnaissance → Attack Simulation → Windows Telemetry → SIEM Ingestion → Detection Engineering → Validation → Investigation**

The project culminates in the creation and validation of a custom Wazuh correlation rule capable of detecting repeated authentication failures originating from the same source IP address.

---

## Project Objectives

The primary objectives of this project were to:

- Simulate reconnaissance against a Windows Active Directory environment
- Generate controlled SMB authentication activity from Kali Linux
- Generate failed Windows authentication events against a domain joined workstation
- Analyze Windows Security Event ID 4625
- Ingest Windows security telemetry into Wazuh
- Investigate authentication events through Wazuh Threat Hunting
- Develop a custom Wazuh correlation rule
- Detect repeated authentication failures within a defined time window
- Reduce false positives by correlating authentication attempts by source IP
- Map the custom detection to MITRE ATT&CK
- Validate the detection using positive and negative control tests
- Perform a SOC style investigation of the resulting alert

---

# Lab Architecture

The project uses an isolated VMware network:

`192.168.100.0/24`

The primary systems used during the project were:

| System | Role | Operating System | IP Address |
|---|---|---|---|
| `CYBER-DC01` | Active Directory Domain Controller / DNS | Windows Server 2022 | `192.168.100.10` |
| `WIN11-CLIENT01` | Domain joined monitored endpoint | Windows 11 Enterprise | `192.168.100.20` |
| `WAZUH-SERVER01` | SIEM / Wazuh Manager | Linux | Lab SIEM Network |
| `KALI-ATTACKER01` | Attack simulation system | Kali Linux | `192.168.100.40` |

Active Directory domain:

```text
cybersoc.lab
```

The Wazuh agent was installed on `WIN11-CLIENT01`.

The Domain Controller was intentionally left without a Wazuh agent for this project. Authentication attack simulations were therefore directed against the monitored Windows 11 endpoint so the resulting Windows Security events could be collected and analyzed by Wazuh.

### Architecture Diagram

![Architectural Diagram](/images/Foundational-Architectural-Diagram.jpg)

---

# Active Directory Environment

The `cybersoc.lab` Active Directory environment contains organizational units representing a small enterprise environment.

OUs include:

```text
Admin
├── Domain Admins
├── Server Admins
└── Service Accounts

Groups
├── Distribution
└── Security

Servers
├── Infrastructure
└── SIEM

User Accounts
├── Finance
├── HR
├── IT
└── Marketing

Workstations
├── Finance
├── HR
├── IT
└── Marketing
```

Security groups were used to represent department based access.

For example, the HR user `emma.davis` was assigned membership in the appropriate HR security group.

The Linux based Wazuh server was not joined to the Active Directory domain. As a result, no Wazuh computer object exists in Active Directory.

### Active Directory Structure

![Active Directory Structure](/images/ad-structure.png)

### User Security Group Membership

![User Group Membership](/images/user-group-membership.png)

---

# Attack System Preparation

`KALI-ATTACKER01` was used as the adversary simulation system.

The Kali system was placed on the isolated CyberSOC lab network and used to perform reconnaissance and controlled authentication testing against the Windows environment.

Attack and reconnaissance output generated during the project remained inside the Kali VM.

Files included:

```text
01-nmap-host-discovery.txt
02-service-discovery.txt
03-netexec-authentication.txt
```

These files were used as working evidence inside `KALI-ATTACKER01` and were not added as additional directories to this GitHub repository.

---

# Phase 1 — Network Reconnaissance

The first stage of the attack simulation involved identifying systems available on the isolated lab network.

Nmap was used from `KALI-ATTACKER01` to perform host discovery.

The objective was to identify potential Windows systems before performing more targeted service enumeration.

### Host Discovery

![Nmap Host Discovery](/images/nmap-host-discovery.png)

---

# Phase 2 — Service Discovery

After identifying available systems, additional reconnaissance was performed to determine which network services were exposed.

The Windows systems exposed services associated with Active Directory and Windows networking.

Of particular interest for this project was SMB on TCP port `445`.

The discovery of SMB provided a realistic path for controlled authentication testing against the Windows environment.

### Service Enumeration

![Service Discovery](/images/service-discovery.png)

---

# Phase 3 — SMB Authentication Testing

NetExec was used from `KALI-ATTACKER01` to interact with SMB services in the lab.

Authentication testing was performed using the `emma.davis` domain account.

Example syntax used during controlled authentication testing:

```bash
netexec smb 192.168.100.20 \
-u 'emma.davis' \
-p 'Password123!'
```

The purpose of this activity was not exploitation. The objective was to generate controlled Windows authentication telemetry that could subsequently be analyzed through the endpoint and SIEM.

### Kali Authentication Activity

![NetExec Authentication](/images/netexec-authentication.png)

---

# Phase 4 — Windows Authentication Telemetry

Authentication activity against `WIN11-CLIENT01` generated Windows Security events.

A primary event used throughout the investigation was:

```text
Event ID: 4625
Description: An account failed to log on
```

PowerShell was used locally on `WIN11-CLIENT01` to inspect the Windows Security log.

Example:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName   = 'Security'
    Id        = 4625
    StartTime = (Get-Date).AddHours(-1)
} |
Select-Object TimeCreated, Id, Message |
Format-List
```

The resulting event contained several useful investigation attributes, including:

```text
Account Name:        emma.davis
Account Domain:      cybersoc.lab
Logon Type:          3
Source Network Address: 192.168.100.40
Authentication Package: NTLM
Failure Reason:      Unknown user name or bad password
```

Logon Type `3` represents a network logon, which was consistent with the SMB authentication activity generated remotely from Kali.

### Windows Event ID 4625

![Windows Event 4625](/images/windows-event-4625.png)

---

# Phase 5 — Wazuh Telemetry Validation

The Wazuh agent installed on `WIN11-CLIENT01` collected the relevant Windows Security events and forwarded them to the Wazuh manager.

The same authentication activity observed locally through PowerShell was visible through Wazuh Threat Hunting.

This established the telemetry path:

```text
KALI-ATTACKER01
        |
        | SMB Authentication
        v
WIN11-CLIENT01
        |
        | Windows Security Event 4625
        v
Wazuh Agent
        |
        v
WAZUH-SERVER01
        |
        v
Wazuh Threat Hunting
```

The timestamps observed in Windows Event Viewer / PowerShell were correlated with the timestamps displayed by Wazuh.

This confirmed that the endpoint and SIEM were observing the same authentication activity.

### Wazuh Authentication Events

![Wazuh Authentication Events 01](/images/wazuh-authentication-events-01.png)
![Wazuh Authentication Events 02](/images/wazuh-authentication-events-02.png)

### Endpoint and SIEM Timestamp Correlation

![Windows Event Correlation](/images/windows-event-correlation.png)

![Wazuh Event Correlation](/images/wazuh-event-correlation.png)

![Wazuh Event Correlation 02](/images/wazuh-event-correlation-02.png)

---

# Phase 6 — Failed Authentication Detection Engineering

After validating telemetry ingestion, the project moved from event visibility into detection engineering.

Repeated failed authentication attempts were intentionally generated against `WIN11-CLIENT01`.

Each failed attempt generated Windows Security Event ID:

```text
4625
```

Wazuh initially processed these as individual authentication failure alerts.

A typical event was classified as:

```text
Logon Failure - Unknown user or bad password
```

with Wazuh rule:

```text
60122
```

The telemetry established that Wazuh could see individual failed authentication attempts, but individual events alone did not provide higher level correlation for a burst of failures.

### Multiple Authentication Failures

![Wazuh Event Correlation](/images/wazuh-event-correlation.png)

---

# Custom Wazuh Detection Rule

A custom correlation rule was developed on `WAZUH-SERVER01`.

The objective was to detect multiple Windows authentication failures occurring within a short period.

The custom rule was assigned:

```text
Rule ID: 100100
Level: 10
Frequency: 5
Timeframe: 60 seconds
```

The rule was configured in:

```text
/var/ossec/etc/rules/local_rules.xml
```

The detection was based on repeated matches of the underlying Windows authentication failure rule.

The resulting custom rule:

```xml
<group name="windows_authentication,authentication_failed,cybersoc_lab,">

  <rule id="100100" level="10" frequency="5" timeframe="60">
    <if_matched_sid>60122</if_matched_sid>
    <same_srcip />
    <description>CYBERSOC LAB: Potential password guessing detected - multiple Windows authentication failures</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>

</group>
```

The `<same_srcip />` condition was important because it required the correlated authentication failures to originate from the same source IP address.

Without this condition, unrelated authentication failures from different systems could potentially contribute to the same correlation threshold.

### Custom Rule Configuration

![Custom Wazuh Rule](/images/custom-wazuh-rule.png)

The rule configuration was validated successfully and the Wazuh manager was restarted before testing.

---

# MITRE ATT&CK Mapping

The custom detection was mapped to:

| Attribute | Mapping |
|---|---|
| MITRE ATT&CK ID | `T1110` |
| Technique | Brute Force |
| Tactic | Credential Access |

This mapping allows the detection to communicate not only what happened technically, but also the adversary behavior represented by the authentication activity.

---

# Detection Validation

Detection engineering should include testing both conditions where an alert **should fire** and conditions where it **should not fire**.

For that reason, Rule `100100` was tested using both positive and negative controls.

---

## Positive Control

Five failed authentication attempts were generated from `KALI-ATTACKER01` against `WIN11-CLIENT01` within the configured 60 second correlation window.

The source IP was:

```text
192.168.100.40
```

Wazuh first generated the individual authentication failure events and then successfully generated:

```text
Rule ID: 100100
Rule Level: 10
Description:
CYBERSOC LAB: Potential password guessing detected - multiple Windows authentication failures
```

This confirmed that the detection successfully identified the intended behavior.

### Successful Rule Trigger

![Rule 100100 Positive Control](/images/rule-100100-positive-control.png)

### Rule Details and MITRE Mapping

![Rule 100100 Mitre](/images/rule-100100-mitre.png)

---

# False Positive Reduction

A second validation test was performed to determine whether authentication failures originating from different systems would incorrectly trigger the custom correlation rule.

The test intentionally distributed failed authentication attempts across different source IP addresses.

The individual attempts continued to generate the underlying Wazuh authentication failure rule:

```text
60122
```

However, the combined events did **not** generate a new Rule `100100` correlation alert.

This demonstrated that:

```xml
<same_srcip />
```

was functioning as intended.

The detection therefore evaluates behavior closer to:

```text
Five failed authentication attempts
FROM THE SAME SOURCE IP
within 60 seconds
```

rather than:

```text
Any five failed authentication attempts
within 60 seconds
```

This distinction reduces the likelihood that unrelated authentication failures from multiple systems will be incorrectly combined into a password guessing alert.

### Negative Control

![Negative Control](/images/negative-control.png)

### Source IP Validation

![Negative Control Source IP](/images/negative-control-source-ip.png)

---

# Phase 7 — SOC Investigation

Following successful detection engineering and validation, a final controlled attack sequence was generated from `KALI-ATTACKER01`.

The objective was to investigate the resulting detection as a SOC analyst rather than simply confirm that the rule triggered.

Five failed SMB authentication attempts were generated against the monitored endpoint.

Wazuh recorded the individual authentication failures and subsequently generated the custom correlation alert.

### Detection Timeline

The Wazuh event sequence demonstrated:

```text
Repeated Windows authentication failures
                |
                v
Individual Rule 60122 alerts
                |
                v
Correlation threshold reached
                |
                v
Custom Rule 100100
                |
                v
Potential Password Guessing
```

### Final Detection

![Final Detection Timeline 01](/images/final-detection-timeline-01.png)

![Final Detection Timeline 02](/images/final-detection-timeline-02.png)

---

# Investigation Findings

Analysis of the Wazuh telemetry produced the following findings:

| Investigation Field | Finding |
|---|---|
| Detection Rule | `100100` |
| Rule Level | `10` |
| Endpoint | `WIN11-CLIENT01` |
| Endpoint IP | `192.168.100.20` |
| Source IP | `192.168.100.40` |
| Target Account | `emma.davis` |
| Target Domain | `cybersoc.lab` |
| Windows Event ID | `4625` |
| Logon Type | `3` |
| Authentication Package | `NTLM` |
| Failure Activity | Unknown user or bad password |
| Correlation Frequency | `5` |
| Correlation Timeframe | `60 seconds` |
| MITRE ATT&CK ID | `T1110` |
| MITRE Tactic | Credential Access |
| MITRE Technique | Brute Force |

The source address `192.168.100.40` corresponded to `KALI-ATTACKER01`.

The destination endpoint `WIN11-CLIENT01` was the monitored Windows system receiving the SMB authentication requests.

### Authentication Event Details

![Final Detection Timeline 01](/images/final-detection-timeline-01.png)

### Detection and MITRE Details

![Final Detection Timeline 02](/images/final-detection-timeline-02.png)

---

# SOC Analyst Assessment

The investigation identified multiple failed network authentication attempts targeting the `emma.davis` domain account on `WIN11-CLIENT01`.

The authentication attempts originated from:

```text
192.168.100.40
```

which corresponded to the controlled `KALI-ATTACKER01` system.

The authentication activity generated Windows Security Event ID `4625` and used network Logon Type `3` with NTLM authentication.

Multiple authentication failures from the same source occurred within the configured detection window.

Wazuh correlated the individual failures and generated custom Rule `100100`, identifying the activity as potential password guessing.

The behavior was mapped to:

```text
MITRE ATT&CK T1110
Brute Force
Credential Access
```

Based on the frequency of authentication failures, common source address, targeted user account, and short time interval, the activity would warrant investigation in a production SOC environment.

In this lab, the activity was determined to be a **true positive generated by an authorized attack simulation**.

---

# Detection Logic Summary

The completed detection pipeline can be represented as:

```text
KALI-ATTACKER01
192.168.100.40
        |
        | SMB / NTLM Authentication
        v
WIN11-CLIENT01
192.168.100.20
        |
        | Windows Security Event 4625
        v
Wazuh Agent
        |
        v
WAZUH-SERVER01
        |
        | Rule 60122
        | Individual authentication failures
        v
Correlation Engine
        |
        | 5 failures
        | 60 seconds
        | Same source IP
        v
CUSTOM RULE 100100
        |
        v
Potential Password Guessing
        |
        v
MITRE ATT&CK T1110
Brute Force
Credential Access
```

---

# Detection Engineering Results

The final custom detection successfully met the intended requirements.

| Requirement | Result |
|---|---|
| Windows authentication telemetry collected | PASS |
| Event ID 4625 visible locally | PASS |
| Event ID 4625 visible in Wazuh | PASS |
| Source IP captured | PASS |
| Target account captured | PASS |
| Individual failures detected | PASS |
| Multiple failures correlated | PASS |
| Same source IP correlation | PASS |
| Positive control triggered Rule 100100 | PASS |
| Mixed source negative control suppressed Rule 100100 | PASS |
| MITRE ATT&CK mapping | PASS |
| SOC investigation completed | PASS |

---

# Skills Demonstrated

This project demonstrates practical experience with:

### Active Directory

- Active Directory Users and Computers
- Organizational Units
- Security groups
- Domain user accounts
- Domain joined Windows endpoints
- Windows authentication

### Windows Security Monitoring

- Windows Security Event Logs
- Event ID 4625
- Network Logon Type 3
- NTLM authentication
- PowerShell event log analysis
- Source IP attribution
- Authentication failure analysis

### SIEM and Detection Engineering

- Wazuh
- Wazuh agents
- Threat Hunting
- Windows event ingestion
- Custom Wazuh rules
- Rule correlation
- Frequency based detection
- Time based correlation
- Source IP correlation
- False positive reduction
- Detection validation

### Offensive Security Validation

- Kali Linux
- Nmap
- Network reconnaissance
- Service enumeration
- NetExec
- SMB authentication testing
- Controlled adversary simulation

### SOC Analysis

- Alert triage
- Event correlation
- Timeline analysis
- Source identification
- User attribution
- Detection validation
- Positive control testing
- Negative control testing
- True positive determination
- MITRE ATT&CK mapping

---

# Key Lessons Learned

## Telemetry Must Be Validated Before Detection Logic

Before creating a custom detection, the underlying telemetry was validated both locally on the Windows endpoint and remotely through Wazuh.

This ensured that detection engineering was based on known good data.

---

## Individual Events Do Not Always Represent the Full Behavior

A single Event ID `4625` can represent an ordinary authentication mistake.

Multiple failures occurring rapidly from the same source provide significantly more context.

Correlation transforms individual low context events into higher confidence security detections.

---

## Source Context Matters

The initial correlation concept focused on the number of authentication failures occurring within a time window.

Adding:

```xml
<same_srcip />
```

made the rule more precise by requiring the failures to originate from the same source.

The negative control demonstrated why this distinction matters.

---

## Detection Engineering Requires Negative Testing

A detection firing successfully does not prove that the rule is well designed.

The project therefore tested both:

```text
Behavior that SHOULD trigger the rule
```

and:

```text
Behavior that SHOULD NOT trigger the rule
```

This provided stronger evidence that Rule `100100` was behaving according to its intended logic.

---

## Endpoint and SIEM Evidence Should Correlate

The same authentication activity was examined through both Windows Security logs and Wazuh.

Matching event attributes and timestamps demonstrated the relationship between endpoint telemetry and centralized SIEM analysis.

---

# Project Outcome

The CyberSOC Active Directory Attack Detection project successfully demonstrated an end to end security monitoring workflow.

The project progressed from network reconnaissance and controlled authentication activity through endpoint telemetry collection, centralized SIEM monitoring, custom detection engineering, rule validation, false positive testing, and SOC investigation.

The completed workflow was:

```text
Reconnaissance
      ↓
Service Discovery
      ↓
Authentication Testing
      ↓
Windows Event Generation
      ↓
Wazuh Telemetry Collection
      ↓
Event Analysis
      ↓
Detection Engineering
      ↓
Correlation
      ↓
MITRE ATT&CK Mapping
      ↓
Positive Validation
      ↓
Negative Validation
      ↓
SOC Investigation
```

The final custom Wazuh detection identified repeated authentication failures from the same source IP within a defined time window and mapped the behavior to MITRE ATT&CK T1110.

The project demonstrates the relationship between offensive security testing and defensive security operations: controlled adversary behavior was used to generate telemetry, develop detection logic, validate alert behavior, reduce false positives, and conduct an evidence based investigation.

---

## Related CyberSOC Portfolio Projects

This project is part of the broader **CyberSOC Portfolio**, a collection of hands on cybersecurity projects focused on security operations, detection engineering, vulnerability management, SIEM, and incident response.
