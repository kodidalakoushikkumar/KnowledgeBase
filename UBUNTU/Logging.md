# 3️⃣3️⃣ Logging
## How Linux Logging Actually Works (Core Knowledge)

Linux logging is NOT one thing.  It is a **pipeline**: `Application → syscalls → systemd-journald → rsyslog/syslog-ng → log files → rotation → archive → SIEM`
### 1. journald (The Heart of Modern Linux Logs)

Location:   `/var/log/journal/`
View logs:   `journalctl`
Real-time:  `journalctl -f`
Boot logs:  `journalctl -b`
Last 1 hour:  `journalctl --since "1 hour ago"`
By service:  `journalctl -u ssh journalctl -u nginx journalctl -u docker`
By PID:  `journalctl _PID=1234`
By user:  `journalctl _UID=1000`
#### Why journald is powerful

Unlike plain files, journald logs contain metadata:
- PID
- user
- command
- cgroup
- container id
- boot id
- SELinux context

This makes hiding actions VERY hard.
### 2. Classic Log Files (rsyslog)

Location:  `/var/log/`, Important ones:

|File|Meaning|
|---|---|
|auth.log / secure|logins, sudo, ssh|
|syslog / messages|system events|
|kern.log|kernel activity|
|dmesg|hardware|
|boot.log|startup|
|apt/history.log|installed packages|

Example:  `sudo cat /var/log/auth.log`
## PART 2 — The Skill of a Real Log Expert (What companies want)

### Skill 1: Investigate a Login Attack

Find failed SSH attempts:   `grep "Failed password" /var/log/auth.log`
Count attackers:   `grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -nr`
Find successful logins:   `grep "Accepted password" /var/log/auth.log`
### Skill 2: Timeline Reconstruction (VERY IMPORTANT)

Security engineers reconstruct incidents like this:  `journalctl --since "2026-02-15 10:00" --until "2026-02-15 11:00"`
Find what a user executed:  `journalctl _UID=1000 | grep sudo`
Find suspicious commands:  `grep "COMMAND=" /var/log/auth.log`
