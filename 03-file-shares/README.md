# Lab 03 — Department File Shares, NTFS Permissions, Drive Mapping and Access Auditing

**Goal:** Give each department a private network share, map it automatically at logon, and audit who reads, writes, or deletes files in the sensitive HR share.

**Environment:** Domain controller `DC01`, domain `lab.local` (built in [Lab 02](../02-active-directory/))

---

## Part 1 — Shares and NTFS Permissions

| Share | Path | NTFS permissions |
|---|---|---|
| `\\DC01\IT` | `C:\Shares\IT` | GRP-IT: Modify · Administrators, SYSTEM: Full |
| `\\DC01\HR` | `C:\Shares\HR` | GRP-HR: Modify · Administrators, SYSTEM: Full |
| `\\DC01\Sales` | `C:\Shares\Sales` | GRP-Sales: Modify · Administrators, SYSTEM: Full |

- **Inheritance disabled** (`icacls /inheritance:r`), so the default *Users: Read* entry is removed and only the department group gets in.
- **Share permissions:** Authenticated Users = Change, Domain Admins = Full. Access is controlled at the **NTFS** layer, which is Microsoft's recommended pattern.

```powershell
icacls C:\Shares\HR /inheritance:r /grant:r "BUILTIN\Administrators:(OI)(CI)F" `
  "NT AUTHORITY\SYSTEM:(OI)(CI)F" "LAB\GRP-HR:(OI)(CI)M"
New-SmbShare -Name HR -Path C:\Shares\HR -FullAccess "LAB\Domain Admins" `
  -ChangeAccess "NT AUTHORITY\Authenticated Users"
```

**Access matrix (verified with `Get-Acl`)**

```text
Share Group     Access
----- -----     ------
IT    GRP-IT    Modify, Synchronize
IT    GRP-HR    no access
IT    GRP-Sales no access
HR    GRP-IT    no access
HR    GRP-HR    Modify, Synchronize
HR    GRP-Sales no access
Sales GRP-IT    no access
Sales GRP-HR    no access
Sales GRP-Sales Modify, Synchronize
```

## Part 2 — Automatic Drive Mapping at Logon

One logon script per department in `NETLOGON` (`C:\Windows\SYSVOL\sysvol\lab.local\scripts`), assigned through each user's **Logon script** attribute (ADUC > Profile tab):

```bat
@echo off
net use H: \\DC01\HR /persistent:no
```

| Department | Drive | Script |
|---|---|---|
| IT | `I:` | map-IT.bat |
| HR | `H:` | map-HR.bat |
| Sales | `S:` | map-Sales.bat |

**Issue found and fixed:** after Sara Hassan was moved to HR in Lab 02, her **Department** attribute still said *IT*. Her logon script was correct because it was assigned by OU, but the stale attribute would mislead anyone filtering users by department. I corrected it with `Set-ADUser sara.hassan -Department HR`.

## Part 3 — Auditing the HR Share (SOC)

1. Enabled the **File System** audit subcategory: `auditpol /set /subcategory:"File System" /success:enable /failure:enable`.
2. Added a **SACL** to `C:\Shares\HR`: Everyone · ReadData, WriteData, Delete · Success and Failure.
3. Generated activity: read `Welcome-HR.txt`, then created and deleted `Salaries-2026.txt`.
4. Queried **Event ID 4663** (*An attempt was made to access an object*):

```text
TimeCreated            Id   User      Object                           Access
-----------            --   ----      ------                           ------
11/21/2026 4:06:57 PM  4663 vboxuser  C:\Shares\HR\Salaries-2026.txt   DELETE
11/21/2026 4:06:57 PM  4663 vboxuser  C:\Shares\HR\Salaries-2026.txt   WriteData (or AddFile)
11/21/2026 4:06:57 PM  4663 vboxuser  C:\Shares\HR\Salaries-2026.txt   ReadData (or ListDirectory)
11/21/2026 4:06:57 PM  4663 vboxuser  C:\Shares\HR\Welcome-HR.txt      ReadData (or ListDirectory)
```

The log shows **who** touched **which file**, **what** they did, and **when**. That is the evidence trail an analyst needs for a data-leak or insider-threat investigation.

---

## Key Event IDs

| ID | Meaning |
|---|---|
| 4663 | An attempt was made to access an object (file read/write/delete) |
| 4656 | A handle to an object was requested |
| 4660 | An object was deleted |
| 5140 | A network share object was accessed |

## What I Learned
- **Share permissions vs NTFS:** the more restrictive of the two wins. Keep the share layer simple and do the real control in NTFS.
- Granting access to **groups**, never to individual users, means onboarding is just "add to group."
- Moving users between OUs can leave **stale attributes** (Department) and **stale group memberships** (Lab 02). Always check both.
- File auditing needs **two** things: the audit policy turned on *and* a SACL on the folder. Either one alone logs nothing.
