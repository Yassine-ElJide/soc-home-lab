# Setup

## 1. Network
- VirtualBox **internal network** `soc-lab`, subnet `192.168.10.0/24`, no NAT adapter on lab VMs (DC-01 shows *"Réseau non identifié – pas d'accès Internet"*).
- Static IPs: DC-01 `.10`, WIN-01 `.20`, LNX-01 `.30`, IDS-01 `.40`.
- DNS for all domain members: `192.168.10.10`.
- IDS-01 has a second adapter in **promiscuous mode (Allow All)** to see lab traffic.

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
All-in-one install (manager + indexer + dashboard):
```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```
Dashboard: `https://<SOC-01>` – admin password printed at the end of the install.

## 4. Agent enrollment
Windows (PowerShell as admin):
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.2-1.msi -OutFile $env:TEMP\wazuh-agent.msi
msiexec.exe /i $env:TEMP\wazuh-agent.msi /q WAZUH_MANAGER="<SOC-01 IP>"
NET START WazuhSvc
```
Linux (Ubuntu):
```bash
sudo WAZUH_MANAGER="<SOC-01 IP>" apt-get install ./wazuh-agent_4.14.2-1_amd64.deb
sudo systemctl enable --now wazuh-agent
```
> The lab VMs have no internet: packages are downloaded on the host and copied over.

## 5. Suricata (IDS-01)
```bash
sudo apt install suricata
sudo sed -i 's/^\s*HOME_NET:.*/    HOME_NET: "[192.168.10.0\/24]"/' /etc/suricata/suricata.yaml
sudo cp suricata/local.rules /etc/suricata/rules/local.rules
# add "- local.rules" under rule-files in suricata.yaml, set af-packet interface
sudo suricata -T -c /etc/suricata/suricata.yaml   # test config
sudo systemctl restart suricata
```
Forward alerts to Wazuh: add [`wazuh/ossec-agent-ids.conf`](../wazuh/ossec-agent-ids.conf) to `/var/ossec/etc/ossec.conf` on IDS-01, then `sudo systemctl restart wazuh-agent`.

## 6. Windows audit policy (Default Domain Controllers Policy)
Needed for the AD scenarios:
`Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy Configuration`
- Account Logon → Audit Kerberos Service Ticket Operations: **Success, Failure**
- Account Management → Audit Security Group Management: **Success**
- Logon/Logoff → Audit Logon: **Success, Failure**

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
