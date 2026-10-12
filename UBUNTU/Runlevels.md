---
Title: fstab
Type:
Topic:
tags:
cover: "[[/IMAGES/runlevel.jpeg]]"
---
# Runlevels
### What Are Runlevels?

A **runlevel** defines the state of the machine.

It determines:
- Which services start
- Whether networking runs
- Whether GUI loads
- Whether system is in rescue mode
- Whether system is shutting down

Think of runlevel as:  “What mode should Linux operate in right now?”

### Old Linux Boot System (SysV Init)

Before systemd, Linux used: `init (PID 1)`
Check it: `ps -p 1`
If it shows `init` → old system  
If it shows `systemd` → modern system (most likely)

### Traditional Runlevels (SysV)

|Runlevel|Meaning|
|---|---|
|0|Shutdown|
|1|Single user mode (Rescue mode)|
|2|Multi-user (no network - varies by distro)|
|3|Multi-user with networking (no GUI)|
|4|Unused (custom)|
|5|Multi-user with GUI|
|6|Reboot|

#### Example (Important for Interviews)

##### Runlevel 3
- Multi-user
- Networking enabled
- No graphical interface
- Used in servers

##### Runlevel 5
- Everything in runlevel 3
- GUI (GNOME, KDE)

### How Runlevels Actually Execute Services

This is where it gets interesting.
When system boots:
1. BIOS/UEFI loads bootloader (GRUB)
2. GRUB loads kernel
3. Kernel starts `/sbin/init`
4. Init checks default runlevel
5. Init executes scripts for that runlevel

#### Where Are These Scripts?

Inside: `/etc/rc.d/ /etc/rc3.d/ /etc/rc5.d/`
For example: `/etc/rc3.d/`
You’ll see something like: `S10network S20sshd K10bluetooth`

##### What do these mean?
- `S` → Start service
- `K` → Kill (stop) service
- Numbers → Order of execution

Example: `S10network → starts first S20sshd → starts after network`
Execution is sequential.
### Changing Runlevels (Old Way)

Temporarily: `init 3` or `telinit 3`
Set default runlevel:
Edit: `/etc/inittab`
Find: `id:3:initdefault:`

### Modern Linux Uses systemd (No More Runlevels)

Most modern distros (Ubuntu, RHEL 7+, CentOS 7+, Debian 8+) use: `systemd`
Runlevels are replaced by: Targets

### systemd Targets (Modern Equivalent)

|Old Runlevel|systemd Target|
|---|---|
|0|poweroff.target|
|1|rescue.target|
|3|multi-user.target|
|5|graphical.target|
|6|reboot.target|

Check Current Target : `systemctl get-default`
Check Current State : `systemctl list-units --type=target`
**Change Default Target**
Set CLI mode (like runlevel 3): `sudo systemctl set-default multi-user.target`
Set GUI mode (like runlevel 5): `sudo systemctl set-default graphical.target`

### How systemd Executes Services

Now this is important for DevOps.
Instead of scripts in `/etc/rc.d/`, we use: `/usr/lib/systemd/system/`
Each service has a `.service` file.
Example: `sshd.service docker.service nginx.service`

### Example: sshd.service

It contains:

```
[Unit] 
Description=OpenSSH server daemon 
After=network.target  
[Service] 
ExecStart=/usr/sbin/sshd -D  
[Install] 
WantedBy=multi-user.target
```

Important part: `WantedBy=multi-user.target`
That means: When system enters multi-user mode, start this service.

### Boot Process Flow (Modern systemd)

```
Power ON  
	↓
  GRUB
	↓
Kernel
	↓
systemd (PID 1)
	↓
Default Target (multi-user / graphical)  
	↓ 
Starts required services based on dependencies
```

systemd does:
- Parallel service startup
- Dependency-based startup
- Faster boot
- Better logging (journalctl)

### Real World Server Example

Let’s say you manage a production server.

You don’t want GUI.  
You only want:
- Networking
- SSH
- Docker
- Nginx

So default target: `multi-user.target`
That’s basically modern runlevel 3.
### Rescue Mode (Important for IT Guys)

If system breaks:
Boot into: `rescue.target`
Equivalent of runlevel 1.
Used for:
- Reset root password
- Fix filesystem
- Remove broken packages

### Key Difference: SysV vs systemd

| SysV               | systemd            |
| ------------------ | ------------------ |
| Sequential startup | Parallel startup   |
| Shell scripts      | Unit files         |
| Runlevels          | Targets            |
| Slower             | Faster             |
| Simple but limited | Powerful & complex |

### How To Check What Your System Uses

`ps -p 1`
If output: `1 systemd` → Modern
If: `1 init` → Old SysV
