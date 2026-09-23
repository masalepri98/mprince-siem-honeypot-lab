# 03 — Windows endpoint, Sysmon, and FIM

**Outcome:** the Windows host appears as a Wazuh agent and sends process and file-change telemetry. Use a disposable Windows VM instead if you plan to test intrusive activity.

## 1. Enroll Windows

In Wazuh Dashboard → **Agents management → Summary → Deploy new agent**, select Windows, enter the Wazuh VM's stable local IP, and copy the generated command. Run it in an elevated PowerShell session. This uses the installer version matching your Wazuh manager. The equivalent [official installation steps](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html) are maintained by Wazuh.

Start and check the service:

```powershell
Start-Service WazuhSvc
Get-Service WazuhSvc
Test-NetConnection <WAZUH_LAN_IP> -Port 1514
```

Wait for the host to show **Active** in Agents management. If enrollment works but status stays disconnected, check TCP 1514, the Windows agent log at `C:\Program Files (x86)\ossec-agent\ossec.log`, and the Wazuh manager log at `/var/ossec/logs/ossec.log`.

## 2. Install Sysmon from Microsoft

Download [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) from Microsoft Sysinternals and choose a reviewed configuration, such as the [SwiftOnSecurity Sysmon configuration](https://github.com/SwiftOnSecurity/sysmon-config). Save your selected XML in the Sysmon download folder as `sysmonconfig.xml`, inspect its rules, and record its commit/version. In elevated PowerShell from that folder:

```powershell
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
Get-WinEvent -ListLog 'Microsoft-Windows-Sysmon/Operational'
```

Edit `C:\Program Files (x86)\ossec-agent\ossec.conf` as Administrator. **Add this block inside the existing `<ossec_config>` element; do not replace the file:**

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Restart the agent with `Restart-Service WazuhSvc`. Run a harmless program such as Notepad and locate the corresponding Sysmon Event ID 1 in Event Viewer, then in Wazuh Threat Hunting. Record the process name and UTC timestamp, with usernames and paths redacted in public screenshots. The [Wazuh event channel guide](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/configuration.html) documents the channel name.

## 3. Add a small FIM test directory

Create `C:\LabEvidence\FIM-Test` locally. Inside the existing `<syscheck>` section of the Windows agent's `ossec.conf`, add:

```xml
<directories realtime="yes">C:\LabEvidence\FIM-Test</directories>
```

Restart the agent. Create, edit, and delete `C:\LabEvidence\FIM-Test\probe.txt` and find the file integrity alerts in Wazuh. Monitor only this dedicated directory first; broad home-directory monitoring can produce noisy events and expose personal filenames. Wazuh requires the directory to exist before restart for real-time monitoring.

```powershell
New-Item -ItemType Directory -Force 'C:\LabEvidence\FIM-Test' | Out-Null
Set-Content 'C:\LabEvidence\FIM-Test\probe.txt' 'first'
Add-Content 'C:\LabEvidence\FIM-Test\probe.txt' 'second'
Remove-Item 'C:\LabEvidence\FIM-Test\probe.txt'
```

**Checkpoint:** Windows is Active; a benign process event and a file-change event can both be located in Wazuh with timestamps that match Windows Event Viewer and the test command.
