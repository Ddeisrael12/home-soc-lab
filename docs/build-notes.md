# Build Notes

How the lab was built, phase by phase. Commands are the ones actually used.

## Phase 1: Hypervisor, network, and firewall

- **Hypervisor:** VMware Workstation Pro (free for personal use).
- **Lab network:** VMnet2, set to **host-only**, with VMware's DHCP off and no host adapter, so the lab is isolated from the home network.
- **FW01 (pfSense):** two adapters. WAN = VMware NAT (internet), LAN = VMnet2.
  - LAN IP `10.0.0.1/24`; DHCP pool `10.0.0.100` to `10.0.0.200`.

## Phase 2: Domain controller (DC001)

- Windows Server 2025 (evaluation), static IP `10.0.0.10`, gateway `10.0.0.1`.
- Domain: `BuildBetter.digital`, with OU `LabUsers` and test user `jdoe`.

```powershell
New-ADOrganizationalUnit -Name "LabUsers" -Path "DC=BuildBetter,DC=digital"
New-ADUser -Name "Jane Doe" -SamAccountName jdoe -Path "OU=LabUsers,DC=BuildBetter,DC=digital" -AccountPassword (Read-Host -AsSecureString) -Enabled $true
```

- **DNS:** forwarders to `10.0.0.1` and `8.8.8.8` for internet names, plus a reverse lookup zone for `10.0.0.0/24`.

```powershell
Set-DnsServerForwarder -IPAddress 10.0.0.1, 8.8.8.8
Add-DnsServerPrimaryZone -NetworkId "10.0.0.0/24" -ReplicationScope Domain
```

- **Audit policy:** set in the **Default Domain Controllers Policy** → Advanced Audit Policy Configuration, Success and Failure for:
  - Account Logon: Credential Validation, Kerberos Authentication Service
  - Account Management: User Account Management, Security Group Management
  - Logon/Logoff: Logon, Account Lockout
  - Security Option "Audit: Force audit policy subcategory settings" = Enabled
- Also enabled: command line in process creation events, PowerShell Script Block Logging (event 4104).

## Phase 3: Windows 11 endpoint (WS001)

- Windows 11 Enterprise on VMnet2, DHCP address from pfSense, DNS pointed to DC001.
- Joined to the domain, then renamed:

```powershell
Add-Computer -DomainName BuildBetter.digital -Credential BuildBetter\Administrator -Restart
Rename-Computer -NewName WS001 -DomainCredential BuildBetter\Administrator -Restart
```

- **Sysmon** with the SwiftOnSecurity community config:

```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "C:\Tools\sysmonconfig-export.xml"
cd C:\Tools
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

## Phase 4: Wazuh SIEM (SIEM001)

- Ubuntu Server, static IP `10.0.0.20`, 47 GB root volume.
- Wazuh 4.14 all-in-one install (`-i` skips the OS check, since the Ubuntu version is newer than the officially supported list):

```bash
curl -O https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a -i
```

- **Windows agents** on DC001 and WS001:

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.9-1.msi" -OutFile "$env:TEMP\wazuh-agent.msi"
msiexec.exe /i "$env:TEMP\wazuh-agent.msi" /q WAZUH_MANAGER="10.0.0.20" WAZUH_AGENT_NAME="WS001"
NET START WazuhSvc
```

- **Extra log channels** on WS001, added to `ossec.conf`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
<localfile>
  <location>Microsoft-Windows-PowerShell/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

## Phase 5: Triage practice

Detections are written up in [tickets/](../tickets/). Useful Wazuh (DQL) queries:

```
agent.name:DC001 and data.win.system.eventID:(4720 or 4728)
agent.name:DC001 and data.win.system.eventID:(4625 or 4740 or 4771)
agent.name:WS001 and rule.level >= 10
```

## Next steps

- Wazuh agent on Azure-Entra01
- pfSense syslog to Wazuh
- File Integrity Monitoring and Vulnerability Detection exercises
