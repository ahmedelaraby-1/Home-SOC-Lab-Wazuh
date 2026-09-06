| MINI SOC LAB |

# SSH Brute-Force Detection — Manual Approach
### Incident Report

*Detection and response using a custom Bash script and Fail2ban*

| Project | Mini SOC Lab |
|---|---|
| Scenario | SSH Brute-Force Detection & Automated Response |
| Status | Attack detected, IP identified, and connection blocked successfully |
| Test date | 3 August 2026 |

---

## 1. Executive Summary

A controlled SSH brute-force attack was launched from a Kali Linux machine against an Ubuntu target on the same local network. Failed authentication attempts were monitored using a custom Bash script that parses `sshd` journal logs, and blocked automatically using Fail2ban.

The attack and the response were confirmed through:
- **158 failed SSH authentication attempts** recorded in `journalctl`/`auth.log`, all originating from the same source IP
- **Fail2ban `sshd` jail** banning the attacker's IP after the configured failed-attempt threshold was reached
- **A follow-up SSH connection attempt from the banned IP being refused**, confirming the block was active

This exercise validates that suspicious SSH activity can be detected and mitigated without a dedicated SIEM, using native Linux logging and a lightweight open-source tool.

## 2. Lab Environment

| Component | Role | IP |
|---|---|---|
| Kali Linux | Attack simulation / SSH brute-force source | 192.168.1.4 |
| Ubuntu | Target / SSH server under monitoring | 192.168.1.3 |

Both machines were on the same local network, connected via SSH (port 22).

## 3. Objective

- Detect a real SSH brute-force attack using only native logs (no SIEM).
- Build a script to summarize failed login attempts and identify the offending IP.
- Configure Fail2ban to automatically ban an IP after repeated failed attempts.
- Confirm the ban is effective by attempting a real connection from the blocked IP.

## 4. Attack Simulation

Hydra was used from the Kali machine to run a dictionary attack against a single known user account on the Ubuntu target, using the `rockyou.txt` wordlist.

```
hydra -l a-a -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.3
```

- **Target**: 192.168.1.3:22 (SSH)
- **Source**: 192.168.1.4 (Kali Linux)
- **Method**: Password guessing against a single, known username
- **Tool**: Hydra

## 5. Detection — Custom Script

A Bash script (`ssh_monitor.sh`) was written to query the `sshd` service log directly and summarize brute-force activity:

```bash
#!/bin/bash

echo "=== Failed SSH Login Attempts ==="
journalctl -u ssh.service | grep "Failed password" | wc -l

echo "=== IPs that tried to login ==="
journalctl -u ssh.service | grep "Failed password" | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
```

Running the script during the attack produced:

```
=== Failed SSH Login Attempts ===
158
=== IPs that tried to login ===
    158 192.168.1.4
```

This confirmed 158 failed login attempts, all from the same source IP (192.168.1.4), matching the Kali attack machine — consistent with a brute-force pattern rather than legitimate user error.

Raw `auth.log` / `journalctl` entries corroborated this, showing rapid, repeated `Failed password for a-a from 192.168.1.4` entries a few milliseconds apart.

## 6. Automated Response — Fail2ban

Fail2ban was configured to monitor the `sshd` jail and ban any IP reaching **5 failed login attempts**.

When Hydra was run again, Fail2ban's status confirmed the ban was triggered automatically:

```
$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed:  1
|  |- Total failed:      28
|  `- Journal matches:   _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned:  1
   |- Total banned:      2
   `- Banned IP list:    192.168.1.4
```

## 7. Verification

To confirm the ban was actually enforced (not just logged), a real SSH connection attempt was made from the banned IP:

```
$ ssh a-a@192.168.1.3
ssh: connect to host 192.168.1.3 port 22: Connection refused
```

The connection was refused, confirming Fail2ban was actively blocking the attacker's IP at the network level.

## 8. Incident Timeline

1. Hydra brute-force attack launched from Kali (192.168.1.4) against Ubuntu (192.168.1.3) on SSH.
2. `sshd` logged repeated `Failed password` entries for user `a-a` from 192.168.1.4.
3. Custom Bash script confirmed 158 failed attempts, all from a single IP.
4. Fail2ban's `sshd` jail reached the 5-failed-attempt threshold and banned 192.168.1.4.
5. A second Hydra run / manual SSH attempt from the banned IP was refused, confirming the block.

## 9. MITRE ATT&CK Mapping

| Tactic | Technique | ID | Relevance |
|---|---|---|---|
| Credential Access | Brute Force: Password Guessing | T1110.001 | Repeated password attempts against a single known SSH user using a wordlist |

## 10. Response / Containment

- The attacker's IP (192.168.1.4) was automatically banned by Fail2ban after reaching the failed-attempt threshold.
- The ban was verified to be effective at the connection level (refused, not just logged).
- In a production environment, the same detection logic (repeated failed logins from one source) would also warrant reviewing whether any attempt succeeded, and rotating credentials for the targeted account as a precaution.

## 11. Conclusion

This exercise demonstrated that an SSH brute-force attack can be detected and mitigated using only native Linux tooling — no dedicated SIEM required. A simple log-parsing script was enough to confirm the attack pattern, and Fail2ban provided automated, verified containment.

**Stage completed:** Brute Force → Detected via native logs → Automatically blocked via Fail2ban → Block verified

---

*This report is Part 1 of a two-part comparison. Part 2 repeats the same attack scenario, detected and analyzed using Wazuh (SIEM) instead of manual log parsing.*
