# SOC Home Lab — MITRE ATT&CK Detection Lab

## Overview
A hands-on SOC home lab built to simulate and detect real cyberattacks using the MITRE ATT&CK framework.

## Lab Setup
| Component | Details |
|-----------|---------|
| Attacker Machine | Kali Linux (VirtualBox) |
| Victim Machine | Windows 7 (VirtualBox) |
| SIEM | Wazuh 4.7.5 |
| Network | Host-Only Adapter |

## Techniques Simulated
| Technique | Description | Result |
|-----------|-------------|--------|
| T1078 | Valid Accounts | Detected |
| T1098 | Account Manipulation | Detected |
| T1484 | Domain Policy Modification | Detected |
| T1059 | Command Execution | Not Detected (Windows 7 limitation) |
| T1110 | Brute Force | Not Detected (SMB blocked) |

## Key Findings
- Default Windows logging is not SOC ready — audit policies must be manually configured
- T1078 and T1098 generated clear alerts in Wazuh
- T1059 process creation events not captured on Windows 7 — known OS limitation
- Detection gaps are as important to document as detections

## Challenges Faced
- Wazuh indexer RAM issues on 8GB laptop — resolved by closing Windows 7 VM during Wazuh setup
- Host-only network adapter required for VM-to-VM communication
- Windows 7 Event Channel does not support process creation logging natively
- Wazuh agent manually transferred via Python HTTP server due to no internet on target VM

## Tools Used
- Wazuh SIEM
- Kali Linux
- Hydra
- VirtualBox
