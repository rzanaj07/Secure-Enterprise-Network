# 3. Active Directory on DC01

**DC01** is the domain controller for the `corp.local` domain. It provides centralized identity (users, computers, groups), authentication (Kerberos), and DNS for the whole network. It sits in the **Servers segment** (`ether3-Servers`, 192.168.20.0/24) behind the MikroTik router.

| Setting | Value |
|---------|-------|
| Hostname | DC01 |
| OS | Windows Server (build 10.0.26100) |
| Domain (FQDN) | `corp.local` |
| NetBIOS name | `CORP` |
| IP address | 192.168.20.10/24 (static) |
| Default gateway | 192.168.20.1 (MikroTik `ether3-Servers`) |
| DNS | 127.0.0.1 / ::1 (itself) |
| Roles | AD DS, DNS, File and Storage Services |

---

## 3.1 Static IP configuration

A domain controller must have a **static IP**. Clients find it through DNS, so its address can't change. DC01 uses the MikroTik as its gateway and **itself** (`127.0.0.1` / `::1`) as its DNS server, because it hosts the `corp.local` DNS zone.

Results of `ipconfig /all`:
- **Host Name:** DC01, **Primary DNS Suffix:** corp.local
- **DHCP Enabled:** No (static)
- **IPv4:** 192.168.20.10, mask 255.255.255.0, gateway 192.168.20.1
- **DNS Servers:** ::1 and 127.0.0.1

![Static IP – ipconfig /all](../screenshots/active-directory/01-static-ip-ipconfig.png)

<details>
<summary>PowerShell equivalent</summary>

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.20.10 -PrefixLength 24 -DefaultGateway 192.168.20.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
Rename-Computer -NewName DC01 -Restart
```
</details>

---

## 3.2 Installing the AD DS and DNS roles

The **Active Directory Domain Services (AD DS)** and **DNS Server** roles were installed through *Server Manager → Manage → Add Roles and Features*. The server was then promoted to a domain controller by creating a **new forest** named `corp.local`.

The Server Manager dashboard shows the three roles installed: **AD DS**, **DNS**, and **File and Storage Services**. All of them report healthy manageability (green).

![Server Manager – roles](../screenshots/active-directory/02-server-manager-roles.png)

<details>
<summary>PowerShell equivalent</summary>

```powershell
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
Install-ADDSForest -DomainName "corp.local" -DomainNetbiosName "CORP" -InstallDns
```
</details>

---

## 3.3 Logging in as the domain administrator

After promotion, the server reboots. The local Administrator account becomes the **domain administrator**, `CORP\Administrator`, and the logon screen now shows the domain name. This confirms the forest was created.

![Domain admin logon](../screenshots/active-directory/03-domain-admin-login.png)

---

## 3.4 Organizational Unit (OU) structure

A **Departments** OU was created with one sub-OU per department. OUs let you link **Group Policy** to specific departments and **delegate administration**. For example, the IT team can manage only the IT OU.

```
corp.local
└── Departments
    ├── Finance
    ├── IT
    └── Office
```

**Tool:** *Server Manager → Tools → Active Directory Users and Computers*

![OU structure](../screenshots/active-directory/04-ou-structure.png)

<details>
<summary>PowerShell equivalent</summary>

```powershell
New-ADOrganizationalUnit -Name "Departments" -Path "DC=corp,DC=local"
"Finance","IT","Office" | ForEach-Object {
    New-ADOrganizationalUnit -Name $_ -Path "OU=Departments,DC=corp,DC=local"
}
```
</details>

---

## 3.5 Domain Controllers OU

The **Domain Controllers** container lists **DC01**. Its DC Type is **GC (Global Catalog)** and its site is `Default-First-Site-Name`. The Global Catalog holds a searchable copy of every object in the forest. Clients need it for logon and for searching AD.

![Domain Controllers](../screenshots/active-directory/05-domain-controllers.png)

---

## 3.6 DNS forwarders

DC01 is authoritative only for `corp.local`. Any name it can't resolve itself (for example, `google.com`) is forwarded to **Google DNS (8.8.8.8 and 8.8.4.4)**, with root hints as a fallback.

With this setup, every domain client uses **DC01 as its only DNS server**. Clients can resolve both internal AD records and internet names, and every DNS query goes through the domain controller, where it can be logged.

**Tool:** *DNS Manager → right-click DC01 → Properties → Forwarders*

![DNS forwarders](../screenshots/active-directory/06-dns-forwarders.png)

<details>
<summary>PowerShell equivalent</summary>

```powershell
Add-DnsServerForwarder -IPAddress 8.8.8.8, 8.8.4.4
```
</details>

---

## 3.7 Verification

| Command | Result | Meaning |
|---------|--------|---------|
| `whoami` | `corp\administrator` | Logged in with a domain account |
| `nslookup corp.local` | `192.168.20.10` | The domain name resolves to the DC through its own DNS (`::1`) |

![Verification – whoami & nslookup](../screenshots/active-directory/07-verify-whoami-nslookup.png)

Other useful checks:

```powershell
dcdiag /q                      # domain controller health, no output = healthy
Get-ADDomain | Select Name, Forest, DomainMode
nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.local   # SRV record that clients use to find the DC
```

---

## Next steps

- Create users and security groups in each department OU.
- Group Policy: password policy (12+ characters), account lockout (5 attempts), and **advanced audit policy** (logon, account management, process creation), so Wazuh receives useful events.
- Join Win 10 to the domain → [04-windows-client.md](04-windows-client.md)
