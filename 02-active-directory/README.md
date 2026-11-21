# Lab 02 — Active Directory Domain Controller and Account Lockout

**Goal:** Build a Windows Server 2022 domain controller from scratch, set up a realistic company structure, handle common help desk tasks, then simulate an account lockout and investigate it in the Security log.

**Environment:** VirtualBox on Windows 11 host · Windows Server 2022 Standard Evaluation (Desktop Experience) · VM `DC01`, 4 GB RAM, 2 vCPU, NAT network

---

## Part 1 — Server Preparation

| Setting | Value | Why |
|---|---|---|
| IPv4 address | `10.0.2.15/24` (static) | A domain controller must not change its IP |
| Default gateway | `10.0.2.2` | VirtualBox NAT gateway |
| DNS server | `127.0.0.1`, `8.8.8.8` | The DC is its own DNS server once AD DS is installed |

```powershell
New-NetIPAddress -InterfaceAlias Ethernet -IPAddress 10.0.2.15 -PrefixLength 24 -DefaultGateway 10.0.2.2
Set-DnsClientServerAddress -InterfaceAlias Ethernet -ServerAddresses 127.0.0.1,8.8.8.8
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

Result: `Success: True — Active Directory Domain Services, Group Policy Management`

## Part 2 — Promote to Domain Controller

```powershell
Install-ADDSForest -DomainName lab.local -DomainNetbiosName LAB -InstallDns `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString)
```

- New forest and domain: **lab.local** (NetBIOS `LAB`), with integrated DNS.
- **Troubleshooting:** the first attempt failed because the DSRM password did not meet the complexity policy. A stronger password fixed it.

## Part 3 — Organizational Structure

| OU | Users | Security group |
|---|---|---|
| IT | Ahmed Ali, Omar Khalid | GRP-IT |
| HR | Noura Saleh, Faisal Nasser, Reem Fahad, Sara Hassan | GRP-HR |
| Sales | Khalid Saad, Huda Majed, Yousef Adel, Lama Turki | GRP-Sales |

All 10 accounts were created **disabled with no password**. Enabling an account and issuing a temporary password is a help desk task, done in Part 5.

## Part 4 — Group Policy

**Default Domain Password and Lockout Policy**

| Setting | Value |
|---|---|
| Minimum password length | 10 |
| Complexity required | Yes |
| Account lockout threshold | 5 invalid attempts |
| Lockout duration | 15 minutes |

**GPO "Sales - Block Control Panel"**, linked to `OU=Sales`, sets `User Configuration > Administrative Templates > Control Panel > Prohibit access to Control Panel` (registry `NoControlPanel = 1`).

## Part 5 — Help Desk Tasks (ADUC)

1. **Reset password** for Ahmed Ali, with *User must change password at next logon* checked.
2. **Enable account:** confirmed with *"Object Ahmed Ali has been enabled."*
3. **Move user:** Sara Hassan moved from IT to HR.
4. **Fix group membership:** moving an object between OUs does **not** change its groups, so Sara still had IT access through `GRP-IT`. I removed her from `GRP-IT` and added her to `GRP-HR`:

```text
Get-ADPrincipalGroupMembership sara.hassan
Name
----
Domain Users
GRP-HR
```

## Part 6 — Account Lockout Investigation

**Simulation:** 6 binds with wrong passwords against `ahmed.ali`.

```text
Name                   : Ahmed Ali
LockedOut              : True
BadLogonCount          : 5
LastBadPasswordAttempt : 11/21/2026 3:19:04 PM
```

**Detection: Security log, Event ID 4740**

```text
TimeCreated    : 11/21/2026 3:19:04 PM
Id             : 4740
Summary        : A user account was locked out.
Account        : ahmed.ali
CallerComputer : DC01
```

The same query for failed logons (**4625**) also turned up four real failed sign-ins for `vboxuser` earlier that day. Those were my own login attempts while I troubleshot a keyboard issue in the VM. In a SOC, a pattern like that is exactly what you check against the user before escalating.

**Resolution**

```powershell
Unlock-ADAccount -Identity ahmed.ali
```
```text
Name          : Ahmed Ali
Enabled       : True
LockedOut     : False
BadLogonCount : 0
```

---

## Key Event IDs

| ID | Meaning |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4740 | User account locked out |
| 4767 | User account unlocked |
| 4771 | Kerberos pre-authentication failed |
| 4776 | NTLM credential validation |

## What I Learned
- A DC needs a static IP and must point DNS at itself, or domain joins and logons fail.
- Moving a user to a new OU changes which GPOs apply, but **not** their group memberships or access.
- Lockout policy is a security control and a common help desk ticket. Event **4740** shows *who* was locked out and *from which machine*, which is the first thing to check before unlocking.
- VirtualBox tips: the host key (Right Ctrl) releases keyboard capture, and *Input > Keyboard > Insert Ctrl-Alt-Del* sends the key combo to the guest.
