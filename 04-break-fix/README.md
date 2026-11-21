# Lab 04 — Help Desk Break & Fix

**Goal:** Break four common things on purpose, then work each one like a real ticket: **symptom → diagnosis → fix → verification**.

**Environment:** Domain controller `DC01` (`lab.local`), Windows Server 2022, PowerShell

| # | Ticket | Root cause | Fix |
|---|---|---|---|
| 1 | "Users can't find the domain" | DNS Server service stopped | Start the DNS service |
| 2 | "I can't print" | Print Spooler stopped and disabled | Set to Automatic and start it |
| 3 | "Server says disk is almost full" | 4 GB runaway log file | Find the largest files and remove the log |
| 4 | "The server is really slow" | Runaway process using a full CPU core | Identify it by live CPU and kill it |

---

## 1 — DNS Server Service Stopped

```text
--- SYMPTOM ---
Resolve-DnsName _ldap._tcp.dc._msdcs.lab.local -> DNS name does not exist
--- DIAGNOSE ---
Name Status  StartType
DNS  Stopped Automatic
Event 3 (DNS Server log): The DNS server has shut down.
--- FIX ---
Start-Service DNS
_ldap._tcp.dc._msdcs.lab.local -> dc01.lab.local, port 389
```

**Why it matters:** clients find domain controllers through DNS **SRV records**. With DNS down, logons, Group Policy and domain joins all fail, even though the DC itself is still running.

**What I learned along the way:** my first attempt was to point the DC's DNS client at a dead server (`10.99.99.99`). Names still resolved, because on a DC the local DNS Server service still answers. Stopping the service was the realistic failure.

## 2 — Print Spooler Disabled

```text
--- SYMPTOM ---
Get-Printer -> The spooler service is not reachable. Ensure the spooler service is running.
--- DIAGNOSE ---
Name    Status  StartType
Spooler Stopped Disabled
--- FIX ---
Set-Service Spooler -StartupType Automatic; Start-Service Spooler
Spooler Running Automatic -> printers listed again (Microsoft Print to PDF, XPS Document Writer)
```

**Tip:** if a service is *Disabled*, starting it fails. Fix the **startup type** first, then start it.

## 3 — Disk Filled by a Log File

```text
--- SYMPTOM ---
Free before: 37 GB  ->  Free now: 33 GB
--- DIAGNOSE: largest files on C: ---
SizeGB FullName
     4 C:\AppLogs\debug-trace.log
  0.52 C:\Windows\SoftwareDistribution\Download\...
  0.36 C:\Program Files (x86)\Microsoft\EdgeCore\...\msedge.dll
--- FIX ---
Free after cleanup: 37 GB
```

```powershell
Get-ChildItem C:\ -Recurse -File -ErrorAction SilentlyContinue |
  Sort-Object Length -Descending | Select-Object -First 3 FullName, Length
```

**Tip:** never delete system files to free space. Look for logs, temp files, old downloads and `SoftwareDistribution` (via Disk Cleanup), and confirm with the owner before removing anything.

## 4 — Runaway Process

```text
--- DIAGNOSE: live CPU per process (2-second sample) ---
Process    CPU %
powershell   156
svchost        0
Suspect PID 5768 command line: powershell.exe -NoProfile -WindowStyle Hidden -Command "while($true){}"
--- FIX ---
Stop-Process -Id 5768 -Force
--- VERIFY ---
PID 5768 still running: False
Top process now: dwm 5 %
```

**What I learned along the way:** my first diagnosis sorted by **total CPU time** (`Get-Process | Sort CPU`). That showed `System` and `lsass` at the top, because they had been running for hours. A process that just started looks small in that view. A **live sample** (`Get-Counter '\Process(*)\% Processor Time'`) found the culprit immediately.

**SOC angle:** a *hidden* PowerShell window running an endless loop is also how many cryptominers and malicious scripts look. Checking the **command line** of the process (`Win32_Process`) is the step that tells you whether it is a broken script or something worse.

---

## Troubleshooting Method
1. **Reproduce and describe the symptom** in the user's words.
2. **Check the obvious layer first:** is the service running, is the disk full, is the network reachable?
3. **Use the evidence:** event logs, service state, live counters.
4. **Fix the root cause**, not just the symptom (for example, the startup type, not only the running state).
5. **Verify** the fix and document it in the ticket.
