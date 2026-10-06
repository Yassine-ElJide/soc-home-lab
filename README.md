# SOC Home Lab: Detection with Wazuh & Suricata

> A virtualized SOC lab on VirtualBox: an Active Directory domain monitored by Wazuh (SIEM/XDR) and Suricata (network IDS), attacked from Kali Linux, to practice detection engineering and alert triage.
> Built while preparing for the **Cisco CyberOps Associate** certification.

![Wazuh agents](screenshots/03-wazuh-agents.jpg)

## Architecture

```mermaid
flowchart LR
  subgraph NET["Isolated lab network 192.168.10.0/24 (no internet)"]
    K[KALI-01<br>attacker]
    DC[DC-01 · 192.168.10.10<br>Windows Server 2016<br>AD DS + DNS + DHCP · soc.local]
    W[WIN-01 · 192.168.10.20<br>Windows 11 Pro<br>domain workstation]
    L[LNX-01 · 192.168.10.30<br>Ubuntu 24.04]
    IDS[IDS-01 · 192.168.10.40<br>Ubuntu 24.04 · Suricata]
    SOC[SOC-01<br>Wazuh 4.14.2<br>manager · indexer · dashboard]
  end
  K -->|attacks| DC
  K -->|attacks| W
  K -->|attacks| L
  K -. traffic captured in promiscuous mode .-> IDS
  DC -->|agent| SOC
  W -->|agent| SOC
  L -->|agent| SOC
  IDS -->|eve.json via agent| SOC
```

| VM | Role | OS | IP | Wazuh agent |
| --- | --- | --- | --- | --- |
| SOC-01 | Wazuh all-in-one (manager, indexer, dashboard) | Ubuntu | - | - |
| DC-01 (VM name DC-Main) | Domain controller `soc.local` (AD DS, DNS, DHCP), 3 vCPU, 4 GB RAM, 50 GB | Windows Server 2016 | 192.168.10.10 | 004 |
| WIN-01 | Domain workstation | Windows 11 Pro | 192.168.10.20 | 003 |
| LNX-01 | Linux server (SSH) | Ubuntu 24.04 LTS | 192.168.10.30 | 002 |
| IDS-01 | Suricata network IDS | Ubuntu 24.04 LTS | 192.168.10.40 | 005 |
| KALI-01 | Attacker | Kali Linux | DHCP from DC-01 | - |

Host: laptop with AMD Ryzen 9 8945HS, Oracle VirtualBox.

## Stack
- **Wazuh 4.14.2**: SIEM/XDR, log collection, decoding, detection rules, alerting, dashboards
- **Suricata 7**: network IDS, alerts shipped to Wazuh through `eve.json`
- **Active Directory** (Windows Server 2016): target environment
- **Kali Linux**: attack simulation
- **VirtualBox**: virtualization, internal network for isolation

## Repository layout
| Path | Content |
| --- | --- |
| [`docs/setup.md`](docs/setup.md) | Build steps: network, Wazuh install, agent enrollment, Suricata integration, Windows audit policy |
| [`docs/scenarios.md`](docs/scenarios.md) | Attack scenario playbook: expected telemetry, ATT&CK mapping, triage notes |
| [`wazuh/local_rules.xml`](wazuh/local_rules.xml) | Custom Wazuh rules for AD attacks (privileged group changes, Kerberoasting, log clearing, password spraying) |
| [`wazuh/ossec-agent-ids.conf`](wazuh/ossec-agent-ids.conf) | Agent config snippet on IDS-01 to forward Suricata alerts |
| [`suricata/local.rules`](suricata/local.rules) | Custom Suricata signatures (port scan, RDP/SMB connection bursts) |
| [`screenshots/`](screenshots) | Lab evidence |

## Lab status
- [x] Network and 6 VMs built, isolated from the internet
- [x] AD domain `soc.local` promoted on DC-01
- [x] Wazuh 4.14.2 deployed, 4 agents enrolled (DC-01, WIN-01, LNX-01, IDS-01)
- [ ] Fix IDS-01 agent connectivity (shown *disconnected* in the screenshot)
- [ ] Deploy custom rules from this repo and validate them with `wazuh-logtest`
- [ ] Run the scenarios in [`docs/scenarios.md`](docs/scenarios.md)

## Attack scenarios
Full playbook with triage steps: [`docs/scenarios.md`](docs/scenarios.md).

| # | Scenario | MITRE ATT&CK | Expected detection |
| --- | --- | --- | --- |
| 1 | Network discovery with Nmap | T1046 | Suricata `sid:1000001` → Wazuh |
| 2 | SSH brute force on LNX-01 | T1110.001 | Wazuh sshd rules 5760 → 5763 (valid user), 5710 → 5712 (unknown user) |
| 3 | RDP / SMB password spraying on domain accounts | T1110.003 | Windows 4625 → Wazuh 60122 → custom rule 100130, Suricata `sid:1000002-3` |
| 4 | Kerberoasting (SPN service account) | T1558.003 | Event 4769 with RC4 → custom rule 100110 |
| 5 | User added to Domain Admins | T1098.007 | Events 4728 / 4732 / 4756 → custom rule 100100 |
| 6 | Security log cleared | T1070.001 | Event 1102 → custom rule 100120 |

## Screenshots
| | |
| --- | --- |
| ![VMs](screenshots/01-virtualbox-vms.jpg) | **VirtualBox**: the six lab VMs running (SOC-01, IDS-01, LNX-01, WIN-01, KALI-01, DC-Main). |
| ![DC](screenshots/02-dc01-soc-local.jpg) | **DC-01**: Windows Server 2016 promoted as domain controller of `soc.local`, no internet access. |
| ![Agents](screenshots/03-wazuh-agents.jpg) | **Wazuh 4.14.2**: 4 enrolled agents across Windows and Linux, IDS-01 disconnected at capture time. |

## What I learned
- Sizing a full SOC stack on one laptop: the Wazuh indexer is the most memory-hungry component, so RAM is planned around SOC-01 first.
- Agent names are fixed at enrollment: DC-01 shows as `WIN-6C2HILDD4A1` because it was enrolled before the rename.
- Default Windows audit policy is not enough. Kerberos, credential validation and group management auditing must be enabled by GPO before Wazuh can see those attacks.
- Custom Wazuh rules must hang off the most specific built-in rule (Wazuh only follows the first matching child), otherwise they never fire.

## Next steps
- Add Sysmon on WIN-01 and DC-01 for process-level telemetry
- Map every alert to MITRE ATT&CK in a Wazuh dashboard
- Document false positives and tune noisy rules

> ⚠️ Isolated lab network only. No production data.
