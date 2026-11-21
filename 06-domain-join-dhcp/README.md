# Lab 06 — DHCP Server and Joining a Computer to the Domain

**Goal:** Give the lab network its own Windows DHCP server, then add a new computer (`CLIENT01`) to the `lab.local` domain and prove it works: it gets an IP automatically, finds the domain controller, and receives Group Policy.

**Environment**

| Machine | Role | IP |
|---|---|---|
| DC01 | Windows Server 2022: AD DS, DNS, **DHCP** | 10.0.2.15 (static) |
| CLIENT01 | Windows Server 2022 as a member workstation* | DHCP → 10.0.2.100 |
| LabNet | VirtualBox NAT Network 10.0.2.0/24, **VirtualBox DHCP turned off** | gateway 10.0.2.1 |

\*Microsoft retired the Windows 10 Enterprise evaluation, so the client runs the Server 2022 image I already had. The steps for joining, DHCP, and Group Policy are the same on Windows 10/11.

> All outputs below are copied from the lab consoles.

---

## Part 1 — Move DC01 to a lab network with no other DHCP

A network must have only one DHCP server answering. I created the NAT Network `LabNet` with its DHCP **disabled**, moved DC01 to it, and fixed the default gateway (NAT Network uses `.1`, not `.2`):

```powershell
$if = (Get-NetAdapter | ? Status -eq 'Up').Name
Get-NetRoute -InterfaceAlias $if -DestinationPrefix 0.0.0.0/0 | Remove-NetRoute -Confirm:$false
New-NetRoute -InterfaceAlias $if -DestinationPrefix 0.0.0.0/0 -NextHop 10.0.2.1
Test-NetConnection 8.8.8.8 -Port 53      # TcpTestSucceeded : True
```

## Part 2 — Install DHCP, create the scope, authorize the server

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
netsh dhcp add securitygroups
Add-DhcpServerv4Scope -Name 'LAB-Clients' -StartRange 10.0.2.100 -EndRange 10.0.2.150 `
    -SubnetMask 255.255.255.0 -LeaseDuration 8.00:00:00 -State Active
Set-DhcpServerv4OptionValue -ScopeId 10.0.2.0 -Router 10.0.2.1 -DnsServer 10.0.2.15 -DnsDomain lab.local
Add-DhcpServerInDC -DnsName dc01.lab.local -IPAddress 10.0.2.15
```

### Problem: authorization failed — `PermissionDenied`

```
Add-DhcpServerInDC : Failed to initialize directory service resources for domain lab.local.
    + CategoryInfo : PermissionDenied: (PS_DhcpServerInDC) [Add-DhcpServerInDC], CimException
```

`Add-DhcpServerInDC` returned *"Failed to initialize directory service resources … PermissionDenied"*.

- **Cause:** In a domain, a DHCP server must be **authorized in Active Directory** before it will hand out addresses. The authorization is written to the forest Configuration partition, and only **Enterprise Admins** can write there. My account was a local administrator on the DC but not in Enterprise Admins.
- **Fix:** Added the account to Enterprise Admins (and Domain Admins), then **signed out and back in**. Group membership is stored in the logon token, so a new group does not apply until the next logon.

```powershell
Add-ADGroupMember 'Enterprise Admins' -Members vboxuser
# sign out / sign in
whoami /groups | findstr /i "Enterprise"     # LAB\Enterprise Admins ... Enabled group
Add-DhcpServerInDC -DnsName dc01.lab.local -IPAddress 10.0.2.15   # success
```

```
DnsName        IPAddress
-------        ---------
dc01.lab.local 10.0.2.15

ScopeId  Name        StartRange EndRange   LeaseDuration State
-------  ----        ---------- --------   ------------- -----
10.0.2.0 LAB-Clients 10.0.2.100 10.0.2.150 8.00:00:00    Active

OptionId Name            Value
-------- ----            -----
      15 DNS Domain Name {lab.local}
       3 Router          {10.0.2.1}
       6 DNS Servers     {10.0.2.15}
      51 Lease           {691200}
```

## Part 3 — Prepare the computer account with Offline Domain Join

Instead of typing domain admin credentials on the new machine, I created the computer account on the DC and exported a provisioning file with `djoin`. I also created an OU for workstations so the computer lands in the right place from the start:

```powershell
New-ADOrganizationalUnit -Name Workstations -Path 'DC=lab,DC=local'
djoin /provision /domain lab.local /machine CLIENT01 `
      /machineou "OU=Workstations,DC=lab,DC=local" /savefile C:\Install\client01-odj.txt
```

```
Provisioning the computer...
Successfully provisioned [CLIENT01] in the domain [lab.local].
Provisioning data was saved successfully to [C:\Install\client01-odj.txt].
Computer provisioning completed successfully.

Name     DistinguishedName                         Enabled
----     -----------------                         -------
CLIENT01 CN=CLIENT01,OU=Workstations,DC=lab,DC=local  True
```

> The provisioning file contains the computer account's secret, so it was handled like a password and deleted after use.

## Part 4 — Client gets its IP from DC01

On CLIENT01:

```powershell
ipconfig /release; ipconfig /renew
ipconfig /all | findstr /i "DHCP Lease Suffix"
```

```
InterfaceAlias      : Ethernet
IPv4Address         : {10.0.2.100}

   DNS Suffix Search List. . . . . . : lab.local
   DHCP Enabled. . . . . . . . . . . : Yes
   Lease Obtained. . . . . . . . . . : Sunday, November 22, 2026 2:14:59 AM
   Lease Expires . . . . . . . . . . : Monday, November 30, 2026 2:14:59 AM
   DHCP Server . . . . . . . . . . . : 10.0.2.15
```

The client received `10.0.2.100`, DNS `10.0.2.15`, suffix `lab.local`, and an 8-day lease from **DHCP Server 10.0.2.15**. The packet-level DORA for this exchange is analyzed in [Lab 01 — Part 3](../01-wireshark/#part-3--dhcp-dora-discover-offer-request-acknowledge).

## Part 5 — Join the domain

```powershell
djoin /requestODJ /loadfile C:\Windows\Temp\odj.txt /windowspath C:\Windows /localos
Restart-Computer
```

### Problem: `0x57 — The parameter is incorrect`

```
Loading provisioning data from the following file: [C:\Windows\Temp\odj.txt].
The provisioning data is not valid: 0x57.
The offline domain join request failed: 0x57.
The parameter is incorrect.
```

- **Check:** The file on the client was **5,506 bytes**; the original on DC01 was **5,508 bytes**, and the SHA-256 hashes did not match.
- **Cause:** `djoin` writes the file as UTF-16 with a 2-byte **BOM** (`FF FE`) at the start. My copy was missing it.
- **Fix:** Added the BOM back, confirmed the hash matched the original, and ran the join again.

```
CA43C2CA3337DEA0E8630C997F04F32C1BECB8F994932837CE99F9704EE0E2E4   <- matches the original
Loading provisioning data from the following file: [C:\Windows\Temp\odj2.txt].
The provisioning request completed successfully.
A reboot is required for changes to be applied.
```

## Part 6 — Verify

**From the domain controller:** the lease is active and the computer account is in `OU=Workstations`.

```
Packets total: 38    Packet drop count: 0

IPAddress  ClientId          HostName           AddressState LeaseExpiryTime
---------  --------          --------           ------------ ---------------
10.0.2.100 08-00-27-78-b4-dc CLIENT01.lab.local Active       11/30/2026 2:19 AM

Name     DNSHostName        OperatingSystem
----     -----------        ---------------
CLIENT01 CLIENT01.lab.local Windows Server 2022 Standard Evaluation
```

**From the client after reboot:**

```powershell
(Get-CimInstance Win32_ComputerSystem).Domain   # lab.local
Test-ComputerSecureChannel                      # True
nltest /dsgetdc:lab.local                       # DC: \\DC01.lab.local
gpupdate /force; gpresult /r /scope computer    # Default Domain Policy applied
```

```
Domain: lab.local
Secure channel OK: True

           DC: \\DC01.lab.local
      Address: \\10.0.2.15
     Dom Name: lab.local
        Flags: PDC GC DS LDAP KDC TIMESERV GTIMESERV WRITABLE DNS_DC DNS_DOMAIN DNS_FOREST ...

    Applied Group Policy Objects
    -----------------------------
        Default Domain Policy
```

---

## Troubleshooting Summary

| Symptom | Root cause | Fix |
|---|---|---|
| `Add-DhcpServerInDC` → PermissionDenied | Account not in Enterprise Admins | Add to group, then sign out/in for a new token |
| `djoin` → `0x57` | Provisioning file missing the UTF-16 BOM (2 bytes) | Compared size + SHA-256, restored the BOM |
| Client could not read the VirtualBox shared folder | Guest Additions needed a reboot | Moved the file another way, then rebooted |
| DC kept locking even after `powercfg` | VM reported **battery** power; I had only changed AC settings | Set both `-ac` and `-dc` timeouts and `CONSOLELOCK` |

## What I Learned
- Only one DHCP server should answer on a network. In AD, a Windows DHCP server will not lease addresses until it is **authorized**.
- New group membership needs a **new logon** to take effect. This is the same reason users are told to sign out after being added to a group.
- **Offline Domain Join** creates the computer account ahead of time, so nobody types domain admin credentials on the new machine. Useful for imaging and remote sites.
- Comparing **file size and hash** quickly shows whether a file was damaged in transit.
- How to confirm a machine is healthy in the domain: secure channel, DC locator (`nltest`), and `gpresult`.

## Commands Used
```
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerv4Scope / Set-DhcpServerv4OptionValue / Add-DhcpServerInDC
Get-DhcpServerv4Lease -ScopeId 10.0.2.0
djoin /provision ... /savefile    |    djoin /requestODJ ... /localos
ipconfig /release  /renew  /all
Test-ComputerSecureChannel
nltest /dsgetdc:lab.local
gpupdate /force   |   gpresult /r
Get-FileHash
```
