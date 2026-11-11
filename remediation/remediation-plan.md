# Remediation Plan

**Target:** Metasploitable 2 (intentionally vulnerable VM) on an isolated host-only network
**Tools:** Nmap 7.99, NVD, Ubuntu Security tracker
**Companion files:** `findings/vulnerability-register.csv` and the raw scans in `scans/`

## 1. Summary

This assessment tested an intentionally vulnerable Metasploitable 2 machine in an
isolated lab network. Nmap scanning and manual CVE research produced 6 findings:
2 Critical or Critical-equivalent, 3 High and 1 Medium. Severity combines CVSS v3,
CVSS v2 and analyst ratings, so the counts are indicative only.

Two findings were fixed and verified by rescan: the exposed Telnet service (V02) and
the root bind shell on port 1524 (V03). Both were closed by disabling the services
in inetd.conf and restarting xinetd. The first two attempts failed because the
wrong service was restarted, and the failed scans are kept as evidence.

Four findings remain: vsftpd 2.3.4, a release known to have shipped with a backdoor
(V01, Critical, open), the unpatched MySQL server (V06, High, confirmed), a possible
UnrealIRCd backdoor (V04, High, needs verification) and the end-of-life Apache 2.2.8
web server (V05, Medium, needs verification). These are not fixed in this lab, and the
recommended fixes are listed in section 3 in priority order. On a real system the
bind shell would also call for a full rebuild, since closing the port does not show
whether it was already used.

## 2. Fixes applied and verified

| ID | Finding | Fix applied | Evidence |
|----|---------|-------------|----------|
| V02 | Telnet (port 23) sends logins in plain text | Commented out the `telnet` line in `/etc/inetd.conf` and restarted xinetd | `scans/06-nmap-after-xinetd-restart.txt` shows port 23 closed |
| V03 | Root bind shell (port 1524) | Commented out the `ingreslock` line in `/etc/inetd.conf` and restarted xinetd | `scans/06-nmap-after-xinetd-restart.txt` shows port 1524 closed |

**What went wrong first, and how it was resolved**

1. Editing `/etc/inetd.conf` alone changed nothing. Scan `04` still showed both ports open.
2. Restarting `openbsd-inetd` did not help either. Scan `05` still showed both ports open.
3. `pgrep` and `fuser` showed that **xinetd** (PID 4510) owned the ports, not inetd.
4. Restarting xinetd applied the change, and scan `06` showed both ports closed.

**Lesson:** a configuration change only counts once you have confirmed which process
reads it and have verified the result with a rescan.

## 3. Remaining findings in priority order

Not fixed in this lab. Confirmed findings are placed ahead of unconfirmed ones at the
same priority level.

### Priority 1 - V01: vsftpd 2.3.4 (port 21)
- **Why first:** Critical (CVSS 9.8, CVE-2011-2523), reachable over the network with no login.
  The version matches a release known to have shipped with a backdoor. Not yet confirmed on this host.
- **Fix:** Replace with a current, maintained vsftpd from a trusted source, or move to SFTP and
  disable FTP.
- **Interim control:** Block port 21 from every network that does not need it.
- **Verify:** Rescan with `nmap -sV -p 21` and confirm the version has changed or the port is closed.

### Priority 2 - V06: MySQL 5.0.51a-3ubuntu5 (port 3306)
- **Why second:** Confirmed unpatched. The installed build is older than Ubuntu's fixed build
  5.0.51a-3ubuntu5.5 for CVE-2009-4484, a buffer overflow that lets remote attackers run code.
  The scan shows SSL support is advertised, which the attack depends on.
- **Fix:** Upgrade MySQL to a supported release.
- **Interim control:** Restrict port 3306 to the application server only. Disable SSL if it is not
  needed until the server is patched.
- **Verify:** Rescan with `nmap -p 3306 --script mysql-info` and confirm the new version.

### Priority 2 - V04: UnrealIRCd (port 6667)
- **Why third:** CVE-2010-2075 affects only some downloads of UnrealIRCd 3.2.8.1. The scan did not show
  a version, so this is unconfirmed.
- **Fix:** Confirm the installed version and where it came from. If it cannot be shown to be clean,
  replace it with a current release or remove the service.
- **Interim control:** Block port 6667 from untrusted networks.
- **Verify:** Rescan with `nmap -sV -p 6667` and record the version found.

### Priority 3 - V05: Apache httpd 2.2.8 (port 80)
- **Why fourth:** Apache 2.2 is end-of-life, so no security fixes are released. The example CVE
  (CVE-2008-2364) is a denial of service that only applies if mod_proxy is in use, so the age of the
  software is the bigger risk.
- **Fix:** Upgrade to a supported Apache 2.4 release.
- **Interim control:** Restrict access to the web server. Disable mod_proxy if it is not used.
- **Verify:** Rescan with `nmap -sV -p 80` and confirm a supported version.

## 4. Other exposed services not yet assessed

The first scan found 23 open ports. These were identified but not researched in depth: SSH (22),
SMTP (25), DNS (53), RPC (111), Samba (139 and 445), the r-services (512 to 514), Java RMI (1099),
NFS (2049), ProFTPD (2121), PostgreSQL (5432), VNC (5900), X11 (6000), AJP (8009) and Tomcat (8180).
The `shell`, `login`, `exec` and `tftp` entries were also enabled in `inetd.conf` and are candidates
for the next round of the register.

## 5. Retest plan

1. Apply fixes in the order above.
2. After each fix, rescan the affected port and save the output in `scans/` with the next number.
3. Update the status in `findings/vulnerability-register.csv`.
4. Mark a finding **Closed** only when a rescan proves the fix.

## 6. Limitations

- Findings are based on service versions and banners. No exploitation was attempted.
- Severity mixes CVSS v3, CVSS v2 and analyst ratings, which are not directly comparable.
- Vendors such as Ubuntu sometimes patch software without changing the version number, so
  version-based findings can overstate risk. That is why some findings are marked *needs verification*.
- The target is a deliberately vulnerable training machine, so results do not represent a real environment.
