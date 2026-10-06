# Setup

## 1. Network
- VirtualBox **internal network** `soc-lab`, subnet `192.168.10.0/24`, no NAT adapter on lab VMs (DC-01 shows "no internet access").
- Static IPs: DC-01 `.10`, WIN-01 `.20`, LNX-01 `.30`, IDS-01 `.40`. KALI-01 gets its address from the DHCP role on DC-01.
- DNS for all domain members: `192.168.10.10`.
- IDS-01 has a second adapter on `soc-lab` in **promiscuous mode (Allow All)**, with no IP, used as the Suricata capture interface.

## 2. Domain controller (DC-01)
```powershell
Rename-Computer -NewName DC-01 -Restart
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "soc.local" -DomainNetbiosName "SOC" -InstallDns
```
Join WIN-01 to the domain:
```powershell
Add-Computer -DomainName soc.local -Credential SOC\Administrator -Restart
```

## 3. Wazuh (SOC-01)
The install script downloads packages, so SOC-01 gets a **temporary NAT adapter** for the install, removed afterwards (or use the Wazuh offline installation).

All-in-one install (manager + indexer + dashboard):
```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```
Dashboard: `https://<SOC-01 IP>`. The admin password is printed at the end of the install.

## 4. Agent enrollment
The lab VMs have no internet, so the agent packages are downloaded on the host and copied to each VM.

Windows (PowerShell as admin, MSI from `https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.2-1.msi`):
```powershell
msiexec.exe /i .\wazuh-agent-4.14.2-1.msi /q WAZUH_MANAGER="<SOC-01 IP>"
NET START WazuhSvc
```
Linux (Ubuntu):
```bash
sudo WAZUH_MANAGER="<SOC-01 IP>" apt-get install ./wazuh-agent_4.14.2-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

## 5. Suricata (IDS-01)
On Ubuntu 24.04 (Suricata 7.0.3) the rule directory is `/var/lib/suricata/rules`.
```bash
sudo apt install suricata
sudo sed -i 's/^\s*HOME_NET:.*/    HOME_NET: "[192.168.10.0\/24]"/' /etc/suricata/suricata.yaml
sudo cp suricata/local.rules /var/lib/suricata/rules/local.rules
# in suricata.yaml: add "- local.rules" under rule-files,
# and set the af-packet interface to the promiscuous capture adapter
sudo suricata -T -c /etc/suricata/suricata.yaml   # test config
sudo systemctl restart suricata
```
Forward alerts to Wazuh: add [`wazuh/ossec-agent-ids.conf`](../wazuh/ossec-agent-ids.conf) to `/var/ossec/etc/ossec.conf` on IDS-01, then `sudo systemctl restart wazuh-agent`.

## 6. Windows audit policy
Needed for the AD scenarios, in `Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy Configuration`.

**Default Domain Controllers Policy** (DC-01):
- Account Logon → Audit Kerberos Service Ticket Operations: **Success, Failure** (4769)
- Account Logon → Audit Kerberos Authentication Service: **Success, Failure** (4768, 4771)
- Account Logon → Audit Credential Validation: **Success, Failure** (4776)
- Account Management → Audit Security Group Management: **Success** (4728, 4732, 4756)
- Logon/Logoff → Audit Logon: **Success, Failure** (4624, 4625)

**GPO linked to the Workstations OU** (WIN-01):
- Logon/Logoff → Audit Logon: **Success, Failure** (4625 for RDP/SMB spraying against WIN-01)

```powershell
gpupdate /force
auditpol /get /category:*
```

## 7. Custom Wazuh rules
```bash
sudo cp wazuh/local_rules.xml /var/ossec/etc/rules/local_rules.xml
sudo /var/ossec/bin/wazuh-logtest     # paste a sample event to validate
sudo systemctl restart wazuh-manager
```

## Troubleshooting
| Symptom | Check |
| --- | --- |
| Agent shows *disconnected* | `systemctl status wazuh-agent`, `/var/ossec/logs/ossec.log`, ports 1514/1515 reachable from the agent |
| No Windows Kerberos events | Audit policy applied? `auditpol /get /subcategory:"Kerberos Service Ticket Operations"` |
| No Suricata alerts in Wazuh | `tail -f /var/log/suricata/eve.json` on IDS-01, `localfile` block present in agent config |
| Custom rule never fires | Run the event through `wazuh-logtest` and check which built-in rule catches it first |
