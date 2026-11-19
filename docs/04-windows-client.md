# 4. Joining the Windows 10 client to the domain

> **Status:** ⏳ Planned. Screenshots will be added when this phase is complete.

The **Win 10** workstation represents an employee computer in the **Office** segment (`ether2-Office`, 192.168.10.0/24). It gets its network settings from the MikroTik DHCP server and authenticates against DC01.

## Prerequisites

| Requirement | Why |
|-------------|-----|
| VirtualBox Adapter 1 set to **Internal Network `LAN-Zyra`** | Connects the VM to `ether2-Office` on the router |
| MikroTik DHCP for 192.168.10.0/24 hands out **DNS 192.168.20.10** | Clients find the domain controller through DNS SRV records hosted on DC01 |
| DC01 reachable from the Office segment | Required for Kerberos, LDAP, and SMB during the join |

## 4.1 Verify network configuration

```cmd
ipconfig /all
```

Expected results:

| Field | Expected value |
|-------|----------------|
| IPv4 address | 192.168.10.x (from the MikroTik pool) |
| Default gateway | 192.168.10.1 |
| DNS server | 192.168.20.10 |

## 4.2 Verify connectivity to the domain controller

```cmd
ping 192.168.20.10
nslookup corp.local
nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.local
```

## 4.3 Join the domain

**GUI:** *Settings → System → About → Rename this PC (advanced) → Change → Domain: `corp.local`*

**PowerShell (as Administrator):**

```powershell
Add-Computer -DomainName corp.local -Credential CORP\Administrator -Restart
```

## 4.4 Organize the computer object

After the reboot, move the computer object from the default `Computers` container to `Departments\Office`, so that department GPOs apply to it:

```powershell
Get-ADComputer WIN10 | Move-ADObject -TargetPath "OU=Office,OU=Departments,DC=corp,DC=local"
```

## 4.5 Verification

```cmd
whoami
systeminfo | findstr /B /C:"Domain"
gpresult /r
```
