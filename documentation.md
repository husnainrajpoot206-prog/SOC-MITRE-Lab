# SOC Home Lab — MITRE ATT&CK Detection Lab
**Muhammad Husnain | SOC Analyst | Lahore, Pakistan | April 2026**

---

## Project Overview

This project focuses on building a hands-on SOC home lab to simulate 
real attack scenarios and analyze detection behavior using Wazuh SIEM 
against a Windows 7 target machine.

The objective was not just to generate alerts, but to validate logging 
configurations, test visibility across different attack techniques, and 
identify detection gaps under real conditions — including cases where 
attacks were executed correctly but went completely undetected.

Using a Kali Linux attacker and a Windows 7 victim connected over a 
Host-Only network, multiple MITRE ATT&CK techniques were executed and 
monitored through Wazuh. While several techniques were successfully 
detected, others exposed fundamental limitations in Windows 7 logging 
and SIEM correlation — limitations that are not visible unless you 
actually run the attacks and trace the failure back to its root cause.

This lab highlights a key reality in SOC work: **SIEM effectiveness 
depends entirely on the quality of underlying log sources.** Without 
proper endpoint visibility, even correctly configured detection rules 
will silently fail.

The outcome is a practical understanding of how detection works, where 
it breaks, and how those gaps must be documented rather than ignored.

**Lab Duration:** April 2026 — 11 days  
**Collaboration:** Parallel lab run by Wilfred Amimo using 
Splunk + Windows 10 for cross-tool comparison

---

## Lab Architecture

| Component | Details | Purpose |
|-----------|---------|---------|
| Attacker Machine | Kali Linux (VirtualBox) | Execute attack techniques mapped to MITRE ATT&CK |
| Victim Machine | Windows 7 (VirtualBox) + Wazuh Agent | Generate logs and simulate compromised endpoint |
| SIEM | Wazuh 4.7.5 (Manager + Dashboard + Indexer) | Centralized log collection, correlation, and alerting |
| Network | Host-Only Adapter (192.168.56.x) | Isolated lab network for controlled attack simulation |
| Hypervisor | VirtualBox | Virtual environment to run and manage lab machines |

### Network Diagram
[Kali Linux - 192.168.56.101]
|
Host-Only Adapter
|
[Windows 7 - 192.168.56.102]
|
Wazuh Agent → Wazuh Manager (Kali)
|
Wazuh Dashboard → https://192.168.56.101

---

## Environment Setup

### Wazuh Installation on Kali
Wazuh was installed on Kali Linux as the SIEM manager. 
The installation included three core components:

- **wazuh-manager** — Core engine for log collection and rule processing
- **wazuh-indexer** — OpenSearch-based storage for security events
- **wazuh-dashboard** — Web interface for alert visualization

### Critical Issue: RAM Constraint
The host machine had 8GB RAM. Running both Kali and Windows 7 
simultaneously caused wazuh-indexer to consume 2.1GB RAM and crash 
repeatedly with a SIGKILL signal.

**Error observed:**
Active: failed (Result: signal) since Mon 2026-04-27
Main PID: 764 (code=killed, signal=KILL)
Mem peak: 2.1G

**Resolution:** Windows 7 VM was shut down during Wazuh setup. 
Once wazuh-indexer stabilized, Windows 7 was restarted. This 
became a recurring workflow throughout the lab.

### Network Configuration Issue
Initial setup used NAT adapter for both VMs. This meant:
- Kali could access internet
- Windows 7 could access internet
- But neither VM could communicate with the other

**Resolution:** Both VMs were reconfigured to Host-Only Adapter.
This enabled VM-to-VM communication on the 192.168.56.x subnet
but removed internet access from both machines.

### Wazuh Agent Deployment Problem
With no internet on Windows 7 (Host-Only network), the standard 
agent download method failed. Browser-based download was not possible.

**Resolution:** A Python HTTP server was run on Kali to serve 
the MSI file locally:

```bash
# Download MSI on Kali (which had dual adapters - NAT + Host-Only)
wget https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi

# Serve it over HTTP
python3 -m http.server 8080
```

Windows 7 browser then accessed:
http://192.168.56.101:8080
And downloaded the MSI directly from Kali. This approach bypassed 
the need for internet access on the target machine entirely.

---

## Challenges Faced

| Challenge | Root Cause | Resolution |
|-----------|------------|------------|
| Wazuh indexer crashing with SIGKILL | 8GB host RAM insufficient for dual VM + Wazuh indexer simultaneously | Shut down Windows 7 VM during Wazuh setup; restarted after indexer stabilized |
| VM-to-VM communication failure | Both VMs on NAT — no internal routing between them | Switched both VMs to Host-Only Adapter (192.168.56.x subnet) |
| Wazuh dashboard not loading | Indexer not fully initialized — dashboard depends on indexer being healthy | Waited for full indexer startup, monitored via systemctl status |
| Wazuh agent MSI download failure | No internet on Windows 7 after Host-Only reconfiguration | Used Python HTTP server on Kali to transfer MSI over local network |
| Copy-paste not working between VMs | VirtualBox Guest Additions not configured | Installed Guest Additions; worked around by manually typing short commands |
| Kali also lost internet after Host-Only | Host-Only adapter replaces NAT, cutting external access | Added second adapter (NAT) to Kali — Adapter 1: Host-Only, Adapter 2: NAT |
| T1059 not detected despite audit policy | Windows 7 Event Channel does not forward process creation via Security channel | Documented as detection gap — root cause traced to OS-level telemetry limitation |

---

## Attack Simulations

### T1078 — Valid Accounts

**Objective:** Simulate misuse of a locally created account to evaluate 
Wazuh's ability to detect authenticated but unauthorized access behavior.

**Attack Logic:**
An attacker who has gained initial access to a system often creates 
a new local account to maintain persistence. By adding it to the 
Administrators group, they ensure elevated access even if the original 
compromise is discovered.

**Execution Steps:**
```powershell
net user attacker Password123! /add
net localgroup administrators attacker /add
runas /user:attacker cmd
```

**Observed Behavior:**
A new local user "attacker" was created with administrator privileges.
The runas command opened a new CMD session under the attacker account,
simulating credential-based lateral movement on the same machine.

**Detection Result:** Detected ✅

**Wazuh Alert Details:**

| Field | Value |
|-------|-------|
| Technique | T1078 — Valid Accounts |
| Event ID | 4624 |
| Event Type | Windows Logon Success |
| Target User | attacker |
| Subject User | Husnain |
| Logon Type | 2 (Interactive) |
| Severity | Level 3 (Medium) |
| Agent | Husnain-PC (192.168.56.102) |

**Key Insight:**
Wazuh correctly identified the logon event and mapped it to T1078.
However, valid account logins are indistinguishable from legitimate 
user activity at the event level alone. In production SOC environments,
this technique requires behavioral baselining — alerting on first-time
logons, off-hours access, or accounts that have never logged in before.
Single-event detection is insufficient.

---

### T1098 — Account Manipulation

**Objective:** Test Wazuh's ability to detect unauthorized privilege 
escalation through local account and group modification.

**Attack Logic:**
After gaining initial access, attackers frequently modify existing 
accounts or create new privileged accounts to establish persistence 
and ensure continued elevated access.

**Execution Steps:**
```powershell
net user attacker2 Husnain123 /add
net localgroup administrators attacker2 /add
```

**Observed Behavior:**
A second attacker account was created and added to the Administrators
group. This triggered both a user creation event and a group membership
change event — two separate log entries that Wazuh correlated.

**Detection Result:** Detected ✅

**Wazuh Alert Details:**

| Field | Value |
|-------|-------|
| Technique | T1098 — Account Manipulation |
| Event Type | User Account Created + Group Membership Change |
| Severity | Medium |
| Rule | Local group modification activity detected |

**Key Insight:**
Privilege escalation through account manipulation is one of the most 
commonly overlooked activities in basic SIEM setups. Wazuh detected 
the group membership change successfully. However, in real environments 
this technique generates noise because legitimate IT administrators 
perform the same actions routinely. Detection effectiveness depends 
on role-based access baselining and anomaly detection — not just 
rule matching.

---

### T1484 — Domain Policy Modification

**Objective:** Evaluate Wazuh's detection of unauthorized changes to 
local group membership and security policy configuration.

**Attack Logic:**
Modifying group policies or adding accounts to privileged groups 
allows attackers to expand their foothold. This technique is often 
used post-compromise to ensure persistence across reboots.

**Execution Steps:**
```powershell
net localgroup administrators attacker /add
net localgroup administrators attacker2 /add
```

**Observed Behavior:**
Adding accounts to the local Administrators group triggered a 
group membership change event. Wazuh mapped this to T1484 through
security event correlation — specifically monitoring changes to 
privileged local groups.

**Detection Result:** Detected ✅

**Wazuh Alert Details:**

| Field | Value |
|-------|-------|
| Technique | T1484 — Domain Policy Modification |
| Event Type | Administrators Group Membership Change |
| Severity | High (Level 12) |
| Rule Trigger | Local privileged group modification |

**Key Insight:**
This was the highest severity alert generated in the lab — Level 12.
Wazuh correctly flagged changes to the Administrators group as a 
high-risk event. In production environments, this type of alert 
should trigger immediate investigation. However, distinguishing 
malicious group changes from legitimate IT administration requires 
change management integration — alerts alone are not sufficient 
without context about who authorized the change.

---

### T1059 — Command and Scripting Interpreter

**Objective:** Assess Wazuh's ability to detect command execution 
activity through Windows command-line and PowerShell.

**Attack Logic:**
Command execution is one of the most common attacker techniques.
After gaining access, attackers run reconnaissance commands to 
understand the environment before moving laterally or escalating.

**Execution Steps:**
```powershell
whoami
ipconfig
net user
systeminfo
```

**Observed Behavior:**
Commands were executed in PowerShell as Administrator. All commands
ran successfully from the attacker's perspective — output was visible.
However, Wazuh generated zero alerts for any of these commands.

**Detection Result:** Not Detected ❌

**Investigation Performed:**
This was not accepted as a simple failure. The following steps were 
taken to diagnose and attempt to fix the detection gap:

**Step 1 — Enabled Windows Audit Policy:**
```powershell
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable
gpupdate /force
```
Result: Policy applied successfully — confirmed with:
```powershell
auditpol /get /subcategory:"Process Creation"
# Output: Process Creation - Success and Failure
```

**Step 2 — Edited Wazuh Manager config (ossec.conf):**
```xml
<localfile>
  <log_format>eventchannel</log_format>
  <location>Security</location>
</localfile>
```

**Step 3 — Restarted both Wazuh Manager and Wazuh Agent:**
```bash
sudo systemctl restart wazuh-manager
net stop WazuhSvc
net start WazuhSvc
```

**Step 4 — Re-ran simulation and monitored dashboard**
Result: Still no T1059 alerts.

**Root Cause Identified:**
Windows 7 does not forward process creation events (Event ID 4688) 
through the Security event channel reliably. This is a known OS-level
telemetry limitation. Unlike Windows 10+, Windows 7 requires Sysmon 
or additional third-party agents to capture process creation at the 
level needed for T1059 detection.

**Wazuh Alert Summary:**
- No correlated alerts generated
- No MITRE ATT&CK mapping triggered
- Command execution completely invisible to SIEM

**Key Insight:**
This was the most valuable finding in the entire lab. The attack 
succeeded. The audit policy was correctly configured. The SIEM rule 
was in place. Yet detection failed — because the log source itself 
was incomplete. This is exactly the kind of gap that exists in real 
SOC environments running legacy endpoints. The lesson: **validate 
your log sources before trusting your detections.**

---

### T1110 — Brute Force

**Objective:** Evaluate Wazuh's ability to detect repeated failed 
authentication attempts simulating password guessing against a 
Windows service.

**Attack Logic:**
Brute force attacks attempt to gain access by systematically trying
passwords from a wordlist. SMB is a common target because it exposes
Windows authentication over the network.

**Execution Steps:**
```bash
hydra -l administrator -P /usr/share/wordlists/rockyou.txt 192.168.56.102 smb
```

**Observed Behavior:**
Hydra initiated a brute force attack against the Windows 7 SMB 
service on port 445 using the rockyou wordlist (14 million passwords).
The attack ran for several minutes before being stopped manually.

**Detection Result:** Not Detected ❌

**Root Cause:**
SMB on Windows 7 in this lab configuration did not generate 
consistent Event ID 4625 (failed logon) entries that Wazuh could 
aggregate. Without multiple failed logon events being forwarded 
to Wazuh, no threshold-based brute force correlation was possible.

**Wazuh Alert Summary:**
- No brute force correlation alert triggered
- Failed logon attempts not aggregated
- No threshold-based detection observed

**Key Insight:**
Brute force detection is not about detecting a single failed login —
it requires correlating multiple failures within a time window from 
the same source. Without that aggregation, each failed attempt looks
like an isolated harmless event. In production environments, this 
requires tuned threshold rules: for example, "5 failed logins from 
the same IP within 60 seconds = brute force alert." The absence of 
this detection in the lab highlights a missing correlation layer.

---

### T1562.001 — Impair Defenses (Disable Security Tools)

**Objective:** Evaluate Wazuh's ability to detect attempts to 
disable endpoint security monitoring by stopping the Wazuh agent.

**Attack Logic:**
Before executing noisy attack techniques, sophisticated attackers 
often disable security tools to reduce detection coverage. Stopping
the monitoring agent is one of the most direct ways to create a 
blind spot in SIEM visibility.

**Execution Steps:**
```powershell
net stop WazuhSvc
net start WazuhSvc
```

**Observed Behavior:**
The Wazuh agent service was manually stopped and restarted on 
Windows 7. This simulated an attacker attempting to impair 
endpoint monitoring before further malicious activity.

**Detection Result:** Detected ✅

**Wazuh Alert Details:**

| Field | Value |
|-------|-------|
| Technique | T1562.001 — Impair Defenses |
| Event Type | Security Agent Service Stop/Start |
| Severity | High |
| Rule Trigger | Wazuh agent service interruption |
| Event Source | Windows Service Control Manager |

**Key Insight:**
Wazuh successfully detected its own agent being stopped — 
generating a high-severity alert in real time. This is a 
critical detection capability. However, in production environments,
attackers combine agent tampering with log clearing and event 
log deletion, which makes detection significantly more complex.
Centralized logging with backup telemetry is essential to 
maintain visibility even when endpoint agents are compromised.

---

## Findings Summary

| Technique | Name | Result | Severity | Key Finding |
|-----------|------|--------|----------|-------------|
| T1078 | Valid Accounts | Detected ✅ | Medium | Logon events correlated but requires behavioral baselining |
| T1098 | Account Manipulation | Detected ✅ | Medium | Group membership changes detected through event correlation |
| T1484 | Domain Policy Modification | Detected ✅ | High (L12) | Highest severity alert — Administrators group change flagged |
| T1562.001 | Impair Defenses | Detected ✅ | High | Agent service interruption detected in real time |
| T1059 | Command Execution | Not Detected ❌ | N/A | Windows 7 telemetry gap — process creation not forwarded |
| T1110 | Brute Force | Not Detected ❌ | N/A | SMB failed logons not aggregated — no threshold correlation |

**Detection Rate: 4/6 techniques — 67%**

---

## Lessons Learned

1. **Log source quality determines SIEM effectiveness.** Wazuh 
performed correctly — the failure was at the endpoint telemetry 
layer. A SIEM is only as good as the logs it receives.

2. **Not all attack techniques are equally visible.** Service 
tampering generates immediate high-severity alerts. Command 
execution on legacy OS can be completely invisible. Understanding
this difference is essential for SOC prioritization.

3. **Default Windows logging is not SOC-ready.** Audit policies 
must be manually configured before a SIEM becomes useful. Without
this step, entire attack categories go undetected.

4. **Detection gaps must be actively investigated, not assumed.** 
T1059 was not accepted as "just not working." The audit policy 
was configured, Wazuh config was edited, services were restarted,
and the root cause was traced to the OS event channel. That 
investigation process is real SOC work.

5. **Context separates detections from noise.** T1078 and T1098 
generated alerts — but the same events occur during legitimate 
administration. Real detection requires behavioral baselines,
not just rule matching.

6. **Missing detections are as important as successful ones.**
The two undetected techniques revealed more about the monitoring
environment than the four that were caught. Documenting blind 
spots is a core SOC responsibility.

7. **OS version directly impacts detection coverage.** The 
parallel lab running Splunk + Windows 10 detected T1059 with 
114 events. The same technique on Windows 7 with Wazuh generated
zero. Tool selection and endpoint OS are inseparable from 
detection strategy.

---

## Quick Reference

**Lab Stack:**
- Attacker: Kali Linux (VirtualBox) — 192.168.56.101
- Victim: Windows 7 + Wazuh Agent — 192.168.56.102
- SIEM: Wazuh 4.7.5
- Network: Host-Only Adapter

**MITRE ATT&CK Techniques:**
- T1078 — Valid Accounts ✅
- T1098 — Account Manipulation ✅
- T1484 — Domain Policy Modification ✅
- T1562.001 — Impair Defenses ✅
- T1059 — Command Execution ❌ (Windows 7 telemetry gap)
- T1110 — Brute Force ❌ (SMB correlation missing)

**Detection Rate: 4/6 — 67%**

**Key Tools Used:**
- Wazuh 4.7.5 (SIEM)
- Hydra (Brute Force)
- Python HTTP Server (Agent Transfer)
- VirtualBox (Hypervisor)
- PowerShell (Attack Simulation)
