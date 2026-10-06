# Detection scenario playbook

This lab is about **detection engineering**: for each adversary technique, what telemetry it produces, which rule should fire, and how an analyst triages the alert. The offensive side is run from KALI-01 with standard, well-documented tools. This file stays at the technique level and focuses on the blue-team response. Fill **Observed** after each run (alert ID, timestamp, screenshot in `/screenshots`).

Targets: DC-01 `192.168.10.10`, WIN-01 `192.168.10.20`, LNX-01 `192.168.10.30`.

---

## 1. Network discovery (T1046)
- **Technique:** host/port scanning across the subnet.
- **Telemetry:** many new TCP connections (SYN) from one source to many ports/hosts, seen by Suricata on IDS-01.
- **Detects:** Suricata `sid:1000001` (threshold-based scan signature), forwarded to Wazuh (built-in Suricata rules 86600/86601).
- **Triage:** confirm source IP, scope (how many hosts/ports), whether it precedes other activity. In this lab the source is always KALI-01.
- **Observed:** _to run_

## 2. SSH brute force on LNX-01 (T1110.001)
- **Technique:** repeated failed SSH authentication.
- **Telemetry:** `/var/log/auth.log` failed-password lines, collected by the Wazuh agent.
- **Detects:** Wazuh built-in sshd rules:
  - **5760** (failed password for a valid user), escalating to **5763** (brute force, 8 failures in 120 s from the same source)
  - **5710** (attempt with a non-existent user), escalating to **5712**
- **Triage:** source IP, targeted usernames, whether any attempt succeeded (rule 5715). Response: block source, review account.
- **Observed:** _to run_

## 3. RDP / SMB password spraying (T1110.003)
- **Technique:** a few passwords tried against many domain accounts (low and slow to avoid lockout).
- **Telemetry:**
  - **4625** (failed logon) on the targeted host (WIN-01 or DC-01)
  - **4776** (NTLM credential validation) and **4771** (Kerberos pre-auth failed) on DC-01
  - connection bursts seen by Suricata
- **Detects:** Wazuh 60122 (failed logon) feeding custom rule **100130** (one source, many different accounts), built-in 60204 (multiple failures), Suricata `sid:1000002-3`.
- **Triage:** pivot on source IP and the set of targeted accounts. A spray looks like *one source, many accounts, few attempts each*, unlike a brute force.
- **Observed:** _to run_

## 4. Kerberoasting (T1558.003)
- **Technique:** requesting Kerberos service tickets for SPN-bearing service accounts to crack them offline.
- **Telemetry:** event **4769** (Kerberos service ticket) with encryption type **0x17 (RC4)** for a non-machine account.
- **Detects:** custom Wazuh rule **100110** (see `wazuh/local_rules.xml`).
- **Triage:** which account requested which SPN, from where. Baseline normal ticket activity to cut false positives. Requires Kerberos auditing enabled (see `setup.md` §6).
- **Observed:** _to run_

## 5. Persistence: user added to Domain Admins (T1098.007)
- **Technique:** adding an account to a privileged group.
- **Telemetry:** events **4728 / 4732 / 4756** (member added to a security-enabled global / local / universal group).
- **Detects:** custom Wazuh rule **100100** scoped to privileged groups (Domain Admins, Enterprise Admins, Administrators, Schema Admins).
- **Triage:** who made the change, which account was added, was it authorized. High-value, low-noise alert.
- **Observed:** _to run_

## 6. Defense evasion: security log cleared (T1070.001)
- **Technique:** clearing the Windows Security event log to destroy evidence.
- **Telemetry:** event **1102** (audit log cleared).
- **Detects:** built-in Wazuh 60117, raised to level 12 by custom rule **100120**.
- **Triage:** treat as high severity, since legitimate clears are rare. Correlate with activity just before the gap.
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
