# Lab 07 — Help Desk Ticketing with osTicket

**Goal:** Run a real ticketing system for the `lab.local` domain and work tickets the way a Level 1 help desk does: categorize, prioritize with SLAs, investigate, write internal notes, reply to the user, and close.

**Environment**

| Item | Value |
|---|---|
| Server | DC01 (Windows Server 2022) |
| Stack | XAMPP 8.2.12: Apache 2.4, MariaDB, PHP 8.2 |
| App | osTicket v1.18.4 at `http://helpdesk.lab.local/helpdesk/` |
| Access from my laptop | VirtualBox NAT Network port forward `127.0.0.1:8080 → 10.0.2.15:80` |

> The tickets are realistic scenarios based on the faults I reproduced and fixed in Labs 02–05. Credentials are lab-only and not published.

---

## Part 1 — Install the stack

```powershell
# Unpack osTicket and publish the web files
tar -xf osticket.zip -C C:\Install\ost
robocopy C:\Install\ost\upload C:\xampp\htdocs\helpdesk /E
Copy-Item ...\include\ost-sampleconfig.php ...\include\ost-config.php

# Run Apache and MariaDB as Windows services
httpd.exe -k install -n Apache2.4
mysqld.exe --install mysql --defaults-file=C:\xampp\mysql\bin\my.ini
Start-Service mysql, Apache2.4

# Database with its own least-privilege account
CREATE DATABASE osticket CHARACTER SET utf8mb4;
CREATE USER 'osticket'@'localhost' IDENTIFIED BY '********';
GRANT ALL ON osticket.* TO 'osticket'@'localhost';

# Network access and a friendly name
New-NetFirewallRule -DisplayName 'HTTP - Helpdesk' -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
Add-DnsServerResourceRecordCName -ZoneName lab.local -Name helpdesk -HostNameAlias dc01.lab.local
```

```
Name      Status  StartType
----      ------  ---------
Apache2.4 Running Automatic
mysql     Running Automatic
```

Then I ran the web installer (help desk name, admin account, database settings). The prerequisite check passed: PHP 8.2.12 and the MySQLi extension.

**Post-install hardening** (the installer asks for this):
```powershell
Rename-Item C:\xampp\htdocs\helpdesk\setup setup_disabled      # installer can't be re-run
icacls include\ost-config.php /reset; attrib +R include\ost-config.php   # config is read-only
```

## Part 2 — Configure the help desk

**SLA plans** (time to resolve):

| SLA | Grace period | Used for |
|---|---|---|
| P1 - Critical | 4 h | Whole team or a server is down |
| P2 - High | 8 h | One user is blocked with no workaround |
| P3 - Normal | 24 h | A workaround exists, or it is a request |

**Help topics** (what users pick when they open a ticket). Each one sets the department, priority, and SLA, and assigns the ticket to the *Level I Support* team:

| Help topic | Priority | SLA |
|---|---|---|
| Server / Service Outage | Emergency | P1 |
| Account Locked / Password Reset | Normal | P2 |
| Network / Mapped Drive | Normal | P3 |
| Printer / Print Spooler | Normal | P3 |
| New Hire / Leaver Request | Normal | P3 |

**API key** for importing tickets, restricted to `127.0.0.1` with the *create tickets* permission only.

## Part 3 — Tickets come in

Five users reported problems. Tickets were created through the osTicket API (`POST /api/tickets.json`):

```
Nobody in Sales can open any website or internal page -> #505924
Account locked after password change                  -> #384154
S: drive missing after login                          -> #860798
Cannot print - documents stuck in queue               -> #516447
New hire starting Sunday - needs account and access   -> #284858
```

The help topic routed each one automatically. The outage arrived as **Emergency / P1**, and every ticket was assigned to *Level I Support*.

## Part 4 — Work the tickets

For each ticket I followed the same steps: **investigate → internal note (what I found and did) → reply to the user in plain language → set the status.**

| Ticket | User | Root cause (internal note) | Fix | Status |
|---|---|---|---|---|
| #505924 (P1) | Khalid Saad, Sales | `ping` worked but `nslookup` timed out → DNS Server service stopped on DC01 (Event 7036) | Started the service and set it to Automatic | Resolved |
| #384154 | Sara, HR | Event **4740** on DC01: lockout came from CLIENT01; old password saved in Credential Manager | `Unlock-ADAccount`, user removed the saved credential | Resolved |
| #860798 | Turki Alqahtani, Sales | Logon script ran but `net use S:` → Access denied; user not in **GRP-Sales** | `Add-ADGroupMember GRP-Sales`, sign out and in | Resolved |
| #516447 | Maha Alotaibi, HR | Print Spooler stopped and Disabled | Set to Automatic, started it, cleared the spool folder, printed a test page | Closed |
| #284858 | Noura Saleh, HR manager | Request, not a fault | Account created **disabled** with groups, home folder, and manager set; enable on the start date | **Open** (waiting for the start date) |

**Example — ticket #505924 as it appears in osTicket**

```
Ticket #505924  Nobody in Sales can open any website or internal page
Status: Resolved      Priority: Emergency      SLA Plan: P1 - Critical (4h)
Help Topic: Server / Service Outage            Source: API

Khalid Saad posted 11:31   Since 9:00 nobody on our floor can open websites or the shared
                           drive by name. Teams also says no connection. Urgent.
Ticket Filter assigned this to Level I Support
Alaa Ghareeb (internal note) 11:34   Root cause: DNS Server service stopped on DC01
    Checked from a client: ipconfig OK, ping 10.0.2.15 OK, nslookup dc01.lab.local timed out
    -> name resolution problem, not network. Started the service, set it to Automatic.
Alaa Ghareeb (reply) 11:34   The DNS service on the domain controller had stopped, so computers
    could not translate site names into addresses. I restarted it ... please let me know if
    anyone on your floor still has a problem.
Closed by Alaa Ghareeb with status of Resolved
```

**Queues at the end**

```
Closed:  #516447  #860798  #384154  #505924   (Closed By: Alaa Ghareeb)
Open:    #284858  New hire starting Sunday     (Assigned to me)
```

---

## Writing Ticket Notes — What I Practiced

- **Internal note = for IT.** Exact evidence (commands, Event IDs, service names) so the next technician can follow it.
- **Reply = for the user.** No jargon. Say what was wrong, what was done, and what the user needs to do, if anything.
- **Resolved vs Closed.** *Resolved* means fixed and the user can still reply; *Closed* means finished.
- **Requests are not incidents.** The new-hire ticket stays open until the work is actually done on the start date.

## What I Learned
- How a ticketing system turns "it's broken" into categories, priorities, SLAs, and an audit trail.
- Help topics can route and prioritize tickets automatically, so a P1 outage doesn't wait behind a password reset.
- Setting up a LAMP-style app on Windows: web server and database as services, a least-privilege database account, firewall rule, DNS name, and post-install hardening.
- Using an API key safely: restricted by source IP and limited to the permission it needs.

## Problems I Hit
| Problem | Fix |
|---|---|
| Posting replies through the agent panel returned *"This action requires a lock"* | osTicket locks a ticket while an agent edits it; acquire the lock first, then post |
| `Expand-Archive` took over 10 minutes on the external disk | Used the built-in `tar -xf`, which finished in seconds |
| Laptop browser couldn't reach the VM's web server | Added a NAT Network port forward `127.0.0.1:8080 → 10.0.2.15:80` |
