# SOC Analyst Bootcamp – Week 1 Team Project
## Build, Baseline & Investigate a Small IT Environment

| | |
|---|---|
| **Author** | Samuel Ayodele Christian |
| **Team / role** | **Team sentinel** |
| **Dates** | 9–10 October 2026 |
| **Time zones** | Windows and Kali use WAT (UTC+1). svr1 logs are in **UTC**, so 22:34 WAT = 21:34 UTC. 

---

## 01 Executive summary

I built and documented a three-machine lab in VirtualBox: a Windows 10 endpoint (`WIN-ENDPOINT`), an Ubuntu 24.04 server (`svr1`) and a Kali Linux testing machine (`kali`). All three sit on one NAT Network (10.0.2.0/24). I proved they communicate (ping, DNS, SSH), recorded a Windows baseline (processes, services, ports, firewall, users, logon events), and carried out Linux administration (users, a group, file permissions, sudo and service changes, normal and failed logins).

I then found the evidence each action left in svr1's logs and in Windows Event Viewer, captured normal traffic in Wireshark (ICMP, DNS, TCP, SSH, HTTPS), and ran a controlled TCP port scan from Kali against svr1. The main findings were:

- Almost every admin action is recorded in svr1's logs, tied to the user who ran it through `sudo`.
- Failed SSH logins are logged with source IP and user name, and invalid users are labeled as such.
- The port scan was clearly visible in Wireshark (1,024 SYN packets in about a second) but left **no entries** in svr1's `auth.log` or SSH journal.

This shows that host logs alone are not enough, and that network visibility must be part of central monitoring.

---

## 02 Environment overview and network diagram

| Machine | Role | Purpose in the project |
|---|---|---|
| WIN-ENDPOINT | Employee workstation | Endpoint baseline, process and Event Viewer analysis |
| svr1 | Organization server | Users, permissions, logs; planned SIEM host next week |
| kali | Security testing machine | Controlled testing, Wireshark captures |

**Network:** VirtualBox NAT Network, 10.0.2.0/24. IP addresses come from DHCP (server 10.0.2.2). Default gateway 10.0.0.1, DNS server 10.111.216.55.

```
                     Internet
                        |
        Gateway 10.0.0.1 / DNS 10.111.216.55
                        |
 ============ NAT Network 10.0.2.0/24 (VirtualBox) ============
      |                     |                      |
 WIN-ENDPOINT              svr1                   kali
 10.0.2.4                  10.0.2.3               10.0.2.15
 Windows 10                Ubuntu 24.04.4 LTS     Kali Linux
 Employee workstation      Organization server    Security testing

 kali ----SSH (TCP 22)----> svr1
 kali <--- ICMP ping ---> svr1 and WIN-ENDPOINT
```

![Network diagram](evidence/A-network-diagram.png)
**[TODO: draw this diagram in draw.io, export as `A-network-diagram.png`]**

### Communication verification

| From | To | Address used | Method | Result |
|---|---|---|---|---|
| kali | svr1 | 10.0.2.3 | `ping -c 4` | 4/4 replies, 0% loss, TTL 64 |
| kali | WIN-ENDPOINT | 10.0.2.4 | `ping -c 4` | **100% loss** (before the ICMP rule; 4/4 replies (TTL 128)after it |
| WIN-ENDPOINT | svr1, kali | 10.0.2.3, 10.0.2.15 | `ping` | 0% loss, TTL 64 |
| svr1 | kali, WIN-ENDPOINT | 10.0.2.15, 10.0.2.4 | `ping -c 4` | 0% loss; Windows replies had TTL 128 |
| any | DNS 10.111.216.55 | google.com | `nslookup` (captured in Wireshark) | Query answered |
| kali | svr1 | 10.0.2.3:22 | `ssh` | Login succeeded (service connection) |

**Windows firewall note.** The failed ping to Windows was caused by the default firewall, not a network fault, because Kali reached svr1 on the same network. I added an inbound ICMPv4 allow rule with: `netsh advfirewall firewall add rule name="Allow ICMPv4 In" protocol=icmpv4 dir=in action=allow` (it returned **Ok.**). After the rule, Kali's ping to 10.0.2.4 succeeded (4/4 replies, 0% loss, TTL 128), and `ip neigh` on Kali showed Windows' MAC address 08:00:27:e2:c6:f5 as REACHABLE.



![Kali pings svr1 and Windows after the rule](evidence/A-ping-kali-after-rule.png)



Evidence: `A-ping-kali-after-rule.png`.
**Why TTL differs:** Linux hosts reply with TTL 64 and Windows with TTL 128 by default, which is consistent with each machine's operating system.

---

## 03 Asset and network inventory

| Hostname | OS | IP | MAC | Role | Adapter | Gateway | DNS |
|---|---|---|---|---|---|---|---|
| WIN-ENDPOINT | Windows 10 (build 19045.3803) | 10.0.2.4 | 08-00-27-E2-C6-F5 | Employee workstation | Intel PRO/1000 MT Desktop, VirtualBox NAT Network | 10.0.0.1 | 10.111.216.55 |
| svr1 | Ubuntu 24.04.4 LTS | 10.0.2.3/24 (enp0s3) | 08:00:27:d9:85:7c | Organization server | VirtualBox NAT Network | 10.0.0.1 | 10.111.216.55 |
| kali | Kali Linux **[TODO: version from `/etc/os-release`]** | 10.0.2.15/24 (eth0) | 08:00:27:90:fa:3d | Security testing | VirtualBox NAT Network | 10.0.0.1 | 10.111.216.55 |

All addresses are DHCP-assigned, so they may change after a reboot.
**[TODO: screenshots of each VM's VirtualBox Settings > Network showing "NAT Network", and Kali `hostname` + `cat /etc/os-release`.]**

### Lab verification

| Member | Connectivity test | Evidence |
|---|---|---|
| Haywhy | Kali → svr1 ping 0% loss; Windows → svr1/Kali ping 0% loss; SSH Kali → svr1 | `A-ping-*.png`, `C-ssh-session.png` |
| **[TODO: other members]** | | |

---

## 04 Windows baseline

**System:** hostname `WIN-ENDPOINT`, Windows 10 (10.0.19045.3803), IP 10.0.2.4, logged-in user `Haywhy` (administrator).
Evidence: `B-system-info.png`, `B-winver.png`, `B-ipconfig.png`.

### Selected processes
*PIDs change at each boot. The values below are from the session in which they were captured.*
**[TODO: add the `B-processes-marked.png` screenshot with the Image path name column, and verify each path.]**

| # | Process | PID | User | Expected path |
|---|---|---|---|---|
| 1 | lsass.exe | 676 | SYSTEM | C:\Windows\System32\lsass.exe |
| 2 | explorer.exe | 3572 | Haywhy | C:\Windows\explorer.exe |
| 3 | csrss.exe | 452, 536 | SYSTEM | C:\Windows\System32\csrss.exe |
| 4 | MsMpEng.exe | 2296 | SYSTEM | Under Windows Defender's folder |
| 5 | msedge.exe | 5128 and others | Haywhy | ...\Microsoft\Edge\Application\msedge.exe |

**1. lsass.exe**
- *What it does:* It checks usernames and passwords at login, enforces security policy, handles password changes, creates the access token for the session, and writes logon events to the Security log. On this machine it hosts the services SamSs, KeyIso and VaultSvc (`tasklist /svc`).
- *Why it matters:* If it stops, nobody can log in and Windows restarts. It also holds credential material in memory while users are signed in, so attackers target it.
- *Expected?* Yes. Exactly one copy, running as SYSTEM from System32.
- *Suspicious if:* it runs from another folder (Downloads, Temp), runs as a normal user, there is more than one copy, or the name is a lookalike such as `Isass.exe` (capital I). I would verify path, signature and parent process before concluding anything.

**2. explorer.exe**
- *What it does:* It provides the desktop, taskbar, Start menu and file windows.
- *Why it matters:* It is the user's interface and the usual parent of programs the user starts, so it appears in most process trees.
- *Expected?* Yes. One per signed-in desktop, running as the logged-in user.
- *Suspicious if:* it runs as SYSTEM, from another path, spawns unusual command-line tools, or makes network connections.

**3. csrss.exe**
- *What it does:* It is a core Windows subsystem that supports console windows and session handling.
- *Why it matters:* It is critical, and Windows fails if it is terminated.
- *Expected?* Yes. Two copies (PIDs 452 and 536), one per session, running as SYSTEM.
- *Suspicious if:* it runs as a normal user, from another path, there are extra copies, or the name is slightly different.

**4. MsMpEng.exe**
- *What it does:* It is the Microsoft Defender antimalware engine and scans files and processes (service `WinDefend`).
- *Why it matters:* It is the endpoint's first line of defence. If it is missing or stopped, protection may have been disabled or tampered with.
- *Expected?* Yes. One copy running as SYSTEM, with `NisSrv.exe` and `MpDefenderCoreService.exe` alongside it.
- *Suspicious if:* it is stopped, runs from an unexpected path, or shows unexplained sustained high CPU.

**5. msedge.exe**
- *What it does:* It is the Edge browser, which uses many processes (browser, GPU, renderers).
- *Why it matters:* The browser is a main entry point for phishing and downloads.
- *Expected?* Yes. Several copies are normal, running as the logged-in user.
- *Suspicious if:* it runs from Temp or Downloads, runs as SYSTEM, is launched by an Office app or script, or connects to unexpected addresses.

### Services, ports, connections, firewall, users

- **Services (`net start`):** a normal Windows 10 set: DHCP Client, DNS Client, Windows Event Log, Windows Defender Firewall, Microsoft Defender services, Print Spooler, Server, Workstation, plus **VirtualBox Guest Additions Service** (lab-specific). Evidence: `B-services.png`.
- **Listening ports (`netstat -ano | findstr /i listening`):**

| Port | PID | Process | Meaning |
|---|---|---|---|
| 135 | 900 | svchost (RpcSs) | RPC |
| 445, 139 | 4 | System | SMB file sharing / NetBIOS |
| 5040 | 1092 | svchost (CDPSvc) | Connected Devices Platform |
| 49664 | 676 | lsass.exe | Dynamic RPC |
| 49665 | 528 | wininit.exe | Dynamic RPC |
| 49666 | 540 | svchost (EventLog, Dhcp) | Dynamic RPC |
| 49667 | 1132 | svchost (Schedule, UsoSvc and others) | Dynamic RPC |
| 49668 | 2020 | spoolsv.exe | Print Spooler |
| 49669 | 660 | services.exe | Dynamic RPC |

- **Active connection:** 10.0.2.4:49688 → 98.66.133.184:443 (PID 1132, `svchost`, which hosts UsoSvc/WpnService). This is consistent with Windows background update or notification traffic; I did not verify the remote owner.
- **Firewall:** Domain, Private and Public profiles all **ON**.
- **Users:** `query user` shows only `haywhy`, console session, active, logged on 10/9/2026 12:29 PM.
- Evidence: `B-listening-ports.png`, `B-connections.png`, `B-firewall.png`, `B-users.png`, `B-tasklist.png`.

---

## 05 Linux administration (svr1)

### SSH service
SSH was installed but showed `inactive (dead)` and `disabled`, although port 22 was listening (Ubuntu 24.04 uses socket activation). I ran `sudo systemctl enable --now ssh`, after which the service showed `active (running)` and `enabled`. This was my administrative service action.
Evidence: `C-ssh-before-enable.png`, `C-ssh-after-enable.png`.

### Users and group
```
sudo adduser analyst1        # UID 1002
sudo adduser analyst2        # UID 1003
sudo groupadd soc-team       # GID 1004
sudo usermod -aG soc-team analyst1
sudo usermod -aG soc-team analyst2
getent group soc-team        # soc-team:x:1004:analyst1,analyst2
sudo gpasswd -d analyst2 soc-team
getent group soc-team        # soc-team:x:1004:analyst1
```
`-aG` appends a supplementary group. Without `-a`, `usermod -G` would replace the user's other groups.
Evidence: `C-users-created.png`, `C-group-before-removal.png`, `C-group-after-removal.png`.

### Permissions
I created `report.txt`, moved the folder to `/srv/soc-data` (home folders are private, which would have blocked other users for the wrong reason), and set:

```
drwxr-x---  analyst1 soc-team  /srv/soc-data          (750)
-rw-r-----  analyst1 soc-team  /srv/soc-data/report.txt (640)
```

| Account | Relationship | Result | Why |
|---|---|---|---|
| analyst1 | Owner | Read and append worked | Owner has `rw-` |
| soc-team members | Group | Read only | Group has `r--` |
| analyst2 | Removed from group, so "others" | **Permission denied** | Others have `---` |
| haywhy | Not owner, not in group | **Permission denied** (without sudo) | Others have `---` |

Evidence: `C-permissions.png`, `C-test-access.png`, `C-test-haywhy-denied.png`.

### Logins from Kali (10.0.2.15) to svr1
- Normal: `ssh analyst1@10.0.2.3`, successful, `whoami` returned analyst1.
- Controlled failures: `ssh analyst2@10.0.2.3` (wrong password ×3) and `ssh fakeuser@10.0.2.3` (nonexistent user, wrong password ×3). Kali clock: 22:34:09 WAT and 22:34:38 WAT.

Evidence: `C-login-success.png`, `C-login-failed.png`.

---

## 06 Evidence and logs

| Activity | Evidence found | Where | What it tells me |
|---|---|---|---|
| User creation | `sudo ... COMMAND=/usr/sbin/adduser analyst1` (20:46:06), `useradd: new user: name=analyst1, UID=1002`; same for analyst2 (20:47:57) | svr1 `/var/log/auth.log` | Who ran it (haywhy via sudo), what was created, when |
| Group creation / membership | `groupadd: new group: name=soc-team, GID=1004` (20:49:47); `usermod: add 'analyst1' to group 'soc-team'` (20:50:21) | `auth.log` | Group created, members added |
| Group removal | `COMMAND=/usr/bin/gpasswd -d analyst2 soc-team` (21:01:08) | `auth.log` | Membership change tied to haywhy |
| Permission changes | `chown analyst1:soc-team ...`, `chmod 640 ...` (21:05–21:06), `chown`/`chmod 750 /srv/soc-data` (21:10–21:11) | `auth.log` (sudo lines) | Ownership and mode changes recorded |
| Admin action (service) | `COMMAND=/usr/bin/systemctl enable --now ssh` (14:29:21, `TTY=tty1`) | `auth.log` | Done from the VM console, not over SSH |
| Successful login | `Accepted password for analyst1 from 10.0.2.15 port 45372` (21:33:43) | `auth.log` | Source IP is Kali |
| Failed login (real user) | `Failed password for analyst2 from 10.0.2.15` + `message repeated 2 times` (21:34:33) | `auth.log` | 3 failures in one connection |
| Failed login (invalid user) | `Invalid user fakeuser from 10.0.2.15` then `Failed password for invalid user fakeuser` ×3 (21:35:06 onward) | `auth.log` | The log shows the account does not exist |
| Windows logon (user) | Event 4624, **Logon Type 2**, Account `Haywhy`, 12:29:50 PM, process `svchost.exe`, source 127.0.0.1, elevated token Yes | Event Viewer > Windows Logs > Security | Interactive login, matches `query user` |
| Windows logon (system) | Event 4624, **Logon Type 5**, account `WIN-ENDPOINT$`, SYSTEM | Event Viewer | Service logon, normal background noise (249 events in total) |

Search used on svr1: `sudo grep -aiE "accepted|failed|invalid" /var/log/auth.log` (`-a` treats the log as text, `-i` ignores case).
Note: log entries dated 2026-06-12 (user `ayodele`) existed before this project and are not part of my work.
Evidence: `D-ssh-auth-log.png`, `D-user-admin-log.png`, `D-accounts-log.png`, `B-event-4624-haywhy.png`, `B-event-4624-type5.png`.

---

## 07 Wireshark analysis (capture on kali, interface eth0)

| Traffic | Source → Destination | Protocol / ports | Key detail | Generated by | Why it appears |
|---|---|---|---|---|---|
| Ping | 10.0.2.15 ↔ 10.0.2.3 | ICMP (no ports) | 4 Echo requests (type 8) and 4 replies (type 0), 98 bytes, 1 s apart | `ping -c 4 10.0.2.3` | Wireshark sees every packet on Kali's interface |
| DNS | 10.0.2.15 → 10.111.216.55 | DNS over UDP, port 53 | Query `0xaff7` for **A** record of google.com, second query for **AAAA**; responses returned | `nslookup google.com` | Kali asked its configured DNS server |
| TCP handshake | 10.0.2.15:44676 → 10.0.2.3:22 | TCP | `[SYN]` (frame 19), `[SYN, ACK]` (frame 20), then ACK | Start of SSH session | A TCP connection must be set up before SSH starts |
| SSH | 10.0.2.15:44676 ↔ 10.0.2.3:22 | SSHv2 | Version banners, Key Exchange Init, Diffie-Hellman, New Keys, then only "Encrypted packet" (178 of 219 packets) | `ssh analyst1@10.0.2.3` | Content is encrypted, so passwords and commands are unreadable |
| HTTPS | 10.0.2.15:43664 → 104.20.23.154:443 | TLS 1.3 over TCP | Client Hello with **SNI = example.com**, Server Hello, Change Cipher Spec, then Application Data | `curl -I https://example.com` | Site name is visible in the handshake; content is encrypted |

**Background traffic:** in one idle capture I also saw ICMPv6 Router Solicitation, NTP client/server exchange (103.97.91.223) and ARP "Who has 10.0.0.1?" with the gateway answering at 52:54:00:12:35:00. These are normal for a machine on a network.

Evidence: `E-icmp.png`, `E-dns.png`, `E-tcp-handshake.png`, `E-ssh.png`, `E-https.png`, `E-baseline.pcapng`.
**[TODO: save and attach `E-https.pcapng` if the HTTPS capture was not saved; the 104.20.23.154 address is simply what example.com resolved to during the test.]**

---

## 08 Controlled security activity

| Item | Detail |
|---|---|
| **What** | TCP connect scan of ports 1–1024: `nmap -sT -p 1-1024 10.0.2.3` (nmap 7.99) |
| **From → against** | kali (10.0.2.15) → svr1 (10.0.2.3), my own lab only |
| **When** | Sat 10 Oct 2026, 12:18:36 WAT (11:18 UTC) |
| **Why** | To see which services svr1 exposes and what evidence a scan leaves |
| **Result** | `22/tcp open ssh`; 1,023 ports closed (`conn-refused`); scanned in 0.72 s; MAC 08:00:27:D9:85:7C confirmed it was svr1 |
| **Earlier attempt** | At 12:14:05 WAT nmap reported "Host seems down". svr1's SSH service only started at 11:17:14 UTC (12:17 WAT), so svr1 had most likely just booted. |

**Evidence generated**
- *Wireshark:* filter `tcp.flags.syn==1 && tcp.flags.ack==0` showed **1,024 SYN packets** from 10.0.2.15 to 10.0.2.3, each to a different port in random order, microseconds apart, within about one second (2,096 packets in the capture in total).
- *svr1 logs:* no entries from the scan in `auth.log`, and `journalctl -u ssh --since "2026-10-10 11:15" --until "2026-10-10 11:25"` showed only the SSH service start at 11:17. Closed ports refuse the connection without any application logging it.

Evidence: `F-nmap-output.png`, `F-nmap-host-down.png`, `F-wireshark-scan.png`, `F-svr1-log.png`, `F-svr1-journal.png`, `F-portscan.pcapng`.

---

## 09 Before vs after

*Baseline → Change → Evidence → Investigation.*

| Observation | Normal baseline | Controlled activity | Difference | Security relevance |
|---|---|---|---|---|
| Ping Kali → Windows | 100% loss (inbound ICMP blocked) | After I added the ICMP allow rule: **[TODO confirm replies]** | Rule added | Deliberate firewall change, documented |
| svr1 listening ports | Only 22 (`ss -tlnp`) | Same after the scan | None | Scan found only the expected service |
| Traffic Kali → svr1 | A few pings and one SSH session | 1,024 SYN packets in about 1 second | Large burst to many ports | Typical reconnaissance pattern |
| svr1 host logs | Logins, failures and sudo commands all logged | No entries for the scan | Scan invisible in host logs | Host logs alone would miss this |
| Linux accounts | haywhy and a pre-existing `ayodele` | analyst1, analyst2, group soc-team | New accounts and group | Account changes logged via sudo |
| SSH service | `inactive (dead)`, `disabled` | `active (running)`, `enabled` | Service enabled | Logged as a sudo command |
| Failed logins | None | 3 + 3 failed attempts, one invalid user | New failures from 10.0.2.15 | Possible password guessing pattern |

I separate what I **observed** (the log lines and packets above) from what I **concluded**. For example, the scan was observed in Wireshark; the conclusion that host logs miss it comes from checking `auth.log` and the journal.

---

## 10 Mini investigation: repeated failed SSH logins

**Observation**
On 9 Oct 2026, svr1 recorded repeated failed SSH logins from 10.0.2.15 for two accounts: `analyst2` (exists) and `fakeuser` (does not exist).

**Evidence** (svr1 `/var/log/auth.log`, UTC)
- 21:34:33 `Failed password for analyst2 from 10.0.2.15`, repeated 3 times in one connection
- 21:35:06 `Invalid user fakeuser from 10.0.2.15`, followed by 3 `Failed password for invalid user fakeuser`
- 21:35:19 `Connection closed by invalid user fakeuser ... [preauth]`
- 21:36:54 `Failed password for haywhy`, then 21:37:00 `Accepted password for haywhy`
- Kali's `date` output (22:34 WAT) matches 21:34 UTC.

**Interpretation**
- Three failures per connection matches SSH's default of 3 password tries per connection.
- The log distinguishes a real user (`analyst2`) from an invalid one (`fakeuser`), even though the person typing sees the same "Permission denied".
- All attempts came from one source IP within a few minutes.
- No lockout or blocking occurred.

**Alternative explanation**
A user simply mistyping a password. The `haywhy` entries fit that: one failure, then success six seconds later. Repeated failures against a nonexistent username do not fit it.

**Conclusion**
The pattern resembles password guessing, but this was a controlled test I ran from my own Kali machine against my own server. Unusual is not automatically malicious, and the source IP is a known asset in my inventory.

**Additional evidence I would want in a real investigation**
- Whether the source IP is a known asset and who uses it
- Whether a **success** followed failures from the same IP (it did here, for analyst1 and haywhy)
- Longer log history for the same IP and usernames
- Host evidence from the source machine (shell history, running processes)
- A baseline of normal login times, and any lockout or rate-limit settings
- A network capture to confirm volume and timing

---

## 11 Recommendations

**What to monitor centrally next (SIEM)**
1. **Linux authentication logs** from svr1: `sshd` accepted/failed/invalid-user lines, with source IP, and `sudo` commands.
2. **Windows Security events**: 4624 (successful logon, with Logon Type), 4672 (privileged logon), and failed logons if auditing is enabled **[confirm audit settings]**.
3. **Account and group changes**: `useradd`, `groupadd`, `usermod`, `gpasswd`.
4. **Service changes** (for example `systemctl enable`) and **new listening ports** compared with this baseline.
5. **Network telemetry**: bursts of SYN packets to many ports from one source, which host logs missed in my scan. This needs a network sensor or captured flows.
6. **Time handling**: svr1 logs in UTC while the other machines use local time. Use one standard (UTC) in the SIEM.

**Hardening ideas**
- Use SSH key authentication and disable password login, and add rate-limiting or lockout for failed attempts.
- Remove or disable unused accounts and review group membership regularly (a pre-existing `ayodele` account was found).
- Apply pending updates on svr1 (the login banner reports 29 available).
- Keep the Windows firewall on and record every rule change, such as the ICMP rule.
- Alert on a success following several failures from the same source IP.

---

## 12 Conclusion

I built three connected machines, documented their addressing, and proved they communicate. I recorded what normal looks like on the Windows endpoint, performed Linux administration, and then found the evidence of those actions in svr1's logs and in Windows Event Viewer. Wireshark showed what ping, DNS, TCP, SSH and HTTPS traffic looks like on the wire. A controlled port scan against my own server was clear in the packet capture but missing from the host logs, and a set of failed SSH logins was fully logged.

The main lesson is that each data source sees something different: host logs show who did what on a machine, and network data shows what crossed the wire. Next week's centralized logging should combine both, with consistent timestamps, so unusual activity can be spotted and then investigated before it is called malicious.

