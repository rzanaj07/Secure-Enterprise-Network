# 6. Wazuh agents and testing

> **Status:** ⏳ Planned. Screenshots will be added when this phase is complete.

## 6.1 Install agents (DC01 and Win 10)

On each Windows host, run PowerShell as Administrator:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi -OutFile $env:TEMP\wazuh-agent.msi
msiexec.exe /i $env:TEMP\wazuh-agent.msi /q WAZUH_MANAGER="192.168.30.10" WAZUH_AGENT_GROUP="windows"
NET START WazuhSvc
```

Replace `4.x.x` with the version that matches your manager. You can also copy the exact command from Dashboard → **Agents → Deploy new agent**.


## 6.2 Test scenarios

| # | Test | How | Expected result |
|---|------|-----|-----------------|
| 1 | Brute-force logon | Log in to Win 10 with a wrong password 5+ times | Event 4625 alerts, then account lockout (4740) |
| 2 | Privileged group change | Add a user to `Domain Admins` on DC01 | Event 4728 alert, high severity |
| 3 | Segmentation | From Win 10, `Test-NetConnection 192.168.30.10 -Port 22` | Blocked; `FW-DROP` appears in MikroTik log and in Wazuh |
| 4 | Router login failure | Wrong SSH password to the MikroTik | Syslog alert in Wazuh |
| 5 | File integrity | Change a file in a monitored path | FIM alert |

## 6.3 Results

Results and evidence will be documented here after testing.
