| MINI SOC LAB |

# SSH Brute-Force Detection — SIEM Approach
### Incident Report

*Detection and analysis using Wazuh (SIEM)*

| Project | Mini SOC Lab |
|---|---|
| Scenario | SSH Brute-Force Detection via Wazuh |
| Status | Detection successfully validated |
| Test date | 5–6 September 2026 |

---

## 1. Executive Summary

The same SSH brute-force scenario used in the manual detection exercise (Part 1) was repeated against a second Ubuntu host, this time monitored by a Wazuh manager and agent instead of native logs and Fail2ban. The attack, launched from Kali Linux using Hydra against a known user account, was fully captured and classified by Wazuh.

Over the monitored period, Wazuh recorded:
- **501 total alerts** related to the activity
- **347 authentication failure events**
- **3 authentication success events**
- Repeated **Rule 5503** ("PAM: User login failed") and **Rule 5551** ("PAM: Multiple failed logins in a small period of time") firings
- A **Rule 2502** event ("syslog: User missed the password more than one time") at Level 10
- Native MITRE ATT&CK tagging by Wazuh itself, showing **Password Guessing**, **Brute Force**, and **SSH** as the top classifications for the activity

This confirms that Wazuh detected and correctly classified the same brute-force pattern identified manually in Part 1 — without needing a custom script or manual log parsing.

## 2. Lab Environment

| Component | Role | IP |
|---|---|---|
| Kali Linux | Attack simulation / SSH brute-force source | 192.168.1.4 |
| Ubuntu (agent) | Target / SSH server, monitored by Wazuh | 192.168.1.16 |
| Wazuh | SIEM / detection and alerting (manager + agent: `a-a-VirtualBox`) | Wazuh server in lab |

## 3. Objective

- Repeat the same SSH brute-force attack used in the manual (Part 1) exercise, this time against a Wazuh-monitored host.
- Validate that Wazuh natively detects repeated SSH authentication failures without any custom scripting.
- Confirm that Wazuh correctly maps the activity to MITRE ATT&CK.
- Compare the detection depth and effort against the manual approach in Part 1.

## 4. Attack Simulation

The same Hydra command and target user account used in Part 1 were reused, this time pointed at the Wazuh-monitored Ubuntu host:

```
hydra -l a-a -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.16
```

- **Target**: 192.168.1.16:22 (SSH), agent name `a-a-VirtualBox`
- **Source**: 192.168.1.4 (Kali Linux)
- **Method**: Password guessing against a single, known username
- **Tool**: Hydra

## 5. Detection & Telemetry (Wazuh)

| Rule ID | Rule Description | Level |
|---|---|---|
| 5503 | PAM: User login failed | 5 |
| 5551 | PAM: Multiple failed logins in a small period of time | 10 |
| 2502 | syslog: User missed the password more than one time | 10 |

A sample raw log line captured by Wazuh (decoder: `pam`, source `journald`):

```
Sep 05 21:37:29 a-a-VirtualBox sshd[7505]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.1.4  user=a-a
```

Wazuh correctly extracted the structured fields from this raw log, including:
- `data.srcip`: 192.168.1.4 (the Kali attacker)
- `data.dstuser`: a-a (the targeted account)
- `data.tty`: ssh
- `rule.description`: "PAM: Multiple failed logins in a small period of time"
- `rule.firedtimes`: 5, `rule.frequency`: 8 (for this correlated alert)

## 6. Detection Summary Dashboard

Wazuh's own dashboard summarized the attack window (last 24 hours) as follows:

- **501** total alerts
- **0** alerts at Level 12 or above (no critical escalation)
- **347** authentication failure events
- **3** authentication success events
- Alert levels observed: 3, 5, 7, 8, 9, and 10 — with a sharp spike of Level 10 events during the attack burst
- Top 5 agents: all activity attributed to the single monitored agent, `a-a-VirtualBox`

Wazuh's built-in **Top 10 MITRE ATT&CK** view for this activity automatically classified the events under:
- Password Guessing
- SSH
- Brute Force
- Valid Accounts
- Sudo and Sudo Caching

This mapping was generated automatically by Wazuh's rule engine, without any manual MITRE tagging.

## 7. Incident Timeline

1. Hydra brute-force attack launched from Kali (192.168.1.4) against the Wazuh-monitored Ubuntu host (192.168.1.16), targeting user `a-a`.
2. PAM/sshd authentication failures began streaming into Wazuh via the agent, each matched to **Rule 5503** (Level 5).
3. As failures repeated within a short window, Wazuh escalated matching events to **Rule 5551** (Level 10) — "Multiple failed logins in a small period of time."
4. A related **Rule 2502** (Level 10) alert fired for repeated password misses at the syslog level.
5. Across the attack window, Wazuh logged 501 total alerts, with 347 classified as authentication failures and 3 as authentication successes.
6. Wazuh's dashboard automatically tagged the activity under the Password Guessing / Brute Force / SSH MITRE ATT&CK categories.

## 8. Detection Analysis

Unlike the manual approach in Part 1 — which required a custom script to count failed attempts per IP and Fail2ban to act on them — Wazuh detected, classified, and severity-scored the same activity out of the box, using its built-in PAM and syslog decoders and correlation rules (5503, 5551, 2502). The correlation rule (5551) in particular demonstrates SIEM-style behavior: it doesn't just log a single failure, it recognizes the *pattern* of repeated failures in a short time window and raises the severity accordingly (Level 10 vs Level 5 for a single failure).

The presence of 3 "authentication success" events alongside the failures is also notable: in a real incident, these would need immediate follow-up to confirm whether the brute-force attempt eventually succeeded.

## 9. MITRE ATT&CK Mapping

| Tactic | Technique | ID | Relevance |
|---|---|---|---|
| Credential Access | Brute Force: Password Guessing | T1110.001 | Repeated password attempts against a single known SSH user using a wordlist; independently confirmed by Wazuh's own MITRE classification |

## 10. Response / Containment

- The activity was confirmed and scored automatically by Wazuh; no automated blocking (e.g. Fail2ban) was configured on this host, since the focus of this exercise was detection, not response.
- In a production deployment, Wazuh's active response module could be configured to automatically block an IP once Rule 5551 or similar correlation rules fire — mirroring the Fail2ban behavior from Part 1, but triggered from within the SIEM itself.
- The 3 authentication success events should be investigated first in any real incident, to rule out a successful compromise.

## 11. Conclusion

This exercise repeated the exact same SSH brute-force scenario as Part 1, but replaced manual log parsing and Fail2ban with Wazuh as a centralized SIEM. Wazuh detected the same underlying pattern — and more: it automatically classified severity levels, correlated repeated failures into a single higher-severity alert (Rule 5551), and mapped the activity to MITRE ATT&CK without any manual tagging.

**Stage completed:** Brute Force → Detected and classified natively by Wazuh (Rules 5503 / 5551 / 2502) → Automatically mapped to MITRE ATT&CK

---

*This report is Part 2 of a two-part comparison. Part 1 covers the same attack scenario, detected manually via a custom Bash script and Fail2ban.*
