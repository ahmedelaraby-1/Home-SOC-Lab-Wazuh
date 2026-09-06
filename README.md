# SSH Brute-Force Detection: Manual vs SIEM Approach

## Objective
Before relying on a SIEM to do the work for me, I wanted to actually understand how detection works under the hood — to be the one hunting for the raw logs myself, instead of a tool just handing me an alert. So I ran the exact same SSH brute-force attack twice: once detecting and stopping it manually, and once letting Wazuh (a full SIEM) do the detection and classification for me. The goal was to compare both approaches and understand what a SIEM is actually doing for you under the surface.

## Environment
- **Target**: Ubuntu (two separate hosts — one for each phase)
- **Attacker**: Kali Linux
- Both machines were on the same network, connected via SSH.

## Tools Used
- Hydra
- Bash scripting
- Fail2ban
- Wazuh (SIEM)

## Part 1: Manual Detection & Response
I started with `/var/log/auth.log` (via `journalctl`) as my only source of truth. I wrote a script to filter the raw logs down to failed SSH login attempts, count how many times each IP repeated, and pull out the source IP — basically doing my own log parsing, filtering, and normalization by hand with manual commands, no tool doing it for me.

Once I could see the pattern clearly (158 failed attempts from a single IP), I brought in Fail2ban to act on it automatically: it watches the same logs and bans an IP after 5 failed attempts. I confirmed the ban was real — not just logged — by trying to SSH in from the banned IP and getting "Connection refused."

📄 Full report: [01_SSH_BruteForce_Manual_Detection.md](./01_SSH_BruteForce_Manual_Detection.md)

## Part 2: SIEM-Based Detection
Next, I ran the identical Hydra attack against a second Ubuntu host, this time monitored by a Wazuh agent. Instead of me writing a script to find and count the failed logins, Wazuh did that automatically — decoding the raw PAM/sshd logs, correlating repeated failures into a single higher-severity alert (Rule 5551, Level 10), and even classifying the activity under MITRE ATT&CK (Password Guessing / Brute Force / SSH) without me tagging anything myself.

📄 Full report: [02_SSH_BruteForce_Wazuh_Detection.md](./02_SSH_BruteForce_Wazuh_Detection.md)

## Comparison

| | Manual Approach | Wazuh (SIEM) Approach |
|---|---|---|
| Log source | Native `auth.log` / `journalctl` | Wazuh agent + decoders |
| Filtering & normalization | Manual, via a custom script | Automatic, via Wazuh's PAM/syslog decoders |
| Severity scoring | None — just a raw count | Automatic (Level 5 → Level 10 escalation on pattern match) |
| Correlation (single failure vs. attack pattern) | Manual logic in the script | Built-in correlation rule (5551) |
| MITRE ATT&CK mapping | Done manually, after the fact | Generated automatically by Wazuh |
| Response | Fail2ban, configured and verified manually | Detection only in this exercise (active response not configured) |
| Effort required | Higher — I had to build every piece | Lower — the SIEM did the heavy lifting |

## What I Learned
Doing it manually first changed how I think about what a SIEM is actually for. Writing the script myself taught me what "detection" really means at the log level — pulling raw entries, filtering the noise, extracting the fields that matter, and normalizing them into something you can actually count and act on. None of that is magic; it's exactly what a tool like Wazuh is doing internally, just automated and scaled.

That's what stood out most once I moved to Wazuh: it wasn't doing anything conceptually different from my script, it was doing the same job — decode, filter, correlate, score — but instantly, across multiple agents, with severity levels and MITRE mapping built in. Understanding the manual side first made the value of the SIEM click in a way it wouldn't have if I'd just started with Wazuh from day one.
