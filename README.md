# Vulnerability Assessment & Remediation Lab

A hands-on vulnerability management project. I scanned an intentionally vulnerable machine (Metasploitable 2), researched and ranked the findings, fixed two of them, and proved the fixes with rescans.

## Lab setup
- **Target:** Metasploitable 2 in VirtualBox on a host-only network, isolated from my real network
- **Scanner:** Nmap 7.99 on Windows
- **Research:** NVD (CVE and CVSS data) and the Ubuntu Security tracker
- **Safety:** no exploitation was attempted, and the target was never exposed to a real network

## Method
1. Discover services and versions with Nmap
2. Research CVEs on NVD
3. Rate and prioritise each finding
4. Fix selected findings
5. Rescan to verify

## Findings summary

| ID | Finding | Severity | Status |
|----|---------|----------|--------|
| V01 | vsftpd 2.3.4, release known to have shipped with a backdoor (CVE-2011-2523) | Critical | Open |
| V02 | Telnet sends logins in plain text | High (analyst rating) | Closed - verified by rescan |
| V03 | Root bind shell on port 1524 | Critical (analyst rating) | Closed - verified by rescan |
| V04 | UnrealIRCd, possible backdoor (CVE-2010-2075) | High (CVSS v2) | Needs verification |
| V05 | Apache 2.2.8, end-of-life (example CVE-2008-2364) | Medium (CVSS v2) | Needs verification |
| V06 | MySQL 5.0.51a-3ubuntu5 older than the fixed build (CVE-2009-4484) | High (CVSS v2) | Confirmed, unpatched |

Severity mixes CVSS v3, CVSS v2 and analyst ratings, so it is indicative only.

## Files
- [Vulnerability register](findings/vulnerability-register.csv)
- [Remediation plan](remediation/remediation-plan.md)
- [Raw Nmap scans](scans/), starting from the initial scan

## The troubleshooting story
Disabling telnet and the root shell took three tries. Editing `inetd.conf` changed nothing, and restarting `openbsd-inetd` did not help either. Checking which process owned the ports showed it was **xinetd**. Restarting xinetd closed both ports. The failed scans (04 and 05) are kept next to the successful one (06) as evidence.

## What I learned
- A configuration change only counts once you confirm which process reads it and verify with a rescan
- A version match is not proof of a vulnerability, because vendors backport patches and some backdoors affected only tampered downloads
- CVSS v2 and v3 scores should not be compared directly
- Findings need an owner, a priority and a verification step, not just a scanner screenshot
