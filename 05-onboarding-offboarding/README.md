# Lab 05 — User Onboarding and Offboarding

**Goal:** Handle the two most common account tickets in IT support: bring 5 new hires in from an HR list, and safely remove a leaver without losing their data or breaking their team.

**Environment:** Domain controller `DC01` (`lab.local`), Windows Server 2022, PowerShell (ActiveDirectory module)

---

## Part 1 — Request from HR

HR sends a CSV of new starters (`C:\HR-Requests\new-hires.csv`):

```text
FirstName LastName  Department Title                Manager     StartDate
--------- --------  ---------- -----                -------     ---------
Rakan     Alharbi   IT         Help Desk Technician ahmed.ali   2026-11-23
Bader     Alghamdi  IT         System Administrator ahmed.ali   2026-11-23
Maha      Alotaibi  HR         HR Specialist        noura.saleh 2026-11-23
Turki     Alqahtani Sales      Sales Representative khalid.saad 2026-11-23
Dana      Alshehri  Sales      Account Manager      khalid.saad 2026-11-23
```

## Part 2 — Bulk Onboarding

For **each** new hire, the script:

| Step | Setting |
|---|---|
| Account | `firstname.lastname` in the department OU, UPN `@lab.local` |
| Profile | Title, Department, **Manager** |
| Status | **Disabled** until the start date, with the description *"New hire - starts 2026-11-23 - pending activation"* |
| Access | Added to the department group (`GRP-IT`, `GRP-HR`, `GRP-Sales`) |
| Drives | Department logon script (from Lab 03) plus a personal **home folder** `U:` → `\\DC01\Home$\<user>` |
| Home folder NTFS | Only the user (Modify) plus Administrators and SYSTEM |

```powershell
foreach ($h in Import-Csv C:\HR-Requests\new-hires.csv) {
  $sam = ($h.FirstName + '.' + $h.LastName).ToLower()
  New-ADUser -Name "$($h.FirstName) $($h.LastName)" -SamAccountName $sam `
    -UserPrincipalName "$sam@lab.local" -Title $h.Title -Department $h.Department `
    -Manager $h.Manager -Path "OU=$($h.Department),DC=lab,DC=local" `
    -ScriptPath "map-$($h.Department).bat" -HomeDirectory "\\DC01\Home$\$sam" -HomeDrive U: `
    -Description "New hire - starts $($h.StartDate) - pending activation" -Enabled $false
  Add-ADGroupMember "GRP-$($h.Department)" -Members $sam
  New-Item "C:\Home\$sam" -ItemType Directory | Out-Null
  icacls "C:\Home\$sam" /grant:r "LAB\${sam}:(OI)(CI)M"
}
```

**Result**

```text
SamAccountName  Title                Department Manager      Group     Enabled
--------------  -----                ---------- -------      -----     -------
rakan.alharbi   Help Desk Technician IT         Ahmed Ali    GRP-IT    False
bader.alghamdi  System Administrator IT         Ahmed Ali    GRP-IT    False
maha.alotaibi   HR Specialist        HR         Noura Saleh  GRP-HR    False
turki.alqahtani Sales Representative Sales      Khalid Saad  GRP-Sales False
dana.alshehri   Account Manager      Sales      Khalid Saad  GRP-Sales False
```

```text
C:\Home\rakan.alharbi  LAB\rakan.alharbi:(OI)(CI)(M)
                       NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
                       BUILTIN\Administrators:(I)(OI)(CI)(F)
```

**On day one:** reset the password with *User must change password at next logon*, then enable the account (as in [Lab 02](../02-active-directory/)).

**Gotcha I hit:** my first draft stored the home path in a variable named `$home`. That is a **read-only automatic variable** in PowerShell, so the script would have failed. I renamed it before running.

## Part 3 — Offboarding a Leaver (Ticket HD-1042)

**Leaver:** Ahmed Ali (IT), who is also the **manager** of two of the new hires.

```text
=== BEFORE ===
Name              : Ahmed Ali
Enabled           : True
DistinguishedName : CN=Ahmed Ali,OU=IT,DC=lab,DC=local
Groups            : GRP-IT
DirectReports     : Bader Alghamdi, Rakan Alharbi
```

**Checklist**

1. **Record** current groups and the ticket number in `C:\Archive\offboard-ahmed.ali.txt` (so access can be restored if HR reverses the decision).
2. **Disable** the account. Do not delete it on day one.
3. **Remove** all group memberships.
4. **Reassign direct reports** to a new manager, so their approvals and org chart don't break.
5. **Update the description** with the date, reason and ticket, and clear the logon script.
6. **Move** the account to `OU=Disabled Users`.
7. **Archive** the home folder to `C:\Archive\` for the retention period.

```text
=== AFTER ===
Name              : Ahmed Ali
Enabled           : False
DistinguishedName : CN=Ahmed Ali,OU=Disabled Users,DC=lab,DC=local
Description       : Offboarded 2026-11-21 - resigned - ticket HD-1042
ScriptPath        :
Groups            :

SamAccountName New manager
rakan.alharbi  vboxuser
bader.alghamdi vboxuser

C:\Archive\offboard-ahmed.ali.txt
C:\Archive\ahmed.ali\handover-notes.txt
```

---

## What I Learned
- **Disable, don't delete.** Deleting an account loses its SID, group history and the evidence trail. Delete only after the retention period.
- **Offboarding affects other people:** a manager leaving breaks the `Manager` field for their team, so always check *DirectReports*.
- **Write it down before you remove it:** saving the group list first makes a rehire or a reversal a 2-minute job.
- **New hires stay disabled until day one.** An enabled, unused account with no owner yet is a security risk.
- Scripting from an HR CSV turns 5 identical manual tasks into one repeatable run with no typos.
