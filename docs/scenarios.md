# Detection scenario playbook

This lab is about **detection engineering**: for each adversary technique, what telemetry it produces, which rule should fire, and how an analyst triages the alert. The offensive side is run from KALI-01 with standard, well-documented tools; this file deliberately stays at the technique level and focuses on the blue-team response. Fill **Observed** after each run (alert ID, timestamp, screenshot in `/screenshots`).

Targets: DC-01 `192.168.10.10`, WIN-01 `192.168.10.20`, LNX-01 `192.168.10.30`.

---

## 1. Network discovery — T1046
- **Technique:** host/port scanning across the subnet.
- **Telemetry:** many connection attempts from one source to many ports/hosts, seen by Suricata on IDS-01.
- **Detects:** Suricata `sid:1000001` (threshold-based scan signature) → forwarded to Wazuh.
- **Triage:** confirm source IP, scope (how many hosts/ports), whether it precedes other activity. In this lab the source is always KALI-01.
- **Observed:** _to run_

## 2. SSH brute force on LNX-01 — T1110.001
- **Technique:** repeated failed SSH authentication.
- **Telemetry:** `/var/log/auth.log` failed-password lines, collected by the Wazuh agent.
- **Detects:** Wazuh built-in sshd rules — **5710** (failed login) escalating to **5712** (brute-force, multiple failures in a short window).
- **Triage:** source IP, targeted usernames, whether any attempt succeeded (rule 5715). Response: block source, review account.
- **Observed:** _to run_

## 3. RDP / SMB password spraying — T1110.003
- **Technique:** a few passwords tried against many domain accounts (low-and-slow to dodge lockout).
- **Telemetry:** Windows Security event **4625** (failed logon) across multiple accounts; connection bursts seen by Suricata.
- **Detects:** Wazuh Windows auth rules (60122 / 60204 family) + Suricata `sid:1000002-3` (RDP/SMB bursts).
- **Triage:** pivot on source IP and the set of targeted accounts; a spray looks like *one source → many accounts, few attempts each*, unlike a brute force.
- **Observed:** _to run_

## 4. Kerberoasting — T1558.003
- **Technique:** requesting Kerberos service tickets for SPN-bearing service accounts to crack offline.
- **Telemetry:** event **4769** (Kerberos service ticket) with encryption type **0x17 (RC4)** and a non-machine account.
- **Detects:** custom Wazuh rule **100110** (see `wazuh/local_rules.xml`).
- **Triage:** which account requested which SPN, from where; baseline normal ticket activity to cut false positives. Requires Kerberos auditing enabled (see `setup.md §6`).
- **Observed:** _to run_

## 5. Privilege escalation — user added to Domain Admins — T1098
- **Technique:** adding an account to a privileged group for persistence.
- **Telemetry:** events **4728 / 4732 / 4756** (member added to a security-enabled group).
- **Detects:** custom Wazuh rule **100100** scoped to privileged groups (Domain Admins, Enterprise Admins, Administrators).
- **Triage:** who made the change, which account was added, was it authorized. High-value, low-noise alert.
- **Observed:** _to run_

## 6. Defense evasion — security log cleared — T1070.001
- **Technique:** clearing the Windows Security event log to destroy evidence.
- **Telemetry:** event **1102** (audit log cleared).
- **Detects:** custom Wazuh rule **100120**.
- **Triage:** treat as high severity — legitimate clears are rare. Correlate with activity just before the gap.
- **Observed:** _to run_

---

### Triage template
```
Alert ID / rule:
Time (UTC):
Source → target:
ATT&CK technique:
True / false positive:
Action taken:
Screenshot:
```
