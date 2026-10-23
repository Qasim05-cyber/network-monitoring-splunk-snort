# Network Traffic Monitoring with Snort and Splunk

I set up firewall logging and intrusion detection, then tested whether I could find suspicious traffic in the logs. I used pfSense, Snort, Kiwi Syslog, and Splunk, followed by an Nmap scan and an Infection Monkey simulation from the DMZ.

## What I found

- Snort detected the test pings and generated alerts for scanning and possible exploitation attempts.
- Kiwi received pfSense firewall logs, and Splunk indexed syslog events over UDP port 514.
- Infection Monkey reported **12 open services across six machines** and finished after **23 minutes and 1 second**.
- The DMZ simulation reached services on internal LAN systems. That was my biggest concern because it provides paths an attacker could investigate after compromising a DMZ host.
- Splunk showed traffic from the Monkey server, including priority 1 possible CVE-2014-6271 attempts and SSH scan alerts. These show detected activity; an alert alone does not confirm a successful compromise.

## Environment

| System | Role / address |
| --- | --- |
| pfSense | Firewall; LAN 172.30.0.1, DMZ 172.31.0.1 |
| Windows Server 2019 | LAN workstation and logging tools; 172.30.0.2 |
| TargetLinux01 | Ubuntu DMZ target; 172.31.0.20 |
| TargetLinux02 | Unix DMZ target; 172.31.0.30 |
| Infection Monkey | DMZ simulation server, clean-ubuntu; 172.31.0.50 |
| AttackLinux | WAN scan source; 202.20.1.2 |
| Splunk Enterprise | SIEM on the Windows workstation |

All testing took place in the provided isolated lab environment.

## 1. Firewall logs and intrusion detection

I reviewed pfSense system and firewall logs, configured a Snort pass list, and enabled Snort on LAN. I sent three pings from the DMZ interface to 172.30.0.2. All three received replies, and Snort recorded six ICMP alerts from two signatures.

I forwarded firewall logs to Kiwi Syslog and enabled Snort on the DMZ interface with community rules. Both Snort interfaces were running with **blocking disabled**, so this tested detection rather than automatic prevention.

## 2. Perimeter scan and centralized logging

I ran the following command in Zenmap against the lab's public-facing target subnet:

```bash
nmap -sS -sU --script auth 172.40.0.0/27
```

The scan completed in 94.65 seconds, checking 32 IP addresses and finding two hosts up. The visible results included FTP, SSH, Telnet, HTTP, VNC, X11, and IRC. Anonymous FTP login was allowed, although directory listing failed because the advertised passive address differed from the public-facing address.

I searched Splunk for Snort events and observed alerts such as **ET SCAN Potential SSH Scan**, **403 Forbidden**, and **.htaccess access**.

## 3. DMZ breach simulation

I configured Infection Monkey to include the LAN subnet and ran it from the Monkey Island server at 172.31.0.50. After it finished, I reviewed the infection map and security report.

The report listed these services as reachable from clean-ubuntu:

| Address | System | Discovered TCP ports |
| --- | --- | --- |
| 172.30.0.2 | Windows Server 2019 | 445, 22, 80 |
| 172.30.0.1 | Reported as unknown | 80, 22 |
| 172.31.0.20 | Ubuntu-4 | 80, 22 |
| 172.31.0.30 | Unix | 80, 445, 22 |
| 172.31.0.1 | Reported as unknown | 22, 80 |
| 172.31.0.50 | clean-ubuntu | No services listed in this table |

The map showed scan paths and an exploit path toward the Unix target. The report flagged critical security issues. Reachable services do not by themselves prove that a LAN host was infected.

## 4. Finding the simulation in Splunk

The captured search used:

```spl
snort
```

I matched the simulation's time window and source address, 172.31.0.50, to the returned events. The results included:

| Alert | Source | Destination | Meaning |
| --- | --- | --- | --- |
| Possible CVE-2014-6271 Attempt | 172.31.0.50 | 172.31.0.30:80 | Possible web exploitation attempt, priority 1 |
| Possible CVE-2014-6271 Attempt in Headers | 172.31.0.50 | 172.31.0.20:80 | Possible web exploitation attempt, priority 1 |
| Potential SSH Scan OUTBOUND | 172.31.0.50 | 172.30.0.13:22 | Scanning toward a LAN address, priority 2 |

The last event is evidence of a scan attempt, not proof that 172.30.0.13 was reachable or compromised.

A more focused follow-up search would be:

```spl
snort "172.31.0.50"
```

That narrows the results to events mentioning the initial Monkey server. It can miss later activity from other hosts, so the time window and infection map still matter. This refinement is a suggested improvement, not the query shown in the screenshot.

## What I would improve

The results were useful, but important details were crowded into the raw event text. I would extract source and destination IPs, ports, protocol, signature, rule ID, classification, and priority into separate fields. I would also normalize time zones: the event text and Splunk display showed different times.

For the main analyst view, I would hide less useful fields such as punctuation counts while keeping the original logs for investigation. Repeated alerts should be grouped for readability without deleting the underlying evidence.

My next changes would be to:

- Alert on repeated scans and high-priority exploitation signatures.
- Correlate Snort events with pfSense firewall actions and host logs.
- Build a dashboard showing sources, targets, signatures, and activity over time.
- Review DMZ-to-LAN rules and permit only required services.

## What I learned

Collecting logs is only the first step. I could see traces of the simulation in Splunk, but identifying the source, timing, and meaning of each alert took more work. Clear fields, consistent timestamps, and correlation would make those events much easier to investigate.

**Tools:** pfSense · Snort · Kiwi Syslog · Splunk Enterprise · Nmap / Zenmap · Infection Monkey
