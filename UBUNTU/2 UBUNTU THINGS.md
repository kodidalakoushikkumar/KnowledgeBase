# 1️⃣ Do nothing when closing the lid on UBUNTU


➖ sudo edit /etc/systemd/logind.conf
➖ If HandleLidSwitch is not set to ignore then change it:
➖ HandleLidSwitch=ignore
➖ sudo systemctl restart systemd-logind

# 2️⃣ File Share on UBUNTU using samba

```
sudo apt update
sudo apt install samba 
mkdir -p ~/SharedFolder
chmod 777 ~/SharedFolder

sudo nano /etc/samba/smb.conf

[Public]
path = /srv/public
browseable = yes
read only = no
writable = yes
guest ok = yes
guest only = yes
create mask = 0777
directory mask = 0777
force user = nobody

sudo systemctl restart smbd
sudo ufw allow samba
sudo smbpasswd -a username(prefer local user)
sudo smbpasswd -e kodidala.kumar
sudo systemctl restart smbd
```

# 3️⃣ KEYRING ISSUE

**Ubuntu Keyring is like a secure password vault for your applications**.
When you log in to websites or apps such as **Chrome, Slack, Wi-Fi, VPN, etc.**, Ubuntu can store passwords, tokens, and other secrets inside the **GNOME Keyring** instead of leaving them as normal files.
Example, Suppose you log in to Slack:

1. You enter your Slack username/password.
2. Slack receives an authentication token.
3. That token may be stored securely in the **GNOME Keyring**.
4. Next time you open Slack, it tries to read that token automatically.
5. If the keyring is **locked**, corrupted, or its password doesn't match your Ubuntu login password, you may get a popup asking: **Enter password to unlock your login keyring**

#### Option 1: Remove the keyring password (Recommended if you don't need encryption)

This keeps the keyring but disables its password.

1. Install Seahorse if you don't have it: `sudo apt install seahorse`
2. Open it: `seahorse`
3. Under **Passwords → Login**:
    - Right-click **Login**
    - Select **Change Password**
    - Enter your current keyring password.
    - Leave the **New Password** and **Confirm Password** fields **blank**.
    - Accept the warning.

The keyring will still exist, but it won't prompt you anymore.
#### Option 2: Delete the keyring completely

This removes all stored passwords (Wi-Fi, VPN, browser secrets, Git credentials, etc.).

```
rm -rf ~/.local/share/keyrings
```

Log out and log back in. If an application needs a keyring later, Ubuntu may recreate it.

#### Option 3: Disable the GNOME Keyring daemon (Most aggressive)

This prevents the keyring service from starting.

```
systemctl --user mask gnome-keyring-daemon.service
systemctl --user mask gnome-keyring-daemon.socket
```

You can also mask related sockets:

```
systemctl --user mask gnome-keyring-daemon-secrets.socket
systemctl --user mask gnome-keyring-daemon-pkcs11.socket
systemctl --user mask gnome-keyring-daemon-ssh.socket
```

Then log out and log back in.
To undo:

```
systemctl --user unmask gnome-keyring-daemon.service
systemctl --user unmask gnome-keyring-daemon.socket
systemctl --user unmask gnome-keyring-daemon-secrets.socket
systemctl --user unmask gnome-keyring-daemon-pkcs11.socket
systemctl --user unmask gnome-keyring-daemon-ssh.socket
```

#### Option 4: Uninstall it

```
sudo apt purge gnome-keyring seahorse
sudo apt autoremove
```

**Warning:** Some desktop environments and applications expect GNOME Keyring to be present. Removing it may cause:

- Saved Wi-Fi passwords to stop working.
- Browsers to stop securely storing passwords.
- VPN or SSH credentials to require manual entry.


# 4️⃣ AD Realm Config and Kerberos Details

#### The connection between your Ubuntu laptops and Active Directory is handled by three main components:

- **SSSD (System Security Services Daemon)**: This is the **client software** managing the identity and authentication against your Active Directory domain. It acts as the local broker, communicating directly with AD Domain Controllers using LDAP (to look up user/group information) and Kerberos (to authenticate credentials).
- **PAM (Pluggable Authentication Modules)**: Specifically integrated via **libpam-sss**. PAM is the framework in Ubuntu that handles authentication tasks (like logging in, checking passwords, or changing passwords). The SSSD PAM module routes local login or password-change requests to SSSD, which in turn updates Active Directory.
- **NSS (Name Service Switch)**: Specifically integrated via **libnss-sss**. This component tells Ubuntu how to resolve user and group names. It ensures that AD users (such as members of the **Domain Users** group) are recognized as valid system users by local Linux commands and permissions.

#### How Your Joining Script Works

```
#!/bin/bash
set -e  # Exit immediately if a command exits with a non-zero status
sudo apt update
sudo apt install -y realmd sssd sssd-tools adcli oddjob oddjob-mkhomedir
domain_name="impressico.local"
realm discover "$domain_name"
echo -n "Enter username for domain join: "
read domain_user
sudo realm join --user="$domain_user" "$domain_name"
realm list
CONFIG_FILE="/etc/sssd/sssd.conf"
touch "$CONFIG_FILE"
cp "$CONFIG_FILE" "${CONFIG_FILE}.bak"
if grep -q "^use_fully_qualified_names" "$CONFIG_FILE"; then
    sed -i 's/^use_fully_qualified_names.*/use_fully_qualified_names = False/' "$CONFIG_FILE"
else
    echo "use_fully_qualified_names = False" >> "$CONFIG_FILE"
fi

if grep -q "^fallback_homedir" "$CONFIG_FILE"; then
    sed -i 's|^fallback_homedir.*|fallback_homedir = /home/%u|' "$CONFIG_FILE"
else
    echo "fallback_homedir = /home/%u" >> "$CONFIG_FILE"
fi
chmod 600 "$CONFIG_FILE"
chown root:root "$CONFIG_FILE"
sudo systemctl restart sssd
sudo systemctl enable sssd
sudo pam-auth-update --enable mkhomedir
sudo realm permit --groups "Domain Users"
echo "Domain join and configuration completed successfully."
```

Based on your system configuration, the domain-joining script you ran automated several key steps:

1. **Installed Required Dependencies**: It installed packages like `sssd`, `sssd-tools`, `libpam-sss`, `libnss-sss`, `adcli`, and `samba-common-bin`.
2. **Discovered and Joined the Realm**: It performed a DNS lookup to find the Domain Controllers for `impressico.local` and ran `realm join` or `adcli join`. This created a computer account for your laptop in Active Directory, establishing a secure trust relationship and generating a local keytab file (usually `/etc/krb5.keytab`) for machine-level authentication.
3. **Configured SSSD**: It created SSSD's configuration (usually in `/etc/sssd/sssd.conf`) specifying `active-directory` as the backend, defining the login format (`%U`), and enforcing access control such as restricting login to the **Domain Users** group.
4. **Enabled SSSD in PAM and NSS**: It updated `/etc/nsswitch.conf` and updated the PAM configuration (usually via `pam-auth-update`) so that Ubuntu knows to consult SSSD for user details and authentication.

#### What `kinit` and `kpasswd` Do

- **kinit** **(Kerberos Initialize)**: This tool is used to manually authenticate against the Kerberos Key Distribution Center (KDC, which is hosted on your AD Domain Controller). When you run `kinit`, it prompts for your AD password, contacts AD, and obtains a **Ticket-Granting Ticket (TGT)**. This ticket is stored in a local cache (usually in `/tmp`) and proves your identity to Kerberos-aware services without requiring you to re-enter your password.
- **kpasswd** **(Kerberos Password)**: This is a utility specifically designed to change a user's Kerberos password directly on the KDC. Because you have a valid TGT (obtained via `kinit`), `kpasswd` uses that ticket to securely talk to the Kerberos Password Change Service (port 464) on your AD Domain Controller and update your password.

#### Why `passwd` Previously Worked vs. Why It Stopped Working

How it worked before:

When SSSD is functioning correctly, running the standard **passwd** command invokes the PAM stack. Because SSSD is integrated via `libpam-sss`, PAM automatically routes the password-change request to SSSD (`pam_sss.so`). SSSD securely contacts the Domain Controller and handles the password update in Active Directory behind the scenes. **Users do not need to run** **kinit** **manually** because SSSD handles the secure connection on behalf of the system using its machine account trust or by leveraging your credentials during the interactive `passwd` session.

#### Why it stopped working:

Since `kinit` followed by `kpasswd` successfully changes the password, your network connection to the domain controller, Kerberos DNS resolution, and AD password policies are working perfectly. The failure of `passwd` means the **SSSD-to-PAM integration is broken**. This typically happens due to:

1. **SSSD in Offline Mode**: SSSD caches credentials so users can log in when disconnected from the office. If SSSD thinks it is offline (due to a temporary network blip or startup issue), it will refuse to contact AD for password changes. When SSSD is offline, the PAM-based `passwd` command will fail, whereas direct Kerberos tools like `kinit` bypass SSSD's offline state entirely to talk directly to AD.
2. **PAM Configuration Desynchronization**: If an OS update or manual configuration change altered `/etc/pam.d/common-password` or disabled SSSD, the `passwd` command will no longer hand off the request to SSSD.
3. **SSSD Cache Issues**: The local cache database (`/var/lib/sss/db/`) can sometimes become corrupted, causing SSSD to fail to update credentials even if the network is online.
4. **Machine Account Trust Out-of-Sync**: Your Ubuntu machine's computer account password in Active Directory may have expired or been disabled in AD, breaking SSSD's secure system-level connection to the Domain Controller.

#### How to Troubleshoot and Restore `passwd`

To fix the standard `passwd` integration so users do not have to use `kinit` and `kpasswd`, try these steps on an affected machine:

1. **Clear the SSSD Cache**: Corrupted caches are a frequent culprit. You can safely clear the cache and restart SSSD by running:

```
sudo sss_cache -E
sudo systemctl restart sssd
```

1. **Verify SSSD is Enabled in PAM**: Ensure `pam_sss.so` is active in your PAM configuration. You can re-enable and configure it automatically by running: `sudo pam-auth-update`

Ensure that **SSS authentication** is checked in the menu and select **OK**.

1. **Check if SSSD is Online**: Run `sssctl domain-status impressico.local` to verify if SSSD is in "online" or "offline" mode. If it is offline, check your SSSD logs (`/var/log/sssd/sssd_impressico.local.log`) for LDAP connection errors.
2. **Test Machine Account Trust**: Check if the laptop's computer account is still trusted by AD: `sudo adcli testjoin`

If this fails, you may need to rejoin the domain using your script or by running `sudo realm join impressico.local` to renew the machine secrets.

# 5️⃣ Configure Virtual Memory

#### Option 1: Create a Swap File

➖ Check existing swap `swapon --show` , If nothing shows under Swap, you’re good to proceed.
➖ Create a swap file `sudo fallocate -l 4G /swapfile`
➖ Set correct permissions (important) `sudo chmod 600 /swapfile`
➖ Format the file as swap `sudo mkswap /swapfile`
➖ Enable swap `sudo swapon /swapfile`
➖ Verify `swapon --show`
➖ Make swap permanent (very important) Edit /etc/fstab: `sudo nano /etc/fstab`
➖ Add this line at the bottom: `/swapfile none swap sw 0 0` Save and exit.

#### Option 2: Resize Swap File (if already exists)

➖ Disable swap `sudo swapoff /swapfile`
➖  Recreate with new size `sudo fallocate -l 8G /swapfile`
➖ `sudo chmod 600 /swapfile`
➖ `sudo mkswap /swapfile`
➖ `sudo swapon /swapfile`


# 7️⃣ Loop Devices

Usually, the Linux operating system only knows how to read filesystems from a physical storage device (like a physical hard drive or a USB stick). If you want to read files, Linux must mount that physical device.
But what if you have a **single, regular file** on your computer that contains an entire filesystem inside it—such as a downloaded DVD image (`.iso`) or a virtual machine disk image?
Linux cannot mount a plain file directly. It needs a bridge.
This is where a loop device (/dev/loopX) comes in:

1. You point a loop device at your file.
2. The loop device acts as a translator, telling Linux: _"Hey, treat this file as if it were a physical block device (a hard drive)!"_
3. Linux is fooled, maps the "fake disk," and lets you mount it to access the files inside.

#### The "Nesting Doll" Problem

To your main operating system, a `.iso` or `.snap` file is just a giant, solid block of binary data. The operating system's filesystem (like ext4) looks at it and says: _"This is just a single file named_ _ubuntu.iso_ _that is 3 Gigabytes big. I have no idea what is inside it; to me, it's just raw bytes."_
However, that `.iso` or `.snap` file actually contains an **entire secondary filesystem** inside of it. It has its own folder structure (`/bin`, `/lib`, `/usr`), its own files, and its own system permissions.
To read or run anything inside that file, Linux has two choices:

#### Choice A: Extract it (The Slow, Wasteful Way)

Linux could unzip/extract the entire 3GB file onto your hard drive.

- **The problem:** This takes a lot of time, duplicates your data, and wastes massive amounts of disk space. For Snap apps, you would have to extract every app to run it, which defeats the purpose of keeping them compressed.

#### Choice B: Mount it (The Loop Device Way)

Instead of extracting it, we want to read it "live" while it stays compressed inside that single file.
But there is a catch: the Linux kernel's filesystem drivers **cannot talk to plain files**. They are programmed to talk only to **block devices** (hardware like hard drives or USB sticks). They expect to say: _"Give me block number 500 of the hardware drive."_ They do not know how to say: _"Give me byte 10,000 of the file inside the Downloads folder."_
The **loop device** acts as a virtual hardware emulator:

1. It plugs into the `.iso` or `.snap` file.
2. It presents itself to the Linux kernel as a **fake physical hard drive** (e.g., `/dev/loop0`).
3. When the system wants to read a file inside the Snap, it asks the loop device.
4. The loop device instantly translates that request and pulls the correct bytes out of the file on your disk.

#### A Simple Analogy: The Virtual CD Drive
Think of a loop device as the software equivalent of **Daemon Tools** (the virtual CD-ROM drive software from the 2000s).
If you downloaded a DVD image (`.iso`) to your computer back then, you couldn't just double-click it to play the game or movie; your computer expected a physical DVD drive. You had to use software to create a "Virtual DVD Drive" and mount the `.iso` file into it.

- The `.iso` file is the **virtual disc**.
- The loop device (`/dev/loop0`) is the **virtual DVD drive**.
- Mounting it lets the computer read the disc as if you just slid physical plastic into a drive.

By using loop devices, Ubuntu can keep Snap applications compressed in a single file to save space, but still mount them instantly and run them as if they were installed on their own tiny, dedicated hard drives.

Instead of the normal path:  **Physical Disk** (`sda`) ➔ **Filesystem** (your files)
You get the loop path:  **A File on your disk** ➔ **Loop Device** (`/dev/loop0`) ➔ **Filesystem**
Why is Your Screen Flooded with Loop Devices?
If you run the `lsblk` command on Ubuntu, you will likely see a long list of loop devices (`loop0`, `loop1`, `loop2`, etc.). This is because of **Snap packages**.
Ubuntu uses Snap packages by default to install applications.

- Every Snap package you install is actually a **compressed filesystem image** (a `.snap` file).
- To run the application, Ubuntu must mount that `.snap` file.
- To mount it, Ubuntu has to assign **one loop device to every single Snap package**.

So, if you have 10 Snap applications installed, you will have 10 loop devices running.
Three Things That Make Loop Devices Less Confusing

1. **They do not waste your disk space:** Because they are just pointers to files that already exist on your drive, they do not repeatedly consume physical storage space.
2. **They are completely safe and read-only:** They are just mounting background applications. However, you should never try to manually delete an active loop device, or you will break the application using it.
3. **They live in** **/dev** **because it's an interface:** The `/dev` folder is not a regular directory on your hard drive; it is a "device interface". Anything that behaves like a device—whether it's a real SSD, a physical USB port, or a fake loop device—must show up there.

**The Golden Rule**
**If a device name starts with** **/dev/loop****, ignore it**. When you are partitioning your drive, installing an operating system, or troubleshooting a disk problem, just look past the loop devices and focus on real disks like `sda` or `nvme0n1`.

# 8️⃣ Snapshot

A **snapshot** is a **read-only picture of a filesystem at a specific moment in time**. It acts as a "time-machine button" for your operating system.
To understand snapshots clearly, it is easiest to start with what they are _not_:

- **Not a full copy of your data**
- **Not a backup** by itself
- **Not another partition**

Instead, a snapshot consists strictly of **metadata and changed blocks of data**.
#### How Snapshots Work: Copy-on-Write (CoW)

The secret to why snapshots are so incredibly fast and space-efficient is a technology called **Copy-on-Write (CoW)**.
When you click "create snapshot," the system **does not copy any of your existing files**. It simply takes a digital note of exactly where your files are currently pointing.

- **Before the snapshot:** Your file (let's call it File A) points to a physical block on your hard drive (Block 1).
- **You create the snapshot:** Nothing happens. No data is duplicated.
- **You modify the file:** Instead of overwriting Block 1, the system writes your _new_ modifications to a brand-new block (Block 2).
- **The result:** Your live filesystem now points to Block 2 (the new, edited version of the file). Meanwhile, your snapshot continues to point to Block 1 (the old version).

Because the system only writes the differences (the changes made _after_ the snapshot was taken), creating a snapshot is **instant** and **uses almost zero extra disk space** until you start modifying files.

#### Snapshots vs. Backups: The "Save Game" Analogy

It is critical to remember that **snapshots are not backups**. The sources offer a simple analogy to keep them straight:

- **Snapshot = A "Save Game"** If you are playing a video game and are about to fight a tough boss, you save your progress. If you die (or if a software update breaks your system), you can instantly reload your save game and pretend it never happened. However, if your console catches fire (physical disk failure), your save game is lost forever.
- **Backup = Copying that save file to a separate USB drive** If your console catches fire, your progress is safe because it is stored on an entirely different physical device.

|Feature|Snapshot|Backup|
|---|---|---|
|**Speed**|⚡ Instant|Slow|
|**Space**|Highly efficient (only stores changes)|Large (full duplicates of files)|
|**Storage Location**|Lives on the **same** physical disk|Lives on a **separate** physical disk|
|**Protects from Disk Failure?**|❌ No (if the drive dies, snapshots die too)|✅ Yes|


#### How Snapshots Save Your Life

Snapshots are ultimate safety nets for three classic administrative emergencies:

1. **Bad System Updates:** You run a system upgrade (`sudo apt upgrade`), but a bug breaks your display drivers. You roll back to your pre-upgrade snapshot and your computer instantly boots up working again.
2. **Accidental Deletions:** You run a destructive command by mistake (like `rm -rf /etc/*` which wipes out system configurations). Rolling back the snapshot restores your configuration files instantly.
3. **Kernel Panics:** A new Linux kernel breaks your system's boot process. You boot into your pre-upgrade snapshot and restore access immediately.

#### The Different Flavors of Snapshots in Linux

Not all Linux snapshots work the same way. Their behavior depends entirely on the technology or filesystem you choose:

- **LVM Snapshots:** Operating at the block device level, these are manually managed by system administrators and servers. _Warning:_ Keeping LVM snapshots active for a long time will slow down your system.
- **ZFS Snapshots:** Built directly into the ZFS filesystem. They are famous because they are instant, cheap, recursive, and support "boot environments" (letting you choose which snapshot version of the OS to boot into).
- **Btrfs Snapshots:** Popular on desktop-focused distributions like Fedora or openSUSE. However, **Ubuntu does not default to Btrfs**.
- **Timeshift:** A popular graphical tool for Ubuntu users. It looks and acts like a snapshot tool, but it actually performs **file-level backups** (usually using rsync) unless it is specifically configured to use Btrfs.

# 9️⃣ LVM

Logical Volume Manager (LVM) is a powerful, flexible storage administration tool in Linux.

#### The Core Problem LVM Solves: Static vs. Dynamic Partitioning

In traditional Windows or legacy Linux disk management, hard drives are carved into **fixed partitions** (like drive `C:` and `D:`).

- **The Problem:** Imagine you allocate 100 GB to your operating system partition (`C:`) and 400 GB to your data partition (`D:`). Over time, your system partition gets completely full, while your data partition is sitting mostly empty.
- **The Traditional Fix:** To take free space from your data partition and give it to the operating system, you have to back up your data, delete or shrink partitions, resize them, and restore your files. This process is slow, tedious, and carries a high risk of data loss.
- **The LVM Solution:** LVM acts as a **flexible software layer** that sits on top of your physical disks, allowing you to dynamically resize disk space at any time without reinstalling your OS or risking your data.

The Movable Walls Analogy

- **Traditional Partitions = Fixed Brick Walls.** If you build a house with brick walls and realize your bedroom is too small, you are stuck. You have to knock down the brick wall and rebuild the house.
- **LVM = Movable Sliding Walls.** You have one large open-space house, and all the interior walls are on sliding tracks. If your bedroom needs more space, you simply slide the wall into the hallway—no demolition, rebuilding, or drama required.
#### The Three Layers of LVM (Components)

LVM organizes physical hardware into a flexible virtual storage pool through three distinct layers:

```
+-------------------------------------------------------+
|              Logical Volumes (LVs)                    |  <- Think "Slices of the cake"
|     [ lv_root (OS) ]          [ lv_home (Data) ]      |     (Acts like C: and D: drives)
+-------------------------------------------------------+
|               Volume Group (VG)                       |  <- Think "The giant virtual cake"
|                 [ vg_main ]                           |     (Combines all physical space)
+-------------------------------------------------------+
|             Physical Volumes (PVs)                    |  <- Think "Raw ingredients"
|       [ /dev/sda ]              [ /dev/sdb1 ]         |     (Real physical hard drives/partitions)
+-------------------------------------------------------+
```

1. **Physical Volume (PV):** The raw physical storage hardware. This can be an entire unpartitioned physical disk (like `/dev/sda`) or an individual partition on a disk. Before LVM can use a disk, it must be initialized as a PV.
2. **Volume Group (VG):** The pooled storage container. You take one or more Physical Volumes (PVs) and combine them together to create a giant virtual storage pool (e.g., combining a 200 GB disk and a 300 GB disk into a single 500 GB Volume Group called `vg_main`).
3. **Logical Volume (LV):** The virtual partitions carved out of your Volume Group. This is what your Linux operating system actually formats with a filesystem and mounts to store files.

Windows vs. LVM Conceptual Comparison

|Windows|Linux (LVM)|How it behaves in practice|
|---|---|---|
|**Physical Disk**|**Physical Volume (PV)**|The raw, physical storage hardware.|
|**Storage Pool**|**Volume Group (VG)**|A unified virtual pool that combines all disk space.|
|**C: or D: Drive**|**Logical Volume (LV)**|Resizable virtual drives carved out of the pooled space.|
#### Step-by-Step: Setting Up LVM From Scratch

The `LVM.md` file outlines a typical administrator workflow for initializing and building an LVM system. Here is how those commands connect logically:

Step A: Preparing and Partitioning a Raw Disk
Before LVM can manage a drive, we often create a partition on it to make PV management easier.

- For example, given a raw, unpartitioned 5GB disk named `/dev/sdb`, you run **cfdisk /dev/sdb** to launch an intuitive console-based partitioning interface.
- You create a partition (e.g., `/dev/sdb1`), select **"write"** to save changes, and exit the utility.
- You can verify the newly created partition structure using **fdisk -l**.

Step B: Initializing Physical Volumes (PV)
To tell LVM that this raw partition is now available for its pool, you initialize it as a physical volume: `sudo pvcreate /dev/sdb1`
This formats the partition metadata so LVM can recognize it. You can inspect the health, size, and details of all PVs on your system using the **pvdisplay** command.

Step C: Creating a Volume Group (VG)
Next, you create the "big cake" virtual pool. You can name it whatever you like (such as `mynew_vg`) and feed it your initialized Physical Volumes:

```
# Pools a single partition
sudo vgcreate mynew_vg /dev/sdb1

# Alternately, pools two separate partitions together at the same time
sudo vgcreate mynew_vg /dev/sdb1 /dev/sdb2
```

- **Scaling Up:** If your Volume Group runs out of space in the future, you can simply plug in a brand-new physical hard drive, initialize it as a PV, and run **vgextend mynew_vg /dev/sdc** to instantly grow your storage pool.
- You can view details of your pools using **vgdisplay**.

Step D: Carving Out Logical Volumes (LV)
Now, you cut "slices of cake" out of your Volume Group to act as partitions. For example, if you want a small 400MB partition and a larger 1GB partition, you run:

```
# Creates vol01 (400MB) from the pool mynew_vg
sudo lvcreate -L 400M -n vol01 mynew_vg

# Creates vol02 (1GB) from the pool mynew_vg
sudo lvcreate -L 1G -n vol02 mynew_vg
```

Use **lvdisplay** to list and verify the logical paths and sizes of these new virtual drives.

Step E: Formatting with a Filesystem (and the `-m` Root safety net)
To store files on a Logical Volume, you must format it with a filesystem using the **mkfs** command: `sudo mkfs.ext4 -m 0 /dev/mynew_vg/vol01`

- **What is the** **-m** **option?** The `-m` flag specifies the percentage of block space reserved strictly for the super-user (root). By default, Linux reserves **5%** of every filesystem's space. Specifying `-m 0` overrides this, giving you 100% of the drive space.
- **Why does Linux reserve space?** This is a critical safety feature for servers. If a partition hits 100% capacity, the operating system can fail to write logs, cannot create temporary system files, and critical services will crash. By reserving 5% of the space, Linux ensures that even if a system partition "fills up" for normal users, **the root administrator can still log in and system services can keep running** to allow for troubleshooting.

Step F: Mounting the Volume
To access the formatted volume, you logically attach it to your directory tree:

1. Create an empty directory to act as your mount point: `sudo mkdir /mnt/vol01`.
2. Mount the volume manually: `sudo mount /dev/mynew_vg/vol01 /mnt/vol01`.
3. To ensure this virtual partition automatically mounts whenever your computer boots up, edit the system's **/etc/fstab** configuration file and add an entry for it.
4. How Linux Mount Points Work vs. Drive Letters

To understand mounting, it is vital to contrast how Linux and Windows look at filesystems:

- **The Windows Way:** Disks are separate and isolated under drive letters (`C:`, `D:`, `E:`).
- **The Linux Way:** Linux does not use drive letters. It has **one single directory tree starting at the root** **/**.
- **The Root Filesystem (****/****):** The core operating system files (like `/bin`, `/etc`, `/lib`, and `/usr`) are installed directly inside the root filesystem.
- **The Mount Point:** A mount point is **simply an empty directory** where a secondary filesystem is logically attached.

Mounting is Logical, Not Physical
When you mount a separate physical disk partition (e.g., `/dev/sda2`) to `/home`, **/home** **is not physically inside the root partition (****/dev/sda1****)**. They are physically isolated, but Linux **logically merges** them into the single directory tree.

- **The Hiding Rule:** If you mount a partition to a directory that already contains files, the old contents are **temporarily hidden** while the partition is mounted. The directory will only show the files inside the newly attached filesystem.

5. The Superpowers of LVM: Dynamic Resizing and Deletion

LVM's greatest advantage is its ability to adapt on live, running servers.
Growing a Volume Live (No Downtime)
If your operating system partition (`/` at `/dev/vg/root`) is full at 20GB, but your user partition (`/home` at `/dev/vg/home`) is empty at 200GB, you can fix it instantly:

1. **Extend the LVM Container:** Add raw block space to the Logical Volume container: `sudo lvextend -L +10G /dev/vg/root`

_Note:_ This command only grows the LVM "box," not the filesystem inside it. Your OS won't see the new space yet.

1. **Extend the Filesystem:** Stretch the filesystem to fill the newly expanded container: `sudo resize2fs /dev/vg/root`

- **Boom!** Your system partition instantly gains 10GB of usable space live, with no reboots, no backups, and no downtime. (On older filesystems, you might be required to unmount the volume and run a check with `e2fsck` before resizing, but modern ext4 allows online resizing).

Removing a Volume
If you no longer need a partition and want to return its space to the Volume Group pool, you can remove it:

1. Ensure the volume is **not mounted**.
2. Double-check that it does not contain any valuable data.
3. Run the deletion command: `sudo lvremove /dev/mynew_vg/vol01`

The freed-up block space instantly returns to your Volume Group, ready to be assigned to another Logical Volume.

# 1️⃣0️⃣ TTY

#### Demystifying the Linux TTY: A Historical and Architectural Masterclass
When you open a terminal in Linux, you are interacting with a software design that is over 70 years old.
The name **TTY** stands for **Teletypewriter**. It represents the evolutionary bridge from physical, paper-spitting typewriter machines to the modern windowed command lines we use today.
#### Phase 1: The Dawn of Computing (No Screens, No Keyboards)

In the **1950s and 1960s**, computers did not have monitors, keyboards, or mice. Instead, programmers used **punch cards**.

```
[Write Code on Paper] ➔ [Punch Holes in Card] ➔ [Feed Cards to Computer] ➔ [Wait Hours] ➔ [Get Printed Output]
```

This method was painfully slow, tedious, and completely lacked real-time interaction. If you made a single typo, your entire stack of cards was ruined, and you had to start over.

#### Phase 2: Humans Want to "Talk" (The Teletype Era)

To solve this, engineers looked for a way to have a live conversation with the computer. They took an existing technology—the electric typewriter—and wired it directly to the computer mainframe. This machine was called a **Teletype** (or **TTY** for short).

```
+--------------------------------------------------+
|                    TELETYPE                      |
|  [Keyboard] ➔ (User types "ls")                  |
|                                                  |
|  [Printer]  ➔ (Computer prints files on paper)   | [1, 2]
+--------------------------------------------------+
```

When you sat at a TTY:

1. You typed a command on the physical keyboard (e.g., `ls`).
2. The electrical signals traveled down a wire to the massive computer box down the hall.
3. The computer processed the command and sent back electrical signals.
4. The TTY's physical printer typed the output directly onto a roll of paper (e.g., printing `file1 file2`).

There was still **no monitor**. Your command history and the computer's output were physically printed on continuous rolls of paper.
#### Phase 3: From Paper to Glass (The Terminal is Born)

By the **1970s**, continuous paper printing was recognized as slow, noisy, and extremely expensive. Engineers decided to replace the heavy physical printer mechanism with a **glass television screen**. This new combination of a keyboard and a screen was called a **terminal**. Crucially, **the terminal was designed to behave exactly like the old paper-based TTY**. The computer sent the exact same characters, but instead of a physical typewriter hammer hitting paper, a beam of electrons painted characters on a screen. Because the underlying communication protocols didn't change, the term **TTY stuck around** as the standard name for these communication lines.

#### Phase 4: The Birth of UNIX and the "Everything is Text" Philosophy

During this exact transition from teletypes to video terminals, the **UNIX operating system** was born. UNIX was designed around a core philosophy: **"Everything is text"**. Because mainframes were incredibly expensive, companies couldn't buy a computer for every employee. Instead, they bought **one central UNIX mainframe** and connected **dozens of physical terminals** to it using long serial cables.

```
                  +-----------------------+
                  |  Central UNIX Server  |
                  +-----------+-----------+
            /-----------------+-----------------\
           /                  |                  \
  +--------+--------+  +--------+--------+  +--------+--------+
  |  User 1 Terminal|  |  User 2 Terminal|  |  User 3 Terminal|
  |     (TTY1)      |  |     (TTY2)      |  |     (TTY3)      | [2, 3]
  +-----------------+  +-----------------+  +-----------------+
```

Each user sat at their own terminal. To the central mainframe, **each connected wire was designated as a unique TTY**. The operating system read text input from that specific TTY line, processed it, and routed the text output back down the same TTY line to the user's screen.

#### Phase 5: The Architecture (TTY vs. Shell)

It is very common to confuse the terminal (TTY) with the shell, but they perform completely different jobs.

1. **The TTY (The Communication Line):** This is the physical or virtual connection wire. Its job is purely transmission: it takes the raw keystrokes you type and sends them to the computer, and takes the raw text bytes from the computer and displays them on your screen.
2. **The Shell (The Translator):** Computer programs and the operating system kernel cannot natively understand human language. The shell is a special software translator that sits on the receiving end of the TTY.
    - When you type `ls` and hit enter, the **TTY** sends those raw letters to the system.
    - The **Shell** reads the letters `l` and `s`, translates them to mean _"run the list-directory program,"_ executes that program, and feeds the resulting directory list back to the TTY to display on your screen.

#### Phase 6: Personal Computers and Virtual TTYs

By the **1980s and 1990s**, Personal Computers (PCs) arrived. Instead of sharing one giant mainframe down the hall, every user had their own individual computer with a monitor and keyboard physically built right into the box. Even though the physical serial wires were gone, **Linux kept this exact same multi-terminal UNIX architecture** in its core code. When your Linux computer boots up:

1. It initializes several **virtual TTYs** in the background.
2. It displays a text-based login prompt on one of them.
3. Once you log in, it starts a **shell interpreter** inside that TTY.

This design means your single, personal computer still behaves internally like a powerful multi-user mainframe.
#### Phase 7: Graphical User Interfaces (GUIs) and Terminal Emulators

When you use a modern desktop environment like GNOME, KDE, or Ubuntu's default desktop, you are using a **Graphical User Interface (GUI)** with windows, icons, and a mouse. Because modern users expect graphical windows but still need access to the command line, engineers created **Terminal Emulators** (such as GNOME Terminal, Alacritty, or `xterm`). A terminal emulator is simply a **graphical window that pretends to be a physical hardware TTY**.

```
[You Type in GNOME Terminal] 
           │
           ▼
[Linux Creates a Fake TTY: Pseudo-Terminal (PTY)] 
           │
           ▼
[Bash Shell runs inside PTY, thinks it's talking to a hardware teletype] [4]
```

Under the hood, when you open GNOME Terminal, Linux allocates a **Pseudo-Terminal (PTY)**. To the Bash shell running inside it, this PTY behaves exactly like a physical, hardware TTY from the 1970s. The shell has no idea it is running inside a graphical window; it simply reads and writes text streams exactly as it did decades ago.
#### Phase 8: The Ultimate Troubleshooting Lifesaver (Ctrl+Alt+F Keys)

Because Linux maintains these virtual hardware TTYs in the background, you have a powerful safety net built directly into your system keyboard. Linux systems run several real, text-only TTYs in the background (usually designated as `tty1`, `tty2`, `tty3`, up to `tty6`). Your graphical desktop (GUI) is typically launched on just **one** of these virtual screens (frequently `tty1` or `tty2`). The Emergency Rescue Scenario, Imagine you are updating your graphics drivers or running a heavy application, and your graphical desktop suddenly **freezes completely**. Your mouse cursor won't move, your windows are locked, and clicking does nothing.
On Windows, you are forced to hold down the physical power button to hard-reboot, risking data corruption.
On Linux, you can bypass the frozen GUI entirely using the keyboard shortcut: **Ctrl + Alt + F3** (or `F4`, `F5`, `F6`)
This key combination instantly switches your monitor away from the frozen graphical interface (`tty1`) and drops you directly into a raw, black-and-white, text-only virtual console (tty3).
Once there, you can:

1. Type your username and password to log in.
2. Run troubleshooting commands (like `top` or `ps aux` to find the frozen application).
3. Kill the frozen desktop process safely (e.g., `sudo systemctl restart gdm`).
4. Switch back to your fixed graphical desktop using **Ctrl + Alt + F1** or **Ctrl + Alt + F2**.

This keeps your running work safe and prevents you from having to hard-reset your hardware.

Summary Checklist: TTY Concepts at a Glance

|Term|What it is in Simple Terms|Modern Equivalent|
|---|---|---|
|**TTY**|A communication line mapping keystrokes to text output.|`/dev/ttyX` virtual console.|
|**Terminal**|A physical screen and keyboard combination.|The physical monitor and keyboard plugged into your computer.|
|**Shell**|The command interpreter/translator that executes programs.|`bash`, `zsh`, or `sh`.|
|**Terminal Emulator**|A software window drawing a fake terminal.|GNOME Terminal, Termius, or `xterm`.|
|**PTY**|A "Pseudo-Terminal" created inside a GUI.|`/dev/pts/0`, `/dev/pts/1`, etc.|

# Other Version TTY
### STEP 1: Before screens existed (1950s–60s)

Early computers had:

- No monitor
- No keyboard
- No mouse

People used punch cards
You punched holes → fed cards → waited → got printout.
Painful. Slow.

### STEP 2: Humans want to talk to computers

Engineers said: “We need a typewriter connected to the computer”
So they made a machine:

- Keyboard
- Printer (not screen)

You type → computer replies → it prints on paper
This machine was called a: Teletype (short: TTY)
TTY = a typing machine connected to a computer

### STEP 3: TTY becomes standard

Every computer had:

- One computer (big box)
- Many teletypes connected by wires

People sat at TTY machines and typed commands.
Example: ls
Computer prints: file1 file2
No screen. Just paper.

### STEP 4: Screens replace printers (1970s)

Printers were slow and expensive.
So engineers replaced: Printer with Screen
Keyboard + screen = terminal
Important: The terminal behaved EXACTLY like a TTY
So the name TTY stayed even though paper was gone.

### STEP 5: UNIX is born (this matters)UNIX was created in this era.

UNIX philosophy: “Everything is text”
So UNIX:

- Talks through TTY
- Reads input from TTY
- Writes output to TTY
- This is why Linux (UNIX-like) still uses TTY today.

### STEP 6: One big computer, many users

One UNIX computer.
Many people connect using terminals.
Each person:

- gets one TTY
- logs in
- runs commands

TTY = your communication line

### STEP 7: What is a shell? (finally)

Computer programs are dumb.
They don’t understand human words.
So UNIX added a translator.
This translator:

- reads text
- understands commands
- runs programs

This program is the: Shell
Example: You → "ls" Shell → "run the ls program"
So:

- TTY = communication line
- Shell = translator

### STEP 8: PCs arrive (1980s–90s)

Now:

- Each person has their own computer
- Screen + keyboard built in

But UNIX/Linux kept the same design.
So when Linux boots:

1. It creates TTY
2. Shows login3. Starts a shell

Nothing changed conceptually.

### STEP 9: GUI appears (Windows, X11, GNOME)

People want windows and mouse.
So engineers said: “Let’s draw fake terminals inside windows”
This is: Terminal Emulator
Examples:

- GNOME Terminal
- xterm

They pretend to be a real TTY
Internally:

- Linux creates a fake TTY (PTY)
- Shell runs inside it

### STEP 10: Why Ctrl+Alt+F keys exist

Those keys switch between real TTYs.
Linux always keeps:

- TTY1, TTY2, TTY3…

GUI usually runs on one of them.
If GUI dies:

- TTY still works
- You can log in and fix stuff

# 1️⃣1️⃣ UFW

A firewall is your system's digital security guard. It inspects all network traffic trying to enter or leave your computer and decides whether to let it pass.

In Linux, the core firewall engine built into the kernel is called **iptables**. However, writing rules for `iptables` directly is notoriously difficult, verbose, and easy to mess up. To make things easier, Ubuntu uses **UFW (Uncomplicated Firewall)**, which is a user-friendly command-line interface that translates simple human commands into complex `iptables` rules behind the scenes.
#### The Core Risk: The Remote Server Trap

Before touching any firewall settings, you must understand the single most dangerous mistake a systems administrator can make: **locking yourself out of your own server**.
**The Scenario**
You are connected to a cloud server (like AWS, DigitalOcean, or a VPS) using **SSH (Secure Shell)** on its default port, **22**. If you enable UFW without setting up an explicit exception rule first, UFW will immediately block all incoming connections—including your active SSH session. Your terminal will freeze, you will be disconnected, and you will have **zero way to log back in** to turn it off.
The Golden Rule
Always configure your allowance rules (especially for SSH) **before** you enable the firewall. UFW lets you view, create, and modify rules while it is completely disabled, making it safe to set up your safety net first.
#### How UFW Works: Default Policies

At its heart, UFW operates on a set of **default policies** that govern how it behaves when no specific rule matches a connection. By default, UFW is configured as follows:

- **Default Incoming: DENY** Blocks all incoming connection attempts from the outside world. Nobody can connect to your machine unless you explicitly allow them to.
- **Default Outgoing: ALLOW** Allows your machine to freely initiate connections to the outside world. You can browse websites (HTTP/HTTPS) and download updates without restriction.

This configuration is **perfect for desktop users** because you want to fetch data from the internet, but you do not want random machines on the web trying to connect to your computer. However, for a **server**, you must customize this behavior so legitimate users can reach your hosted services.
#### Step-by-Step: Setting Up and Managing UFW

Here is the operational lifecycle of managing UFW on your system:

Step A: Installation and Status Check

On Ubuntu systems, UFW is installed by default but is **disabled** so it does not interfere with your initial network setup.

- **Install UFW** (if it is missing): `sudo apt install ufw`
- **Verify Installation:** `ufw --version`
- **Check Status and Active Rules:** `sudo ufw status`

If UFW is inactive, this command will simply return `Status: inactive`. If it is active, it will display a table of your current active firewall rules.

Step B: Setting Your Initial Rules (Safety First!)

Before turning the firewall on, you must allow your management ports. If you use SSH, you can allow it using either the port number or the service name:

- **Allow by Port:** `sudo ufw allow 22`
- **Allow by Service Name:** `sudo ufw allow ssh`

If your server runs a web server (like Apache or Nginx), you should also allow web traffic on port 80: `sudo ufw allow 80` Or at the service level: `sudo ufw allow apache`

Step C: Enabling and Disabling the Firewall

Once your rules are safely in place, you can turn the firewall on : Enable UFW:  `sudo ufw enable`
  *Note:* This starts the firewall immediately and configures it to **automatically start up every time your system boots** . You will see a warning confirming that the firewall is active and enabled on system startup .
If you need to troubleshoot network issues or turn the firewall off:  **Disable UFW:** `sudo ufw disable` 
  This stops the firewall immediately and prevents it from starting on boot.

#### Step D: Applying New Rules (Reloading)
If UFW is already running and you add or change a rule, **the changes do not take effect immediately** . You have two ways to apply them:

1. **The Manual Toggle:** Disable and re-enable UFW .
2. **The Clean Way (Reload):** Keep the firewall running and simply reload the configuration: `sudo ufw reload`
#### Step E: The Emergency Reset

If you make a mistake, create conflicting rules, or lock yourself into an odd configuration, you can wipe the slate clean: `sudo ufw reset`
This command **disables UFW and deletes all custom rules**, returning the firewall to its factory default state (no exceptions).
#### Advanced Rule Configuration: Ports, Protocols, and Applications

UFW gives you incredibly granular control over how you define what traffic is allowed or blocked.
#### 1. Managing IPv4 vs. IPv6
By default, whenever you add a rule (like allowing port 22), UFW adds two rules: one for **IPv4** and one for **IPv6** . 
If you want to disable IPv6 support entirely to simplify your rules:
1. Open the configuration file **/etc/default/ufw** in a text editor.
2. Find the line that says `IPV6=yes`.
3. Change it to **`IPV6=no`**.
#### 2. Filtering by Specific Protocols (TCP vs. UDP)
Some services use specific network transport protocols. By default, UFW allows both protocols. If you want to restrict a rule to just one:
* **Allow TCP only** (common for web traffic): `sudo ufw allow 80/tcp`
* **Allow UDP only** (common for DNS or streaming): `sudo ufw allow 53/udp`
#### 3. Allowing Port Ranges
If you run an application that requires multiple consecutive ports (for example, a game server or a voice-over-IP service), you can open a range. **When specifying a range, you must define the protocol (tcp or udp):** `sudo ufw allow 90:100/tcp`
This opens all ports from 90 through 100 for TCP connections.
#### 4. The Magic of `/etc/services`
When you run a command like `sudo ufw allow ssh` or `sudo ufw deny http`, how does UFW know what numbers those services use?
It reads a special system dictionary file located at **`/etc/services`**. This file maps thousands of human-readable service names to their official port numbers. You can inspect this file yourself using: `less /etc/services`
#### 5. Application Profiles
Many modern Linux applications register **profiles** with UFW during installation. These profiles package all the specific ports and protocols the application needs into one simple, named package.

* **List available application profiles:** `sudo ufw app list`
  *(On a standard machine, you will typically see things like `OpenSSH` or `CUPS` (printing service) listed .)*
* **Allow an application profile:**
  Instead of memorizing which ports a complex app needs, you can simply allow the app profile: `sudo ufw allow OpenSSH`
### Summary Checklist: Essential UFW Commands

| Action | Command | When to use it |
| :--- | :--- | :--- |
| **Check Active Rules** | `sudo ufw status` | To see what ports are currently open or closed [4]. |
| **Pre-Authorize SSH** | `sudo ufw allow 22` | Run this first on remote servers to prevent lockouts [2]. |
| **Turn On** | `sudo ufw enable` | Activates protection and enables boot-start [5]. |
| **Apply Changes** | `sudo ufw reload` | Run this after adding new rules to a running firewall [5]. |
| **Block a Port** | `sudo ufw deny 80` | Blocks incoming traffic on a specific port [6, 7]. |
| **Start Over** | `sudo ufw reset` | Erases all rules and turns off the firewall [5]. |

# 1️⃣2️⃣ What is `chroot`?

The word **chroot** stands for **"change root"**.
To understand it, imagine Linux as a **city** where the root directory (`/`) represents the city boundary. Normally, running programs think this boundary is the entire world—they cannot see or travel outside of it.
When you run `chroot` on a directory (such as `/mnt`), you are telling a program: **"Your world ends here now"**. From that moment on, that specific directory behaves exactly like the root directory (`/`) for any process running inside it. It is **not** a Virtual Machine (VM) or a container; it does not reboot your system, and it shares the exact same running kernel as the host. It is simply a way to swap out the filesystem "mask" that the system is currently wearing.

#### Why is `chroot` Needed? (The Real-World Problems)

The absolute most common "why" behind `chroot` is **boot repair and disaster recovery**.

When a system fails to boot—whether due to a kernel panic, a black screen after typing your disk password, a broken GRUB bootloader, or a missing initial ramdisk (initramfs)—you cannot access the operating system normally. To fix it, you boot into a "Live USB" rescue environment.

However, if you try to fix things directly from the Live USB's terminal, you run into a pathing conflict:

- If you run `apt install` or edit `/etc/default/grub`, you will modify the **Live USB's** temporary files, not the files on your broken hard drive.
- If you try to run bootloader setup tools, they won't know where the actual operating system files are.

**chroot** **solves this** by allowing you to mount your offline hard drive to a folder (like `/mnt`) and "step inside" it. Once inside, your terminal treats `/mnt` as `/`. This gives you:

1. **The correct filesystem paths** (so `/etc` points to your actual system config, not the USB's config).
2. **The correct package database** (so your package manager knows exactly what is installed on your hard drive).
3. **The ability to run repair commands** (like rebuilding the bootloader or resetting a forgotten password) as if you had successfully booted into the system.

Beyond boot repairs, `chroot` is an industry-standard DevOps tool used to **install Linux without booting it** (how cloud images for AWS or Azure are pre-built), repair broken software packages when the desktop GUI won't load, build software for different hardware architectures (like compiling ARM code on an Intel laptop), and analyze compromised filesystems safely without executing potentially dangerous startup scripts on the host.

#### What Must Be Set Up Before `chroot`? (And Why)

If you simply type `sudo chroot /mnt` immediately after mounting your hard drive, it will not work properly. Because your installed operating system is not actually booted, none of the dynamic system components are running.

During a healthy boot, Linux reads the `/etc/fstab` file to automatically mount your drives and set up the environment. When troubleshooting, **you must manually recreate what** **fstab** **normally does**.

1. Mounting the Real Root (`/`)

You first need to find the large partition containing your actual operating system files (often living inside a Logical Volume Manager (LVM) volume) and mount it to `/mnt`.

2. Mounting Separate Partitions (Risk Control)

Linux split filesystems into separate physical partitions to **control risk** (preventing a full disk or corrupted partition from destroying the entire OS). Because of this, you must mount these separate pieces exactly where they belong inside `/mnt`:

- **/boot**: The GRUB bootloader is simple by design and cannot read complex, modern layers like encrypted disks (LUKS) or LVM pools in its early stages. Therefore, we separate `/boot` into its own unencrypted, plain filesystem so GRUB can easily read the kernel and boot the machine. If your system uses a separate `/boot`, you must mount it to `/mnt/boot`.
- **/boot/efi**: Modern UEFI firmware is incredibly strict and can _only_ read filesystems formatted as FAT32. This EFI System Partition (ESP) must be mounted to `/mnt/boot/efi` for any bootloader repairs to succeed.

3. Re-creating the Virtual Kernel Interfaces (The `--bind` Mounts)

This is the most confusing part of system recovery. Directories like `/dev`, `/proc`, `/sys`, and `/run` **do not actually exist as real files on your hard disk**. They are virtual "pseudo-filesystems" created dynamically in-memory by the active, running kernel.

Because your installed system is offline, these directories on your hard drive are completely empty. However, repair tools desperately need them:

- `grub-install` needs to see physical disks via `/dev`.
- System tools need to check active processes and hardware states via `/proc` and `/sys`.

To solve this, we use a **bind mount (****mount --bind****)**. While a normal mount connects a physical disk to a folder, a bind mount **mirrors an existing directory to another location**. By running `mount --bind /dev /mnt/dev`, you are essentially "borrowing" the running Live USB's virtual device interfaces and mirroring them inside your recovery folder so the chrooted environment can use them.

#### Who Are You Inside `chroot`? (Identity & Security)

Inside a `chroot` session, you are **not** a new virtual user. You are still the **exact same user ID** (typically root, UID 0) with the exact same administrative privileges granted by the running kernel of the Live USB.

What _does_ change is where your user databases are read from:

- If you run `cat /etc/passwd` or change a password using `passwd`, Linux reads and writes to the files belonging to the **installed target system** (e.g., `/mnt/etc/passwd` and `/mnt/etc/shadow`).
- If you install a package, the files and database records are written strictly to the offline target disk. Once you reboot, those changes will be permanently present.

#### The Disk Encryption Boundary

Because Unix assumes that anyone with physical access and root privileges can boot a Live USB, mount a connected drive, and `chroot` into it to read any file or reset any password, physical security alone is not enough.

This is the exact reason **Full Disk Encryption (such as LUKS)** exists. Encryption acts as the ultimate boundary: if someone physically steals your hardware, they cannot mount your filesystems or `chroot` into your operating system to bypass your password files unless they possess your encryption decryption key.

# 1️⃣3️⃣ Disable Wayland and force Ubuntu/GNOME to use Xorg (X11).

`sudo nano /etc/gdm3/custom.conf`  and  `WaylandEnable=false`
For
➖ Remote desktop / screen sharing issues
➖ Screen recording or screen capture problems
➖ Older applications

# 1️⃣4️⃣ UBUNTU NETWORK

**Phase 1: Physical, Driver & Hardware Verification (Bottom-Up Diagnostics)**
**Step 1: Verify Driver & Hardware Detection**

- **Action:** Check if the operating system recognizes the Network Interface Card (NIC). `lspci | grep -i net`
- **Action:** Check if the kernel driver module for your interface is loaded. `lsmod | grep <driver>`
- **Diagnostic:** If these commands return nothing, **the driver is missing** or the hardware is not properly seated.

**Step 2: Check Physical Link Status**

- **For Ethernet:** Verify the hardware connection and link speed. `ethtool <interface>`
- _Look for:_ `Link detected: yes` and `Speed: 1000Mb/s` (or `100/10` depending on your hardware).
- **For Wi-Fi:** Check the device management status. `nmcli device status`
- _Look for:_ The Wi-Fi interface should be listed as **connected** or **disconnected** (it should not read **unavailable**).
- **Diagnostic:** If `Link detected: no` or the status is unavailable, you have a **cable, port, or physical NIC issue**.

**Phase 2: Interface Control & Local IP Configuration**
**Step 3: Check IP Address Assignment**

- **Action:** Check the IP configuration of your active interfaces. `ip a`
- _Identify interfaces:_ Ethernet interfaces typically start with `enp...` or `eth0`; Wi-Fi interfaces typically start with `wlp...` or `wlan0`.
- **Diagnostic (IP States):**
    - `192.x.x.x`, `172.x.x.x`, or `10.x.x.x` $\rightarrow$ **DHCP is working properly**.
    - `169.254.x.x` $\rightarrow$ **DHCP failed** (Auto-IP / APIPA fallback).
    - _No IP address at all_ $\rightarrow$ **The interface is not up**.
- **Remediation:** If the interface is down, attempt to bring it up manually: `sudo ip link set <iface> up`

**Step 4: Clear & Renew DHCP**

- **Remediation:** If DHCP failed or you need a fresh lease, release the old lease and request a new IP:

```
sudo dhclient -r
sudo dhclient
```

**Phase 3: Network Services & Local Security**
**Step 5: Verify Network Manager Status**

- **Action:** Check if the system's network service is running. `systemctl status NetworkManager`
- **Remediation:** If the service is inactive or down, restart it: `sudo systemctl restart NetworkManager`

**Step 6: Check Firewall Restrictions**

- **Action:** Check if the Uncomplicated Firewall (UFW) is actively blocking connection attempts. `sudo ufw status`
- **Remediation:** If UFW is active and suspected of causing connectivity issues, temporarily disable it to isolate the issue: `sudo ufw disable`

**Phase 4: Routing & DNS Resolution**
**Step 7: Check Gateway Routing**

- **Action:** Verify that a path exists from your system to external networks. `ip route`
- **Diagnostic:** Ensure a default route exists in the output (e.g., `default via <gateway_ip>`). **If this default route is missing, you will not have internet access.**

**Step 8: Check DNS Server Settings**

- **Action:** Verify that your system knows where to send domain name lookup queries. `cat /etc/resolv.conf`
- **Diagnostic:** Look for a configured nameserver (e.g., `nameserver 8.8.8.8` or your local/company DNS addresses).

**Phase 5: Diagnostics & Global Connectivity**
**Step 9: Run Raw Connectivity Tests**

- **Action:** Execute sequential ping tests to isolate where traffic is dropping.

```
ping -c 3 8.8.8.8
ping -c 3 google.com
```

- **Diagnostic Matrix:** 

| Ping Test Result                        | Indicated Failure Point                        |     |
| :-------------------------------------- | :--------------------------------------------- | --- |
| `8.8.8.8` fails                         | **Gateway, routing, or ISP issue**             |     |
| `8.8.8.8` works, but `google.com` fails | **DNS configuration issue**                    |     |
| Both fail                               | **Network stack, hardware, or firewall block** |     |

**Step 10: Worst Case — Reset Network Stack**

- **Remediation:** If configuration files are correct but traffic is still stalled, toggle your networking services off and on to clear the stack:

```
sudo nmcli networking off
sudo nmcli networking on
# Alternate method: sudo systemctl restart NetworkManager
```

# 1️⃣5️⃣ Booting Process of Linux OS

The **Linux booting process** defines the step-by-step sequence that occurs **from powering on the machine until the login screen appears**. Understanding the boot process is very important for **troubleshooting boot-related issues** in Linux systems.
### High-Level Linux Boot Stages

The Linux booting process consists of the following major stages:

1. **BIOS / UEFI**
2. **Boot Loader (GRUB / GRUB2)**
3. **Kernel**
4. **Init System (systemd / init)**
5. **User Space (Login Prompt)**
### Detailed Linux Booting Process

### 1. Power On & SMPS

1. When the machine is powered on, the **SMPS (Switched-Mode Power Supply)** supplies power to all motherboard-connected devices.
2. It converts **AC (Alternating Current)** to **DC (Direct Current)**.
3. It ensures proper voltage is supplied to the system components.
### 2. BIOS Initialization

1. **BIOS (Basic Input/Output System)** is firmware used to initialize hardware during startup.
2. BIOS is the **first program executed** when the system powers on.
3. It verifies whether the system hardware is present and functioning.
4. BIOS manages data flow between:
    - Hard disk
    - Keyboard
    - Mouse
    - Display adapter
    - Other peripherals
5. You can access BIOS setup using keys like **F2, DEL, ESC**, etc.

### 3. POST (Power-On Self Test)

1. BIOS performs **POST** to detect, test, and initialize hardware components.
2. If all devices are healthy, the system gives a **beep sound**.
3. If any hardware issue exists, an **error message** is displayed.
4. After successful POST, BIOS checks boot devices based on **boot priority**.
### 4. CMOS Initialization

1. If POST succeeds, BIOS initializes **CMOS**.
2. CMOS stores: System date and time, Hardware configuration, Boot sequence
3. CMOS is powered by a **CMOS battery**, so data remains even when the system is off.
### 5. MBR (Master Boot Record)

1. BIOS reads the **MBR**, located in the **first 512 bytes** of the boot disk.
2. The MBR identifies:
    - Bootable partitions
    - Location of the boot loader
3. Structure of MBR (512 bytes):

|Section|Size|Description|
|---|---|---|
|Boot Loader Info|446 bytes|Boot loader code|
|Partition Table|64 bytes|Partition details|
|Magic Number|2 bytes|Validation code|
4. The MBR loads and transfers control to **GRUB / GRUB2**.
#### Difference Between BIOS and CMOS

| BIOS                            | CMOS                          |
| ------------------------------- | ----------------------------- |
| Firmware stored in ROM          | RAM chip with volatile memory |
| Initializes hardware & boots OS | Stores BIOS configuration     |
| Loaded into RAM during boot     | Holds ~256 bytes of data      |
| Controls system startup         | Stores time, date, boot order |

### 6. Boot Loader (GRUB / GRUB2)

1. A **boot loader** loads the operating system into memory.
2. GRUB stands for **Grand Unified Boot Loader**.
3. **GRUB2** is the default boot loader for: CentOS, RHEL 7, RHEL 8
4. All boot files are stored in the **`/boot`** directory.
5. GRUB reads its configuration from: `/boot/grub2/grub.cfg`
6. GRUB presents a menu for selecting operating systems.

#### GRUB2 Responsibilities

1. Loads the **Linux kernel (vmlinuz)** into memory.
2. Loads **initramfs / initrd** image.
3. Transfers control to the kernel.
### 7. Kernel & Initramfs

1. The kernel extracts **initramfs** into a temporary memory filesystem (**tmpfs**).
2. Initramfs contains:
    - Required drivers
    - Kernel modules
    - Tools to mount the real root filesystem
3. Temporary filesystem is unmounted after real root (`/`) is mounted.
4. Kernel and initramfs files are located in: `/boot`

#### initrd vs initramfs

|initrd|initramfs|
|---|---|
|Used in older 2.4 kernels|Used in 2.6+ kernels|
|Disk-based|Memory-based|
|Older technology|Successor of initrd|
### 8. Root Filesystem Mounting

1. Kernel mounts:
    - **Virtual root filesystem** (temporary)
    - **Real root filesystem (`/`)**
2. After mounting `/`, kernel executes:
    - `init` (older systems)
    - `systemd` (RHEL 7+)

Check systemd PID: `pidof systemd`
Systemd always runs with **PID 1**.

### 9. systemd / init System

1. `systemd` is the **parent process** of all processes.
2. It controls system services and startup order.
3. Reads default target from: `/etc/systemd/system/default.target`
4. Runlevel mapping:

| Target            | Equivalent Runlevel |
| ----------------- | ------------------- |
| multi-user.target | Runlevel 3          |
| graphical.target  | Runlevel 5          |

5. Legacy runlevel config: `/etc/inittab`
#### systemd Performs

- Setting hostname
- Initializing network
- Initializing SELinux
- Mounting filesystems (`/`, `/proc`, etc.)
- Cleaning `/var` directories
- Enabling swap
- Running `/etc/rc.local` if present
### 10. Login Process

Once all services are started, the system shows the **login screen**. During login, Linux verifies:

- `/etc/passwd`
- `/etc/group`
- `/etc/shadow`
- Password policies
- User bash profile files

If authentication succeeds, the user receives a **command prompt based on their shell**.

# 1️⃣6️⃣ Even more detailed Linux boot process
### BIOS (Basic Input/Output System)

**BIOS** stands for **Basic Input/Output System**. It is firmware stored on the motherboard that runs immediately after the computer is powered on.
### Functions of BIOS
1. **System Integrity Checks (POST)**
    - When the computer starts, the BIOS performs **Power-On Self Test (POST)**.
    - It checks essential hardware components such as: CPU, RAM, HDD/SSD, Keyboard and other basic devices
2. **Boot Device Selection**
    - BIOS searches for a bootable device based on the **boot sequence** configured in BIOS settings.
    - Common boot devices include: Hard drive / SSD, USB drive, CD/DVD, Floppy disk (older systems)
    - You can change the boot order by pressing keys like **F2, F12, DEL, or ESC** during startup (depends on the system).
3. **Loading the Boot Loader**
    - Once a bootable device is found, the BIOS locates the **boot loader**.
    - In legacy systems, the boot loader is stored in the **Master Boot Record (MBR)**.
    - The MBR is located in the **first sector of the disk**.
4. **Handing Control to the Boot Loader**
    - The BIOS loads the boot loader into memory.
    - Control of the system is then transferred from BIOS to the boot loader.
    - From this point onward, the boot loader loads the operating system.
### In Simple Terms
- BIOS starts the computer
- Checks hardware
- Finds a bootable device
- Loads and executes the **MBR boot loader**
- Hands control to the boot loader

## MBR (Master Boot Record)
**MBR** stands for **Master Boot Record**. It is a special boot sector on a storage device and is responsible for loading and executing the **GRUB boot loader** (or **LILO** in very old Linux systems).
#### Location of MBR
- The MBR is located in the **first sector of the bootable disk**.
- Common disk names:
    - `/dev/hda` (older IDE disks)
    - `/dev/sda` (modern SATA/SCSI/USB disks)
#### Size of MBR
- Total size: **512 bytes**
#### Structure of MBR
The 512 bytes of the MBR are divided into three parts:
1. **Primary Boot Loader (446 bytes)**
    - Contains executable boot code.
    - Responsible for loading the next stage of the boot loader (e.g., GRUB).
2. **Partition Table (64 bytes)**
    - Stores information about disk partitions.
    - Supports up to **4 primary partitions**.
3. **MBR Validation Check / Boot Signature (2 bytes)**
    - Contains the boot signature `0x55AA`.
    - Used by BIOS to verify a valid boot sector.
#### Role of MBR in Booting
- BIOS loads the MBR into memory.
- The MBR executes its boot code.
- The MBR then loads and hands control to the **GRUB boot loader**.
#### In Simple Terms
- BIOS → loads MBR
- MBR → loads and executes GRUB
- GRUB → loads the operating system
### GRUB (GNU GRand Unified Bootloader)
**GRUB**, sometimes called **GNU GRUB**, stands for **GNU GRand Unified Bootloader**. It is the **default and most commonly used boot loader** in modern Linux systems.
#### Purpose of GRUB
- GRUB is responsible for **loading the Linux kernel into memory**.
- It allows users to **select which operating system or kernel version** to boot.
- Unlike older boot loaders such as **LILO**, GRUB **understands file systems**, making it more flexible and powerful.
#### GRUB Splash Screen
- The **GRUB splash screen** is often the first screen displayed during system startup.
- It presents a **menu of boot options**.
- If multiple kernel images are installed, you can use the keyboard to choose which kernel to boot.
- By default, the **latest kernel** is selected.
#### Default Boot Behavior
- GRUB waits for a **specified timeout period** for user input.
- If no selection is made, it **automatically loads the default kernel image** defined in the GRUB configuration file.
#### GRUB Configuration File
- The GRUB configuration file is usually located at:
    - `/boot/grub/grub.conf`
    - `/etc/grub.conf` (symbolic link to `/boot/grub/grub.conf`)
- This file defines:
    - Default boot entry
    - Timeout value
    - Kernel and initrd paths
    - Boot menu entries
#### Sample `grub.conf` (CentOS)

```
#boot=/dev/sda
default=0
timeout=5
splashimage=(hd0,0)/boot/grub/splash.xpm.gz
hiddenmenu

title CentOS (2.6.18-194.el5PAE)
      root (hd0,0)
      kernel /boot/vmlinuz-2.6.18-194.el5PAE ro root=LABEL=/
      initrd /boot/initrd-2.6.18-194.el5PAE.img
```
#### Explanation of Key Directives
- **default=0** → Boots the first menu entry by default
- **timeout=5** → Waits 5 seconds for user input
- **splashimage** → Displays the background image
- **hiddenmenu** → Hides the menu unless a key is pressed
- **kernel** → Specifies the kernel image and boot parameters
- **initrd** → Specifies the initial RAM disk image
#### In Simple Terms
- BIOS loads MBR
- MBR loads GRUB
- GRUB displays a menu
- GRUB loads the selected Linux kernel
## Linux Kernel and initrd (Boot Process Stage)
### Role of the Linux Kernel
The **kernel** is the **core of the Linux operating system**. It has complete control over system resources such as:
- CPU
- Memory
- Storage
- Hardware devices
Once GRUB loads the selected kernel, the kernel takes control of the system.
### Mounting the Root File System
- The kernel mounts the **root file system** as specified by the `root=` parameter in the **GRUB configuration file** (`grub.conf`).
- This root file system becomes `/`, from which all other directories are accessed.
### initrd (Initial RAM Disk)
- **initrd** stands for **Initial RAM Disk**.
- It is a **temporary root file system** loaded into memory during the early boot process.
- The kernel uses initrd until the **actual root file system** is mounted.
#### Purpose of initrd
- Contains essential **drivers and kernel modules**.
- Helps the kernel:
    - Access disk controllers
    - Detect storage devices
    - Mount hard drive partitions
- Especially important when required drivers are **not compiled directly into the kernel**.
### Execution of `/sbin/init`
- After mounting the root file system, the kernel executes the program **`/sbin/init`**.
- `init` is the **first user-space process** started by the kernel.
- Since it is the first process, it always has: **Process ID (PID) = 1**
You can verify this using: `ps -ef | grep init`
### Summary of This Boot Stage
1. GRUB loads the selected kernel and initrd.
2. Kernel initializes hardware and core subsystems.
3. Kernel mounts initrd as a temporary root file system.
4. Kernel mounts the real root file system (`root=`).
5. Kernel executes `/sbin/init` (PID 1).
### In Simple Terms
- GRUB loads kernel →
- Kernel uses initrd →
- Kernel mounts real root filesystem →
- Kernel starts `init` (PID 1)
## Why does Linux load a **temporary filesystem (initrd/initramfs)** instead of the real root filesystem immediately?
#### Short answer
Because **the kernel often cannot access the real disk yet**.
#### What the kernel can do at boot time
When the kernel is first loaded:
CPU → OK  
RAM → OK  
Disk → **NOT OK yet**
Why?
Because access to the disk depends on:
- Storage controller drivers (SATA / NVMe / RAID)
- Filesystem drivers (ext4, xfs, etc.)
- LVM, RAID, encryption (LUKS)

These are **drivers/modules**, and many of them are **NOT built into the kernel**.
So the kernel is stuck thinking: “I need the disk to load drivers,  but I need drivers to access the disk.”
Classic **chicken-and-egg problem**
#### That’s where the **temporary filesystem** comes in
**initrd / initramfs**:
- Is loaded **into RAM**
- Needs **no disk access**
- Contains:
    - Essential drivers
    - Tools to detect disks
    - Scripts to mount the real filesystem
It gives the kernel a **safe workspace** to prepare the system.
#### Real-world analogy (very important)
Imagine moving into a new house :
- You don’t know where anything is
- Electricity is not connected yet
So you: Bring a **temporary toolbox**
- Use it to:
    - Turn on electricity
    - Unlock doors
    - Make the house usable
That toolbox = **initrd**  
The house = **real root filesystem**
## What does “mounting the root filesystem” ACTUALLY mean?

#### First, what is “mounting”?
In Linux: **Mounting = attaching a filesystem to a directory**
Example: `mount /dev/sda1 /mnt`
Now `/mnt` shows the contents of `/dev/sda1`.
#### What is the **root filesystem**?
The root filesystem is: `/`
Everything in Linux starts from `/`:
- `/bin`
- `/etc`
- `/home`
- `/var`
- `/usr`
Without `/`, **Linux cannot run**.
### Where is the root filesystem mounted?
#### This is the KEY point most people miss 
#### The real root filesystem is mounted at `/`
Yes — **literally at `/`**. But there’s a twist…
### Boot sequence with mounting (step-by-step)

#### Step 1: Kernel starts
- No filesystem mounted yet
- Only RAM exists
#### Step 2: initrd is mounted as **temporary root**
The kernel mounts initrd as: `/`
So temporarily: `/  → initrd (in RAM)`
This temporary `/` contains:
- `/bin`
- `/sbin`
- `/lib`
- Scripts to find the real disk
#### Step 3: initrd finds the real root filesystem
Example: `root=/dev/mapper/vg0-root`
initrd:
- Loads disk drivers
- Activates LVM / RAID / encryption
- Locates the real root filesystem
#### Step 4: Switch root (VERY IMPORTANT)
Now Linux does: `switch_root` (or `pivot_root` in older systems)
This:
- **Unmounts initrd**
- **Mounts the real filesystem at `/`**
After this: `/ → real disk filesystem`
initrd is gone or freed from RAM.
#### Step 5: Start init (/sbin/init)
Now the kernel runs: `/sbin/init`
From the **real root filesystem**, not initrd.
## One picture in words (remember this)

```
Boot start    
↓ 
Kernel loaded    
↓ 
Temporary / (initrd in RAM)    
↓ 
Drivers loaded, disk found    
↓ 
switch_root    
↓ 
Real / (disk filesystem)    
↓ 
/sbin/init starts
```
## Why not always use initrd?

Actually… **modern Linux always does**.
Even if:
- Kernel _can_ access the disk
- Drivers are built-in
initramfs is still used because:
- It’s flexible
- Supports encryption, LVM, RAID
- Makes kernel smaller and modular
## Linux Run Levels and systemd Targets

After the kernel starts `/sbin/init` (PID 1), the system enters the **runlevel stage**, where startup services and system processes are launched.
## Legacy Init (SysV init)
### `/etc/inittab`
- In traditional Linux systems, **init** reads the file: `/etc/inittab`
- This file determines the **default run level** of the system.
- The default run level is identified by the `initdefault` entry.
You can check the default run level using: `grep initdefault /etc/inittab`
### Legacy Run Levels

|Run Level|Description|
|---|---|
|0|Halt (Shutdown)|
|1|Single-user mode (Maintenance)|
|2|Multi-user mode (without NFS)|
|3|Full multi-user mode (Text mode)|
|4|Unused|
|5|Multi-user mode with GUI (X11)|
|6|Reboot|

### Important Notes
- Run levels **0** and **6** are dangerous as defaults:
    - `0` → System shuts down
    - `6` → System reboots
- Typically, systems are set to:
    - **Run level 3** (servers)
    - **Run level 5** (desktop systems)
## Modern Linux: systemd
Modern Linux distributions use **systemd** instead of SysV init.  
systemd replaces run levels with **targets**, while still maintaining compatibility.
### systemd Run Level Mappings

|Legacy Run Level|systemd Target|
|---|---|
|0|`poweroff.target`|
|1|`rescue.target`|
|3|`multi-user.target`|
|5|`graphical.target`|
|6|`reboot.target`|
|Emergency|`emergency.target`|

- `runlevelX.target` files are symbolic links to the corresponding systemd targets.
- systemd executes services and units based on the selected target.
## Boot Flow at This Stage
1. Kernel starts `/sbin/init` (PID 1)
2. init/systemd determines the **default run level or target**
3. Required services and processes are started
4. System reaches a usable state (text mode or GUI)
## In Simple Terms
- Old systems → `/etc/inittab` decides run level
- New systems → systemd decides target
- Run level 3 → Text mode
- Run level 5 → GUI mode
## Runlevel Programs (SysV init)
During system startup, Linux starts various services. You may see messages such as:  `Starting sendmail … OK`
These messages indicate that **runlevel programs (services)** are being executed.
## Runlevel Directories
Each run level has a corresponding directory that contains scripts to be executed when the system enters that run level.
### Common Runlevel Directories
Depending on the distribution, these directories may exist in either form:
- `/etc/rc.d/rcX.d/`
- or symbolic links directly under `/etc/rcX.d/`

|Run Level|Directory|
|---|---|
|0|`/etc/rc.d/rc0.d/`|
|1|`/etc/rc.d/rc1.d/`|
|2|`/etc/rc.d/rc2.d/`|
|3|`/etc/rc.d/rc3.d/`|
|4|`/etc/rc.d/rc4.d/`|
|5|`/etc/rc.d/rc5.d/`|
|6|`/etc/rc.d/rc6.d/`|

Example: `/etc/rc0.d` is usually a symbolic link to `/etc/rc.d/rc0.d`
## Startup (S) and Kill (K) Scripts
Inside each runlevel directory, you will find scripts whose names start with **S** or **K**.
### Script Naming Convention
- **S** → Startup scripts (executed when entering the run level)
- **K** → Kill scripts (executed when leaving the run level or during shutdown)
Each script name also contains a **sequence number** that controls execution order.
### Example Script Names
- `S12syslog`
    - Starts the **syslog daemon**
    - Sequence number: **12**
- `S80sendmail`
    - Starts the **sendmail daemon**
    - Sequence number: **80**
Since `12 < 80`, **syslog starts before sendmail**.
## Execution Order Logic
- Scripts are executed in **ascending numerical order**
- Lower numbers start (or stop) **earlier**
- This ensures proper dependency handling between services
## In Simple Terms
- Linux starts services based on the current run level
- Each run level has its own directory
- `S` scripts start services
- `K` scripts stop services
- Numbers decide the order of execution

## What is the ACTUAL use of runlevels?
**Runlevels control _how much of the system is turned on_.**  
That’s it. They decide:

- Which **services start**
- Whether you get **CLI or GUI**
- Whether the system is in **normal mode, maintenance mode, shutdown, or reboot**

Think of runlevels as **startup modes**.
## Simple analogy (best way to get it)
### Like starting a car 🚗
- **Ignition only** → checking engine (maintenance)
- **Engine on, no AC, no music** → basic driving
- **Everything on (AC, music, GPS)** → full mode
- **Turn off engine** → shutdown
- **Restart engine** → reboot
Runlevels do **exactly this** for Linux.
## Why Linux needs runlevels

Linux can be used as:
- A **server** (no GUI needed)
- A **desktop** (GUI needed)
- A **broken system** (needs repair)
- A system that must **shutdown or reboot**
So Linux asks: “How much should I start right now?”
Runlevel answers that question.
## What each runlevel ACTUALLY does (real meaning)
### Runlevel 1 – Single-user mode (FIXING MODE)
- Only root user
- No network
- No GUI
- Used when:
    - System is broken
    - You forgot root password
    - Filesystem repair
👉 **Purpose:** _Fix the system safely_
### Runlevel 3 – Normal SERVER mode
- Multi-user
- Networking ON
- NO GUI
👉 **Purpose:** _Run servers efficiently_  
(why waste RAM on GUI?)
### Runlevel 5 – Normal DESKTOP mode
- Everything in runlevel 3
- PLUS GUI (X11)
👉 **Purpose:** _Normal desktop usage_
### Runlevel 0 – Shutdown
- Stops all services
- Powers off system
👉 **Purpose:** _Turn off machine_
### Runlevel 6 – Reboot
- Stops services
- Restarts system
👉 **Purpose:** _Restart machine_

## So what REALLY happens behind the scenes?

Runlevel decides: “Which services should start?”
Example:
- Runlevel 3 → start ssh, network, logging
- Runlevel 5 → start ssh, network, logging **+ GUI**
- Runlevel 1 → start **almost nothing**
That’s why you saw: `Starting sendmail... OK Starting syslog... OK`
Those services are started **because the current runlevel allows them**.
## One killer sentence (remember this)

**Runlevels decide the system’s operating mode by controlling which services are started.**
If you remember ONLY this → you’re good.
### Modern reality (important)
Today:
- **systemd replaced runlevels**
- But the IDEA is the same
Runlevel → **systemd target**
Example:
- runlevel 3 → `multi-user.target`
- runlevel 5 → `graphical.target`
Same concept, better implementation.
# 1️⃣7️⃣ Linux Graphical Stack

`Display Manager → Display Server → Window Manager / Compositor → Applications`

### 1. Display Manager (Login / Lock screen)

The **Display Manager** starts the graphical session and shows the **login screen**.
Examples:
- GDM
- SDDM
- LightDM

What it does:
- Shows **login screen**
- Lets you choose **X11 or Wayland session**
- Starts the desktop environment or window manager after login

Example flow:  `Boot → GDM → login → start GNOME / i3 / Hyprland`

### 2. Display Server (Graphics system)

This is the system that **talks to the GPU and draws windows**. Two main ones:

- X Window System
- Wayland

They handle:
- drawing windows
- keyboard input
- mouse input
- screen rendering

### 3. Window Manager / Compositor

This controls **how windows behave**.
Examples:
- i3
- Openbox
- Hyprland

What it controls:
- window tiling
- moving/resizing windows
- animations
- workspaces

Note:
- In **X11**, window manager is separate.
- In **Wayland**, the **compositor + window manager are usually the same program**.

## Architecture

### X11 system

```
Display Manager (GDM)  
        ↓  
X Server (X11)  
        ↓  
Window Manager (i3 / Openbox)  
        ↓  
Applications
```

### Wayland system

```
Display Manager (GDM)  
        ↓  
Wayland Compositor (Hyprland / GNOME)  
        ↓  
Applications
```

**Simple way to remember**

|Component|Role|
|---|---|
|Display Manager|Login screen|
|Display Server|Draws graphics|
|Window Manager|Controls windows|

 
# 1️⃣8️⃣ Another partitions “inside” root

### WHY PARTITION
#### Why folders alone are NOT enough

If everything lived in `/` on one partition:
- Disk corruption → entire OS dead
- Encryption → bootloader can’t read kernel
- Full disk → system won’t boot
- Filesystem mismatch → bootloader can’t read it

Linux splits disks to **control failure domains**.
#### The bootloader problem (this is the real reason)

Bootloader (GRUB) is **stupid by design**. It can only read:
- very few filesystems
- **no encryption** (early stage)
- **no LVM** (early stage)

So we separate:

`/boot`
- Plain filesystem (ext4)
- Unencrypted
- Simple layout

So GRUB can:
- read kernel
- read initramfs
- start Linux

If `/boot` lived inside encrypted `/`:   ❌ GRUB would be blind. That’s the **#1 reason** `/boot` exists.
#### EFI is even stricter

UEFI firmware can only read:
- **FAT32**
- No ext4
- No encryption
- No LVM

So we need: EFI System Partition (ESP)
- FAT32
- Very small
- Firmware-readable

Mounted at: `/boot/efi`
Without it: System literally cannot start
This is not a Linux choice — it’s a firmware rule.
#### Why not just “create a folder in /”?

Because:

Folder:
- Exists **inside** a filesystem
- Dies with that filesystem
- Shares same limits

Partition:
- Independent filesystem
- Independent corruption risk
- Independent mount options
- Independent encryption

Example: `/boot is separate → root filesystem corrupt → boot still works → system is recoverable`

That’s HUGE.

## other  version
### 1 One Disk, Many Layers (Big Picture)

Think in **layers**, not folders.
```
Physical Disk
 └─ Partition
     └─ (Optional) LUKS Encryption
         └─ (Optional) LVM
             └─ Filesystems
                 ├─ /        (root)
                 ├─ /home
                 ├─ /var
                 ├─ swap
```

`/` is the **top of the directory tree**, not the disk itself.
### 2 What `/` (root filesystem) REALLY Is

`/` is:
- The **first filesystem mounted**
- Contains the **entire OS**
- Every other filesystem is mounted _into it_

Example:
```
/
├── bin
├── boot
├── etc
├── home   ← another filesystem may be mounted here
├── var
└── usr
```

So `/home` **looks like a folder**, but it can be:
- A separate partition
- A separate LVM volume
- A separate encrypted filesystem

### 3 Are `/home`, `swap`, etc “inside” `/`?

Logically: YES
Physically:  NO
They are:
- **Mounted at directories under `/`**
- Backed by **different block devices**

Example: `lsblk`

```
nvme0n1
 └─nvme0n1p3
    └─cryptroot
       ├─ ubuntu--vg-root  → /
       ├─ ubuntu--vg-home  → /home
       └─ ubuntu--vg-swap  → [SWAP]
```

### 4 Important Mount Points (In Detail)

####  `/` (root)
- OS binaries
- Libraries
- System configs
- Required to boot

If `/` is broken → system won’t boot.
####  `/home`
- User files
- Desktop configs
- Downloads, documents

Why separate it?
- Reinstall OS without deleting user data
- Different backup strategy
- Different encryption keys (advanced)
####  `swap`
- Disk-based RAM
- Used when RAM is full
- Used for hibernation

Can be:
- Swap partition
- Swap file (Ubuntu default now)

Swap does **not mount into `/` as a directory**.
####  `/boot`
- Kernel
- initramfs
- GRUB files

Encryption note:
- `/boot` is **usually NOT encrypted**
- Needed early in boot
####  `/boot/efi`
(UEFI systems only)
- EFI System Partition (ESP)
- FAT32
- Used by firmware

Mounted at: `/boot/efi`
####  `/var` (Advanced setups)
Contains:
- Logs
- Package cache
- Databases

Separate `/var` is used on:
- Servers
- Systems with heavy logging
####  `/tmp`
Temporary files , Often **tmpfs (RAM-based)**.
### 5 Why Ubuntu Uses LVM for This

With **LVM**, these aren’t fixed partitions:

```
LUKS container
 └─ LVM Volume Group
     ├─ LV root
     ├─ LV home
     └─ LV swap
```

Advantages:
- Resize `/` or `/home` later
- Add new logical volumes
- Snapshots
### 6 Mounting: How Linux “glues” it together

Example: `mount /dev/mapper/ubuntu--vg-home /home`
This says: “Show this filesystem **at** `/home`”
Without mounting → directory is empty.
### 7 How the System Knows What to Mount

File: `/etc/fstab`
Example entry: `UUID=xxxx  /home  ext4  defaults  0  2`
Tells system:
- What device
- Where to mount
- When to check
### 8 Common Beginner Misconception (Very Important)

❌ `/home` is NOT a sub-folder on disk  
❌ `/` is NOT “the whole disk”
✅ `/` is a **mount point**  
✅ Everything else is **mounted into it**
### 9 Simple Mental Model (Remember This)

**Linux has ONE directory tree.  
Disks are plugged into that tree using mount points.**
### 10 What Ubuntu Installer Does Automatically

With **LVM + encryption**:
- Creates one encrypted container
- Inside it:
    - root LV → `/`
    - home LV → `/home`
    - swap LV → swap
	You never see this complexity unless you look.

# 1️⃣9️⃣ Password Change

In Ubuntu 20, `passwd` often worked more transparently with domain users. In Ubuntu 24, stricter policies and updated PAM/SSSD configs can prevent local password changes for domain accounts.

### 🔍 What’s happening

When you run: `passwd` on a **domain user**, Ubuntu 24 may:

- Reject the change silently or fail
- Require proper Kerberos tickets
- Or defer password changes to the domain controller (Windows side)

Meanwhile, on Windows, the password change works because it’s directly talking to Active Directory.
### ✅ Ways to fix / work around it

#### 1. Try using `kpasswd` (Kerberos-native way)

If your system uses Kerberos (very likely), use: `kpasswd`
This talks directly to the domain controller and usually works even when `passwd` doesn’t.
If this works, it’s the _correct_ method for domain users in Ubuntu 24.
#### 2. Make sure you have a valid Kerberos ticket

Before changing password:

```
kinit your-username  
klist
```

Then: `kpasswd`, If `kinit` fails, your system isn’t properly authenticated to the domain.
#### 3. Check SSSD configuration

Ubuntu 24 relies more strictly on SSSD.
Check: `sudo nano /etc/sssd/sssd.conf`
Look for: `chpass_provider = ad`
If missing, add it under your domain section:

```
[domain/your-domain]  
chpass_provider = ad
```

Then restart: `sudo systemctl restart sssd`
#### 4. Verify PAM is allowing password changes

Check: `cat /etc/pam.d/common-password`
You should see something involving: `pam_sss.so`
If not, domain password changes won’t work.

### 🔑 `kinit your-username`

This command **logs you into the domain from the terminal**.
When you run: `kinit your-username`, It will:

- Ask for your **domain password**
- Contact the domain controller (AD)
- If successful, it gives you a **Kerberos ticket**

Think of it like: “Hey domain, verify me and give me temporary access.”
That ticket is what allows you to:

- Access domain resources
- Change password using `kpasswd`
- Authenticate without re-entering password repeatedly

### 📜 `klist`

This command shows your **current Kerberos tickets**.
Run: `klist`, You’ll see something like:

```
Ticket cache: FILE:/tmp/krb5cc_1000  
Default principal: user@DOMAIN.COM  
  
Valid starting     Expires            Service principal  
04/10/26 10:00    04/10/26 20:00     krbtgt/DOMAIN.COM@DOMAIN.COM
```

#### What this means:

- ✅ You are authenticated
- ⏳ Ticket has an expiry time
- 🎫 You can now use Kerberos-based services

# 2️⃣0️⃣ AD from Ubuntu Perspective

### The Core Network Infrastructure: Active Directory and the Domain Controller

Managing individual user accounts across hundreds of separate physical computers in a corporate environment is highly inefficient. To streamline this, organizations centralize all user, group, computer, and network identities inside a unified directory service called **Active Directory (AD)**. The physical server hosting this central database and managing the domain's security policies is known as the **Domain Controller (DC)**. In your specific setup, your Ubuntu laptop (identified as `IBS-LAP-074`) is integrated into an Active Directory domain named `IMPRESSICO.LOCAL`, which relies on a Domain Controller situated at the IP address `10.60.10.3`. Operating under your corporate identity (`kodidala.kumar@IMPRESSICO.LOCAL`), your laptop queries this Domain Controller to verify your identity, inspect group memberships, or request directory details.

### Authentication and Single Sign-On: How Kerberos Secures the Network

To access network-wide resources securely without repeatedly typing your password, the network employs **Kerberos**, an authentication protocol designed to enable **Single Sign-On (SSO)**. When you log in initially, your laptop authenticates with the Domain Controller and receives a cryptographic credential called a **Ticket Granting Ticket (TGT)**. You can think of this process like entering a secure office building: you present your primary ID (your password) at the security desk once, and security hands you a company badge (the TGT). As you access various Kerberos-aware services across the domain—such as a Windows File Server, an internal web application, a Microsoft SQL Server, or a Linux SSH server—you simply present this badge to obtain service-specific tickets without having to re-enter your password.

You can view your active tickets at any time using the `klist` command, which confirms whether you are authenticated and displays your ticket expiration times. Crucially, possessing a Kerberos ticket does not automatically grant you access to every system; it simply provides trusted proof of identity that Kerberos-aware services can verify. When you successfully query an LDAP service using your ticket, your Kerberos list updates to show that in addition to the original Ticket Granting Ticket (`krbtgt`), you have acquired a specific service ticket for the LDAP directory.

### Querying the Directory: The Language of LDAP

While Kerberos handles the _authentication_ (proving who you are), **LDAP (Lightweight Directory Access Protocol)** is the standardized language and protocol used to _search_ and query the Active Directory database. If Active Directory is a massive directory database, LDAP is conceptually like a Google search engine used to ask questions, fetch user lists, or locate groups. To run these queries from the Linux terminal, admins use a utility called `ldapsearch`.

Active Directory organizes objects in a hierarchical **tree structure**, much like folders and subfolders. Because of this, LDAP paths are written using "Domain Components" or **DC** notation. For your domain, the top of the tree is written as `DC=IMPRESSICO,DC=LOCAL`. When you run a query, specifying this as the search base (using the `-b` option) tells the system to start at the root of the tree and search downwards. To keep search results focused, you use logical **filters** like `(&(objectCategory=person)(objectClass=user))`, which tell AD to return only objects that are both people and user accounts. Finally, you can specify individual **fields/columns** to retrieve, such as `sAMAccountName` (the login username) and `displayName` (the full display name), which is equivalent to displaying only those specific columns in an Excel sheet. Combining these elements with your existing Kerberos ticket (`-Y GSSAPI` utilizing SASL/GSS-SPNEGO) allows you to search the entire directory and retrieve a clean list of domain usernames and display names without ever typing your password—a prime example of Single Sign-On in action.

### Linux-to-Windows Integration: SSSD vs. Winbind

To make Ubuntu understand Windows Active Directory identities, the system must utilize a bridge daemon. The two most common integration mechanisms are **SSSD** (System Security Services Daemon) and **Winbind**.

- **SSSD** is the modern integration standard. It translates Active Directory identities so that native Linux commands can understand them. When SSSD is running, a command like `getent passwd` (which lists all users known to the OS) can query Active Directory and display domain users mixed right in with your local accounts. You can also query specific AD users on demand using `getent passwd <username>` to verify that Ubuntu is successfully communicating with Active Directory. However, depending on SSSD configuration settings, a broad command like `getent passwd` may not always list every domain user automatically.
- **Winbind** is an alternative integration daemon that utilizes Samba. When using Winbind, you can list all domain users with `wbinfo -u` and all domain groups with `wbinfo -g`, which is the closest terminal equivalent to listing all AD accounts.

Since SSSD and Winbind are completely different, mutually exclusive integration layers, running Winbind commands like `wbinfo -u` on an SSSD-configured machine will fail with errors. This does not mean Active Directory is offline; it simply means your system relies on SSSD instead of Winbind, making the `wbinfo` command irrelevant to your setup.

### Diagnostics and Graphical Administration Tools

On Windows, administrators manage directories using graphical consoles like **Active Directory Users and Computers**. While the default Ubuntu settings panel is restricted to managing local user accounts, Linux administrators can achieve a similar directory-browsing experience. For a graphical interface, you can use specialized tools like **Apache Directory Studio** or **LXAD**, which provide an LDAP GUI to browse organizational containers (or **OUs**) and directory objects.

In the terminal, the single most informative command for diagnosing your integration is `realm list`. This command instantly tells you how your Ubuntu machine is joined to the domain, whether SSSD or Winbind is driving the integration, and what configuration options are active. To troubleshoot further, administrators can inspect configuration files like `/etc/sssd/sssd.conf` (making sure to avoid leaking passwords) or `/etc/krb5.conf`, and use network utilities like `nslookup` to ensure your machine can resolve the Domain Controller over your network's DNS.

### The Linux-to-Windows Translation Layer: SSSD vs. Winbind

To understand how a Linux computer operates within a Windows-centric Active Directory environment, it is helpful to look at how operating systems manage user identities1. By default, Linux systems identify users locally through standard files like `/etc/passwd`12. Active Directory, on the other hand, relies on centralized Windows accounts1. For Ubuntu to understand and work with these domain-level accounts, it requires a specialized translation layer or bridge12. The two primary technologies that handle this translation are **SSSD (System Security Services Daemon)** and **Winbind**13.

Both SSSD and Winbind are designed to connect Linux to Active Directory, handle domain user lookups, facilitate domain logins, and operate in tandem with Kerberos3. However, they do so in different ways. **Winbind** is a service closely tied to the Samba suite, offering deep Samba integration and providing a CLI command-line tool called `wbinfo`2more_horiz. When using Winbind, commands like `wbinfo -u` and `wbinfo -g` query the daemon to list known domain users and groups respectively35.

In contrast, **SSSD** is the modern Linux standard for directory service integration3. SSSD does not use Samba's Winbind under the hood34. If your Ubuntu laptop is configured to use SSSD, running `wbinfo` commands will result in errors35. This failure does not indicate a broken connection to Active Directory; it simply means that `wbinfo` is irrelevant to your system because SSSD is managing your identity integration instead of Winbind3more_horiz. Both systems are capable of managing Kerberos, LDAP, and DNS underneath, but they use different internal routines and client tools47.

### Group Policy Objects (GPOs) and Their Limitations on Linux

In a standard Windows environment, administrators use **Group Policy Objects (GPOs)** to push registry edits, security baselines, and configuration changes to computers across the network89. When you look at how a GPO is built, it actually exists in two completely separate parts on the Domain Controller:

1. **The Active Directory Component:** Inside Active Directory, a GPO is represented as an LDAP object under a container like `CN=Policies,CN=System,DC=IMPRESSICO,DC=LOCAL`810. This object contains metadata, including the policy's name and the file path where its configurations reside11. You can search and view these AD GPO objects and their metadata using LDAP queries811.
2. **The SYSVOL Component:** The actual, physical policy files, administrative templates, and configuration scripts are stored on the Domain Controller in a shared folder called **SYSVOL** (typically accessed as a network share path like `\\IMPRESSICO.LOCAL\sysvol\IMPRESSICO.LOCAL\Policies\...`)12. The LDAP object in Active Directory references these physical files by mapping a human-readable display name to a unique identifier that matches a directory inside SYSVOL12.

A major technical distinction arises when discussing whether Ubuntu can apply these GPOs12. Windows GPOs are heavily designed around Windows-specific mechanisms like the Windows Registry, software installation paths, and desktop settings9. Because Linux does not have a Windows Registry or identical operating system directories, **Ubuntu cannot automatically parse and apply normal Windows GPO settings**912.

However, there is an exception: **SSSD can interact with AD GPO access control configurations**913. When configured to do so, SSSD can read specific security-related GPOs from Active Directory to determine host-access control—meaning it can evaluate whether a particular Active Directory user is permitted to log in or SSH into that specific Linux laptop913. SSSD uses this policy logic strictly for login authorization, without acting as a full-fledged client for typical Windows registry policies13.

### Solving Kerberos Naming Issues: IP Address vs. Hostname

When trying to execute LDAP queries against a Domain Controller using Kerberos authentication, you might encounter a GSSAPI failure such as: `GSSAPI Error: Unspecified GSS failure. Minor code may provide more information (Server not found in Kerberos database)`1014. This error does not mean your credentials are bad, nor does it represent an LDAP permissions issue1516. Instead, it highlights a fundamental characteristic of how **Kerberos** handles network security15.

Kerberos operates strictly on **Service Principal Names (SPNs)**, which are registered hostnames rather than raw network addresses1517. If you run an LDAP search command and direct it to an IP address (e.g., `ldap://10.60.10.3`), the system tries to request a Kerberos service ticket for a name associated with that IP address1017. Because the Domain Controller's Kerberos database registers its LDAP service using fully qualified domain names (FQDNs)—like `DC01.IMPRESSICO.LOCAL`—it does not recognize the raw IP address as a valid principal name1517. Consequently, the Kerberos negotiation fails before the LDAP search can even begin16.

To resolve this issue, you must first discover the Domain Controller’s true hostname by using local lookup commands like `nslookup 10.60.10.3` or `dig -x 10.60.10.3`1518. Once you have determined the FQDN returned by your network's DNS, you must replace the IP address in your LDAP host option (such as changing `-H ldap://10.60.10.3` to `-H ldap://your-dc-hostname.IMPRESSICO.LOCAL`)1519. This ensures that your system requests the correct ticket from the Kerberos Key Distribution Center (KDC), allowing the client to establish a secure, authenticated session without requiring manual password prompts1719.

### Constructing an Active Directory Query Lab on Ubuntu

Once your Kerberos tickets are active (which you can verify using `klist`) and your Domain Controller's hostname is resolved, you can construct `ldapsearch` commands to retrieve a wealth of information from Active Directory17more_horiz. Every command follows a uniform pattern2223: $$\text{ldapsearch} \rightarrow \text{Server Flag (-H)} \rightarrow \text{Search Base (-b)} \rightarrow \text{Authentication (-Y GSSAPI)} \rightarrow \text{Logical Filter} \rightarrow \text{Target Attributes}$$
Using this syntax, you can target specific administrative details across your domain:

- **Inspecting Your Own Account Details:** Running a targeted query for your username (e.g., using the filter `(sAMAccountName=kodidala.kumar)`) retrieves your complete Active Directory record2425. Rather than dumping every single attribute, you can append specific attributes to your command to display only your **Distinguished Name (DN)**, display name, and login credentials25. The DN represents your unique, complete path within the hierarchical directory tree25.
- **Searching for Disabled User Accounts:** Active Directory tracks account state flags using a multi-bit integer attribute called **userAccountControl**26. One specific bit in this integer represents the `ACCOUNTDISABLE` flag (which has a hexadecimal value of `0x0002` or decimal `2`)26. To ask Active Directory to find users with this flag enabled, you utilize a bitwise matching filter: `(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))`26.
- **Locating Locked Accounts:** Locked-out accounts (usually resulting from too many failed login attempts) are flagged via the `lockoutTime` attribute or checked via computed lockout states27. You can query active lockouts by using the bitwise filter `(userAccountControl:1.2.840.113556.1.4.803:=16)`, which isolates the lockout flag (decimal `16`)27. If the search returns no objects, it simply means there are currently no locked-out user accounts in that part of the directory27.
- **Targeting specific OUs:** By default, directory searches start at the root of the domain (`DC=IMPRESSICO,DC=LOCAL`) and search downward28. If you only want to see users in a specific department, such as IT Support, you must modify your search base option (`-b`) to target that specific **Organizational Unit (OU)**29. To find your available OU paths, you can first run a broad query filtering for `(objectClass=organizationalUnit)`29.
- **Analyzing Security Groups and Membership:** Active Directory defines groups as distinct objects29. You can list all domain groups using the filter `(objectClass=group)`29. When investigating who belongs to a group, LDAP does not return basic text names; it returns the complete Distinguished Name of every member inside the `member` attribute30. To perform a reverse lookup and see every group a specific user belongs to, you can query using the filter `(member=CN=Kumar...,OU=...,DC=...)`3031.
- **Auditing Domain Computers:** Computers are also represented as individual security objects in Active Directory31. By running a query with the filter `(objectCategory=computer)` or `(objectClass=computer)`, you can retrieve information on every machine joined to the domain2231.

You can extract specific columns of data by specifying attributes at the end of your query2231. Active Directory maps system properties to specific LDAP attributes:

| Information Target                 | Active Directory LDAP Attribute |
| ---------------------------------- | ------------------------------- |
| **Computer Name**                  | `name` or `cn`31                |
| **Operating System**               | `operatingSystem`31             |
| **Distinguished Name**             | `distinguishedName`31           |
| **Object Creation Date**           | `whenCreated`31                 |
| **Last Modified Date**             | `whenChanged`31                 |
| **OS Version**                     | `operatingSystemVersion`31      |
| **DNS Hostname**                   | `dNSHostName`31                 |
| **Last Logon Timestamp**           | `lastLogonTimestamp`31          |
| **Account Control Flags**          | `userAccountControl`31          |
| **When password was last changed** | `pwdLastSet`                    |
| **Account lockout information**    | `lockoutTime`                   |
| **Bad password attempts**          | `badPwdCount`                   |
| **Account expiration**             | `accountExpires`                |
| **Groups the user belongs to**     | `memberOf`                      |
| **User's manager**                 | `manager`                       |
| **Unique AD object ID**            | `objectGUID`                    |
| **Department**                     | `department`                    |
| **Job title**                      | `title`                         |
| **Username**                       | `sAMAccountName`                |
| **Full name**                      | `displayName`                   |
| **Email**                          | `mail`                          |

By executing a structured search using these attributes (e.g., appending `name operatingSystem distinguishedName` to your computer query), you can format clear, readable reports on your network's physical assets directly from your Ubuntu terminal22.

```
KRB5_CONFIG=<(printf '[libdefaults]\ndefault_realm = IMPRESSICO.LOCAL\nrdns = false\ndns_canonicalize_hostname = false\n') ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(objectClass=user)" sAMAccountName
```

#### How Linux Talks to Active Directory

To interact with AD from a Linux terminal, you use two primary protocols:

1. **LDAP (Lightweight Directory Access Protocol)**: The language used to search and modify objects in the AD database. The tool `ldapsearch` is used for **reading** data, while `ldapmodify` is used for **writing** or editing data.
2. **Kerberos (GSSAPI)**: The authentication protocol that lets you log in securely. By using the `-Y GSSAPI` flag in `ldapsearch`, you are telling Linux to use your active Kerberos login ticket (which you get from running `kinit`) to authenticate to AD automatically without typing a password in plain text

#### Demystifying the Confusing LDAP Filters

In your log, you encountered this filter: `"(&(objectClass=computer)(userAccountControl:1.2.840.113556.1.4.803:=2))"`
Here is exactly what that filter means, broken down step-by-step:

- **&** **(AND Operator)**: This specifies that **both** conditions inside the parentheses must be true.
- **objectClass=computer**: This tells AD to only look for computer accounts, ignoring users, groups, or organizational units.
- **userAccountControl**: This is an integer attribute in AD that acts as a collection of binary "on/off" switches (known as bit flags) representing the status of an account (such as whether it is disabled, locked, or requires a password change).
- **The Number** **2**: This is the specific binary bit that represents `ACCOUNTDISABLE` (Account is Disabled).
- **The Mysterious OID** **1.2.840.113556.1.4.803**: This is an LDAP matching rule called **LDAP_MATCHING_RULE_BIT_AND**.
    - _Why can't we just write_ _userAccountControl=2__?_ Because an account can have multiple flags turned on at the same time. For example, if an account is disabled (`2`) _and_ its password never expires (`65536`), AD stores the sum `65538`.
    - If you searched for `userAccountControl=2`, you would completely miss this account. The matching rule performs a **bitwise AND**, which extracts only the specific "disabled" switch (bit `2`) to see if it is turned on, regardless of what other switches are active.

| Filter                           | Finds            |
| -------------------------------- | ---------------- |
| `objectClass=user`               | User objects     |
| `objectClass=computer`           | Computer objects |
| `objectClass=group`              | Groups           |
| `objectClass=organizationalUnit` | OUs              |
| `objectClass=domain`             | Domain object    |

#### Understanding `lockoutTime` & Windows FILETIME

You saw a value like `lockoutTime: 134256308026930830` and wondered how to read it1112.

- Active Directory does not store dates as simple Unix timestamps. Instead, it uses **Windows FILETIME**, which is the number of **100-nanosecond intervals that have elapsed since January 1, 1601 (UTC)**1112.
- To convert this to a normal, human-readable date, you can use Python in your Linux terminal12. The formula divides the FILETIME value by `10` because Python's `timedelta` library measures time in microseconds ($10^{-6}$ seconds), and there are ten 100-nanosecond intervals in one microsecond ($100\text{ ns} \times 10 = 1000\text{ ns} = 1\text{ }\mu\text{s}$)12.

**The Python Conversion Code**

```
python3 -c "from datetime import datetime,timezone; print(datetime(1601,1,1,tzinfo=timezone.utc)+__import__('datetime').timedelta(microseconds=134256308026930830/10))"
```

This converts the huge number into the actual date: **August 28, 2026, at 12:06:42 PM UTC**1213. Note that a `lockoutTime` of `0` means the account is not locked
#### The Kerberos / SASL Troubleshooting Journey

Your terminal session showed a common, frustrating issue: your queries worked fine, and then suddenly failed with `Server not found in Kerberos database`.

Here is what went wrong and why:

1. **The Hostname Canonicalization Loop**: When you type `SHAIL.impressico.local`, Kerberos and OpenLDAP try to be helpful. They query DNS, get the server's IP address, and then perform a reverse lookup to find the "canonical" hostname.
2. **The Error**: If DNS reverses this IP to a generic address (like an IP string `10.60.10.3`), Kerberos requests a ticket for `ldap/10.60.10.3` instead of `ldap/SHAIL.impressico.local`. Because Active Directory does not have an entry for that IP name in its Kerberos database, it throws the `Server not found` error.

The Fixes:

- **The** **/etc/krb5.conf** **Fix**: Adding these two lines under the `[libdefaults]` section of `/etc/krb5.conf` tells the Kerberos library to stop doing reverse DNS lookups and stop canonicalizing hostnames:

```
rdns = false
dns_canonicalize_hostname = false
```

- **The** **-N** **Flag**: Sometimes the OpenLDAP client library itself tries to canonicalize the hostname during GSSAPI bindings. Adding the **-N** **flag** to your `ldapsearch` command tells it to bypass this step entirely and use the exact host you provided:

```
ldapsearch -N -H ldap://SHAIL.impressico.local -Y GSSAPI ...
```

#### Writing and Resetting (The Secure LDAP Rule)

Your notes point out that while `ldapsearch` works over plain LDAP, Active Directory **will reject password changes or resets over unencrypted connections (****ldap://****)**.

- AD stores passwords in a special attribute called `unicodePwd`.
- To change or reset passwords, AD enforces a security rule requiring **LDAPS (Secure LDAP over SSL/TLS, typically port 636)** or a StartTLS connection. You cannot use a plain connection for password modifications

#### Search for specific User

```
KRB5_CONFIG=<(printf '[libdefaults]\ndefault_realm = IMPRESSICO.LOCAL\nrdns = false\ndns_canonicalize_hostname = false\n') ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -N -b "DC=IMPRESSICO,DC=LOCAL" "(sAMAccountName=kodidala.kumar)" whenCreated whenChanged lastLogonTimestamp lastLogon pwdLastSet badPwdCount lockoutTime badPasswordTime logonCount memberOf
```

### Modify AD Account with ldapmodify

`ldapmodify` is basically the **write counterpart to `ldapsearch`**.

- `ldapsearch` → read AD
- `ldapmodify` → modify AD attributes
- `ldapadd` → create an object
- `ldapdelete` → delete an object

```
printf 'dn: CN=Konduru Manasa Veena,OU=Team-Trainee,DC=impressico,DC=local\nchangetype: modify\nreplace: unicodePwd\nunicodePwd:: '$(printf '"Support@2026@OCT"' | iconv -f UTF-8 -t UTF-16LE | base64 -w0)'\n' | KRB5_CONFIG=<(printf '[libdefaults]\ndefault_realm = IMPRESSICO.LOCAL\nrdns = false\ndns_canonicalize_hostname = false\n') ldapmodify -H ldaps://SHAIL.impressico.local -Y GSSAPI -N
```

With your Kerberos setup, you can use `ldapmodify -Y GSSAPI`.
#### 1. Basic `ldapmodify` structure

The general format is: `ldapmodify -H ldap://SHAIL.impressico.local -Y GSSAPI`
Then provide an LDIF operation:

```
dn: <distinguished name>
changetype: modify
replace: <attribute>
<attribute>: <new value>
```

For example:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
replace: department
department: IT Support
```

For example:

```
dn: uid=username,ou=users,dc=example,dc=com
changetype: modify
replace: userPassword
userPassword: NewPassword123
```

This means: Find John Doe and replace his `department` with `IT Support`.
#### 2. `add`

Use `add` when the attribute doesn't currently have the value you want.

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
add: telephoneNumber
telephoneNumber: +91-1234567890
```

Command: `ldapmodify -H ldap://SHAIL.impressico.local -Y GSSAPI`
Paste the LDIF and press: `Ctrl+D`
#### 3. `replace`

This replaces the existing value.

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
replace: department
department: Infrastructure
```

This is useful for things like:

```
department
title
description
telephoneNumber
mobile
physicalDeliveryOfficeName
manager
```

assuming you have permission to modify them.

#### 4. `delete`

Remove an attribute entirely:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
delete: telephoneNumber
```

Or remove a **specific value**:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
delete: memberOf
memberOf: CN=SomeGroup,OU=Groups,DC=IMPRESSICO,DC=LOCAL
```

However, **don't manually modify `memberOf`** in normal AD administration. Group membership is normally changed through the group's `member` attribute.
#### 5. Multiple modifications in one operation

You can modify several attributes at once:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify

replace: department
department: IT Support
-
replace: title
title: Desktop Support Engineer
-
replace: telephoneNumber
telephoneNumber: +91-1234567890
```

The `-` separates individual modifications.
#### 6. Unlock an account

This is one of the useful AD-specific modifications.

```
dn: <USER-DN>
changetype: modify
replace: lockoutTime
lockoutTime: 0
```

For example:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
replace: lockoutTime
lockoutTime: 0
```

That clears the lockout timestamp. **Your account needs appropriate AD permissions.**
#### 7. Enable a disabled account

This is more interesting because `userAccountControl` is a **bit field**.
Don't blindly do:

```
replace: userAccountControl
userAccountControl: 512
```

because that can accidentally remove other account flags.  Instead, the safest administrative approach is usually to use an AD-aware tool such as PowerShell/AD tools, or calculate the existing flags correctly before modifying them.
For example, normal enabled user accounts commonly have: `512 = NORMAL_ACCOUNT`
while: `514 = NORMAL_ACCOUNT + ACCOUNTDISABLE` , So:

```
512 → enabled
514 → disabled
```

But real accounts can have additional flags.
#### 8. Disable an account

Same issue in reverse: you shouldn't blindly overwrite `userAccountControl`.
You need to preserve the existing flags and set the `ACCOUNTDISABLE` bit.
The relevant bit is: 2
Conceptually: `new UAC = old UAC OR 2`
To enable: `new UAC = old UAC AND NOT 2`
This is why `userAccountControl` is more complicated than ordinary LDAP attributes.
#### 9. Change group membership

Suppose you have: `CN=IT Support,OU=Groups,DC=IMPRESSICO,DC=LOCAL`
and want to add a user.
Modify the **group**, not `memberOf` on the user:

```
dn: CN=IT Support,OU=Groups,DC=IMPRESSICO,DC=LOCAL
changetype: modify
add: member
member: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
```

Remove them:

```
dn: CN=IT Support,OU=Groups,DC=IMPRESSICO,DC=LOCAL
changetype: modify
delete: member
member: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
```

This is a very important AD concept:

```
User
  │
  │ memberOf
  ▼
Group

But when modifying membership:

Group
  │
  │ member
  ▼
User
```

#### 10. Resetting a password

Password modification is a special case. AD's password attribute is: `unicodePwd`
You generally need **LDAPS (`ldaps://`) or another protected channel**.
You cannot safely do: `ldapmodify -H ldap://...` and send a password over ordinary LDAP.
An administrative password reset also requires appropriate privileges.
So don't treat `unicodePwd` like:

```
department
telephoneNumber
description
```

#### 11. Moving an object

`ldapmodify` can also perform a **Modify DN** operation.
For example:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modrdn
newrdn: CN=John Smith
deleteoldrdn: 1
```

That changes the object's RDN/name. You can also move objects between OUs using `newsuperior`, subject to permissions.

#### 12. What can you modify?

Potentially **many AD attributes**, but there are three important categories.
##### Normal editable attributes

Examples:

```
displayName
description
department
title
telephoneNumber
mobile
mail
physicalDeliveryOfficeName
manager
streetAddress
l
st
postalCode
```

Whether you can modify them depends on your permissions.
##### Administrative attributes

Examples:

```
lockoutTime
userAccountControl
member
group membership
accountExpires
pwdLastSet
```

These have more consequences and often require elevated permissions or special handling.
##### Sensitive/protected attributes

Examples:

```
unicodePwd
objectSid
objectGUID
sAMAccountName
servicePrincipalName
```

These shouldn't be modified casually. Some are system-controlled, protected, or require special protocols/permissions.

#### 13. First thing you should always do

Before modifying an object, get its DN and current attributes.

For example:

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI \
-b "DC=IMPRESSICO,DC=LOCAL" \
"(sAMAccountName=USERNAME)" \
distinguishedName userAccountControl lockoutTime department memberOf
```

You'll get something like:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
userAccountControl: 512
lockoutTime: 0
department: IT Support
```

Then you know **exactly which object you're modifying and what its current state is**.

## AD accounts/computers configured for Kerberos delegation

LDAP can find them by querying `userAccountControl` and `msDS-Allowed*` attributes.
#### 1. Unconstrained delegation

Find user accounts with the `TRUSTED_FOR_DELEGATION` bit (`524288`):

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=524288))" sAMAccountName distinguishedName userAccountControl
```

For **computers**:

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(&(objectClass=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))" name dNSHostName userAccountControl
```
#### 2. Constrained delegation

Look for: `msDS-AllowedToDelegateTo`

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(msDS-AllowedToDelegateTo=*)" sAMAccountName name msDS-AllowedToDelegateTo
```

This tells you **which service principals the account is allowed to delegate to**.
#### 3. Resource-based constrained delegation (RBCD)

Look for: `msDS-AllowedToActOnBehalfOfOtherIdentity`

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(msDS-AllowedToActOnBehalfOfOtherIdentity=*)" name dNSHostName
```

This is particularly important because the attribute is attached to the **target computer/service**, and its value is an access-control descriptor rather than a simple username.
#### 4. Find privileged/admin groups

If you mean **who are the actual domain administrators**, start with: `ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(cn=Domain Admins)" member`

Other important built-in groups include:

```
Domain Admins
Enterprise Admins
Administrators
Schema Admins
Account Operators
Server Operators
Backup Operators
Print Operators
DNS Admins
Group Policy Creator Owners
```

For example:

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(|(cn=Domain Admins)(cn=Enterprise Admins)(cn=Administrators)(cn=Schema Admins)(cn=Account Operators)(cn=Server Operators)(cn=Backup Operators)(cn=DNS Admins))" cn member
```

#### One important distinction

**Delegation ≠ administrator.**
For example:

```
Domain Admin
    ↓
Administrative privilege

Kerberos delegation
    ↓
Ability for a service/account to act on behalf of users
```

An account can have delegation configured without being a Domain Admin, and a Domain Admin doesn't automatically mean the account is configured for delegation.

# RootDSE, DNS, and SMB Discovery

The three groups of commands you are using answer three different questions:

```
1. LDAP RootDSE  →  "What AD environment am I connected to?"
2. DNS SRV       →  "Where are the AD services?"
3. SMB           →  "What shares does this server expose?"
```

The overall picture is:

```
                    Active Directory
                           │
              ┌────────────┼────────────┐
              │            │            │
             LDAP          DNS          SMB
              │            │            │
          AD structure   Services      Shares
              │            │            │
          RootDSE       SRV records   NETLOGON
          Users         DCs           SYSVOL
          Computers     LDAP          IPC$
          Groups        Kerberos
          OUs           Global Catalog
```

## 1. LDAP RootDSE discovery

Your command:

```
KRB5_CONFIG=<(printf '[libdefaults]\ndefault_realm = IMPRESSICO.LOCAL\nrdns = false\ndns_canonicalize_hostname = false\n') ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -N -b "" -s base "(objectClass=*)" defaultNamingContext configurationNamingContext rootDomainNamingContext dnsHostName serverName namingContexts
```

This looks complicated because several commands/options are combined together. Let's break it apart.

### 1.1 `KRB5_CONFIG=...`

```
KRB5_CONFIG=<(printf '[libdefaults]
default_realm = IMPRESSICO.LOCAL
rdns = false
dns_canonicalize_hostname = false
')
```

`KRB5_CONFIG` tells Kerberos: "For this command, use this Kerberos configuration."
Normally Kerberos configuration comes from: `/etc/krb5.conf`
But you're temporarily providing a small configuration instead.

#### `default_realm`

```
default_realm = IMPRESSICO.LOCAL
```

Your AD/Kerberos realm is: `IMPRESSICO.LOCAL`
Kerberos realms are conventionally written in uppercase. Your AD DNS domain is: `impressico.local` So:

```
DNS domain:       impressico.local
Kerberos realm:   IMPRESSICO.LOCAL
```

They normally correspond to the same AD environment.
#### `rdns = false`

`rdns = false` This tells Kerberos: Don't perform reverse DNS lookups when determining the identity of a host.
This is useful in your environment because you don't want Kerberos unnecessarily doing:

```
hostname
   ↓
IP
   ↓
reverse DNS
   ↓
hostname
```

#### `dns_canonicalize_hostname = false`

This tells Kerberos: Don't automatically change/canonicalize the hostname you gave it.
You're explicitly connecting to: `SHAIL.impressico.local`
so Kerberos should use that hostname rather than trying to transform it.

### 1.2 `ldapsearch`

`ldapsearch` This is an LDAP client. LDAP is the protocol you use from Ubuntu to communicate with Active Directory's directory service.
Think:

```
Ubuntu
  │
  │ LDAP
  ▼
Domain Controller
  │
  ▼
Active Directory
```

### 1.3 `-H ldap://SHAIL.impressico.local`

```
-H ldap://SHAIL.impressico.local
```

`-H` specifies the LDAP server.
You're saying: Connect to the LDAP service on SHAIL.
The protocol is: `ldap://`
Normally LDAP uses: `TCP 389`
So conceptually:

```
Ubuntu
  │
  │ TCP 389
  ▼
SHAIL.impressico.local
  │
  ▼
Active Directory LDAP
```

### 1.4 `-Y GSSAPI`

`-Y GSSAPI` This specifies the SASL authentication mechanism.
GSSAPI allows LDAP to use Kerberos authentication.
So instead of: `username + password`
you're using your existing Kerberos credentials.
Your previous output:

```
SASL/GSSAPI authentication started
SASL username: kodidala.kumar@IMPRESSICO.LOCAL
SASL SSF: 256
SASL data security layer installed.
```

means Kerberos/GSSAPI authentication succeeded.
### 1.5 `-N`

`-N` This tells `ldapsearch` not to use reverse-DNS canonicalization for the SASL hostname.
It's particularly useful in your setup because you're explicitly controlling the hostname.
### 1.6 `-b ""`

This is very important.
`-b ""` means: Use an empty LDAP search base.
Normally you might search: `DC=impressico,DC=local` for users or computers.
But RootDSE is special.
RootDSE is located at the **top/root of the LDAP server**, so you use: `-b ""`
Think of it as:

```
LDAP server
└── RootDSE
```

rather than:

```
DC=impressico,DC=local
└── Users
└── Computers
└── ...
```

### 1.7 `-s base`

`-s base` means: Search only the exact object specified by the search base.
Since: `-b ""` and: `-s base` together mean: "Give me information about the LDAP server's RootDSE itself."
You're **not searching all users/computers**.

### 1.8 `(objectClass=*)`

`"(objectClass=*)"` This is the LDAP search filter.
`*` means essentially:  Match an object with an objectClass.
Because you're querying RootDSE with `-s base`, you're asking for the RootDSE object.

### 1.9 The attributes you're requesting

At the end you have:

```
defaultNamingContext
configurationNamingContext
rootDomainNamingContext
dnsHostName
serverName
namingContexts
```

These are the important pieces.

#### `defaultNamingContext`

Your output was: `defaultNamingContext: DC=impressico,DC=local`
This tells you: What is the default AD domain partition?
Your domain is: `DC=impressico,DC=local`
In human terms: `impressico.local`
 So when you want to search normal AD objects, you can use: `-b "DC=impressico,DC=local"`
For example:

```
DC=impressico,DC=local
├── CN=Users
├── CN=Computers
├── OU=...
└── ...
```

#### `configurationNamingContext`

You got:

```
configurationNamingContext:
CN=Configuration,DC=impressico,DC=local
```

This is the **Configuration partition**.
It contains forest-wide AD configuration information. Conceptually:

```
Forest
│
├── Domain partition
│    └── impressico.local
│
├── Configuration partition
│    └── CN=Configuration,...
│
└── Schema partition
```

The Configuration partition contains information such as:

- Sites
- Domain Controllers
- Replication configuration
- Services
- Some infrastructure configuration
- DFS-related configuration in environments where it is stored there

That's why you previously used: `-b "CN=Configuration,DC=impressico,DC=local"` for DFS discovery.

#### `rootDomainNamingContext`

You got:

```
rootDomainNamingContext:
DC=impressico,DC=local
```

This tells you the **root domain of the AD forest**.
In your environment the forest root is: `impressico.local`
If an organization had something like:

```
corp.example.com
child.corp.example.com
```

the root domain might be: `corp.example.com`
while the current domain could be: `child.corp.example.com`
In your environment they're currently the same.

#### `dnsHostName`

You got: `dnsHostName: SHAIL.impressico.local`
This is the DNS hostname of the server you're connected to.
So:

```
LDAP server
    ↓
SHAIL.impressico.local
```

And you already know from DNS that:

```
SHAIL.impressico.local
        ↓
    10.60.10.3
```

#### `serverName`

You got something similar to:

```
serverName:
CN=SHAIL,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=impressico,DC=local
```

This is an **AD Distinguished Name (DN)** identifying SHAIL's server object inside the Configuration partition. Break it down:

```
CN=SHAIL
   ↓
CN=Servers
   ↓
CN=Default-First-Site-Name
   ↓
CN=Sites
   ↓
CN=Configuration
   ↓
DC=impressico
DC=local
```

The interesting part for now is:

```
SHAIL
   │
   └── Server object
        │
        └── AD Site
```

This is more infrastructure information than you need for basic resource discovery, but it becomes useful when learning AD topology.

### `namingContexts`

This is one of the most useful RootDSE attributes. You saw:

```
namingContexts:
DC=impressico,DC=local

CN=Configuration,DC=impressico,DC=local

CN=Schema,CN=Configuration,DC=impressico,DC=local

DC=ForestDnsZones,DC=impressico,DC=local

DC=DomainDnsZones,DC=impressico,DC=local
```

This tells you: What LDAP directory partitions are available from this server?
Think of them as different databases/sections of the AD directory.

### Your environment

```
AD LDAP
│
├── DC=impressico,DC=local
│     → Normal domain objects
│
├── CN=Configuration,...
│     → Forest configuration
│
├── CN=Schema,...
│     → AD object/schema definitions
│
├── DC=ForestDnsZones,...
│     → Forest-wide DNS data
│
└── DC=DomainDnsZones,...
      → Domain DNS data
```

This is why RootDSE discovery is so useful. You don't have to hardcode these values.
Your script can ask: "What naming contexts does this AD server provide?"
and then use the returned values.

## 2. DNS SRV discovery

Now your three commands:

```
dig @SHAIL.impressico.local _gc._tcp.impressico.local SRV
dig @SHAIL.impressico.local _ldap._tcp.impressico.local SRV
dig @SHAIL.impressico.local _ldap._tcp.dc._msdcs.impressico.local SRV
```

These are **service discovery queries**.
Instead of asking: "What IP belongs to this hostname?"
you're asking: "Where is this particular AD service?"
### What is an SRV record?

SRV = **Service Record**.
It tells a client:

```
Service
   ↓
Protocol
   ↓
Domain
   ↓
Which server?
   ↓
Which port?
```

For example: `_ldap._tcp.impressico.local`
means approximately:

```
LDAP
 ↓
TCP
 ↓
impressico.local
```

### 2.1 `_ldap._tcp.impressico.local`

Your earlier result contained:

```
_ldap._tcp.impressico.local. 600 IN SRV 0 100 389 nandi.impressico.local.
_ldap._tcp.impressico.local. 600 IN SRV 0 100 389 gana.impressico.local.
_ldap._tcp.impressico.local. 600 IN SRV 0 100 389 shail.impressico.local.
```

Let's understand one line: `0 100 389 shail.impressico.local.`
The important fields are:

```
priority   weight   port   target
   0        100      389   shail.impressico.local
```

So:

```
LDAP service
     ↓
TCP
     ↓
Port 389
     ↓
SHAIL
```

Your DNS server told you that these hosts advertise LDAP:

```
NANDI → 389
GANA  → 389
SHAIL → 389
```

This is **service discovery**. You didn't hardcode those servers.

### 2.2 `_ldap._tcp.dc._msdcs.impressico.local`

This is more specifically related to **Domain Controllers**.

```
_ldap
  ↓
LDAP

_tcp
  ↓
TCP

dc
  ↓
Domain Controller

_msdcs
  ↓
Microsoft Domain Controller DNS discovery namespace
```

So this query is useful when you specifically want to discover **domain controllers advertising LDAP**.
This is more precise for your AD discovery script than simply asking: `_ldap._tcp.impressico.local`

### 2.3 `_gc._tcp.impressico.local`

GC means: **Global Catalog**
A Global Catalog is an AD service that provides a searchable representation of objects across the forest.
The normal Global Catalog port is: `3268`
and secure GC commonly uses: `3269`
So: `dig @SHAIL.impressico.local _gc._tcp.impressico.local SRV`
asks: Which servers provide the Global Catalog service?
Conceptually:

```
AD forest
   │
   └── Global Catalog
          │
          ├── Server A : 3268
          └── Server B : 3268
```

## 3. SMB discovery

Your command: `smbclient -L //SHAIL.impressico.local -U 'impressico\kodidala.kumar'`
is different from LDAP and DNS.
Here you're talking to **SMB**, Microsoft's network file/remote-resource protocol.

```
Ubuntu
   │
   │ SMB
   ▼
SHAIL
```

### `smbclient`

`smbclient` is an Ubuntu/Linux command-line SMB client.
It lets you interact with Windows SMB servers.

#### `-L`

`-L` means: List the shares available on the SMB server.
So: `smbclient -L //SHAIL.impressico.local ...`
means: Ask SHAIL to list its SMB shares.

#### `//SHAIL.impressico.local`

This is the SMB server.
The syntax is: `//SERVER/SHARE`
But because you used: `//SHAIL.impressico.local`
without a share name, you're asking for the **server's share list**.

#### `-U`

`-U 'impressico\kodidala.kumar'` means: Authenticate as this user.
Your username is: `kodidala.kumar`
and the domain is: `IMPRESSICO`
So you're effectively saying:

```
Domain:   IMPRESSICO
User:     kodidala.kumar
```

Then SMB asks for your password.

### Your result

You got:

```
Sharename       Type      Comment
---------       ----      -------
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
SYSVOL          Disk      Logon server share
```

Let's decode that.

#### `IPC$`

IPC = **Inter-Process Communication**.
It's a special SMB share used for communication with Windows services.
It isn't a normal file-storage share.

```
IPC$
 ↓
SMB communication/infrastructure
```

#### `NETLOGON`

`NETLOGON` is a special domain-controller share used for domain-related operations, including logon-related scripts/files.

```
NETLOGON
   ↓
Domain infrastructure
```

#### `SYSVOL`

`SYSVOL` is another special domain-controller share.
It contains replicated domain policy/configuration files, including Group Policy-related files.

```
SYSVOL
   ↓
Domain policy/configuration files
```

So your SHAIL result:

```
SHAIL
 │
 └── SMB
      │
      ├── IPC$
      ├── NETLOGON
      └── SYSVOL
```

is exactly what you'd expect to see when querying a domain controller.
## Putting all three technologies together

This is the key section for your **"AD from Ubuntu Perspective"** notes.

### LDAP

LDAP answers: **What objects and AD structure exist?**

```
LDAP
 │
 ├── Users
 ├── Groups
 ├── Computers
 ├── OUs
 ├── Printers
 └── AD configuration
```

Example: `ldapsearch ...`

### DNS

DNS answers: **Where are the AD services and machines?**

```
DNS
 │
 ├── A records
 │      hostname → IP
 │
 ├── PTR records
 │      IP → hostname
 │
 └── SRV records
        service → server + port
```

Example: `dig @SHAIL.impressico.local _ldap._tcp.dc._msdcs.impressico.local SRV`
### SMB

SMB answers: **What network shares/resources does this SMB server expose to me?**

```
SMB
 │
 └── Server
       │
       ├── IPC$
       ├── NETLOGON
       ├── SYSVOL
       └── Other shares
```

Example: `smbclient -L //SHAIL.impressico.local -U 'impressico\kodidala.kumar'`
## The complete AD discovery workflow

This is the mental model I recommend putting at the top of your notes:

```
                    Ubuntu
                       │
                       │
               ┌───────┴────────┐
               │                │
             Kerberos          DNS
               │                │
          Authentication    Service discovery
               │                │
               └───────┬────────┘
                       │
                       ▼
                      LDAP
                       │
             "What is in AD?"
                       │
          ┌────────────┼────────────┐
          │            │            │
       Users       Computers     Printers
                       │
                       ▼
                  DNS hostname
                       │
                       ▼
                     IP
                       │
                       ▼
                      SMB
                       │
              "What shares exist?"
                       │
          ┌────────────┼────────────┐
          │            │            │
        Shares       SYSVOL       NETLOGON
                       │
                       ▼
                      DFS
```

# 2️⃣1️⃣ Create a Bootable USB Drive Using `dd` on Ubuntu

Find your USB drive : `lsblk`
Unmount the USB : `sudo umount /dev/sdb1`
Write the ISO to the USB : `sudo dd if=/path/to/file.iso of=/dev/sdb bs=4M status=progress conv=fsync`
Example: `sudo dd if=ubuntu.iso of=/dev/sdb bs=4M status=progress conv=fsync`
Wait until it finishes, Then run: `sync`
Safely eject : `sudo eject /dev/sdb
**Important:** Replace `/dev/`

# 2️⃣2️⃣ CHROOT and Storage not found issue
### Understanding Your Ubuntu Architecture

How your storage hierarchy is structured:

```
/dev/nvme0n1 (Physical SSD)
 ├── nvme0n1p1 (vfat)          → Mounted at /boot/efi (UEFI boot files)
 ├── nvme0n1p2 (ext4)          → Mounted at /boot (Linux Kernels &amp; initramfs)
 └── nvme0n1p3 (crypto_LUKS)   → Encrypted partition
        └── cryptroot          → Decrypted LUKS device mapper
              └── ubuntu-vg    → LVM Volume Group
                    └── ubuntu-lv → Mounted at / (Root filesystem)

```

In a healthy system, Linux boots in this order:

1. The **UEFI motherboard firmware** loads the bootloader from the **EFI partition** (`nvme0n1p1`).
2. **GRUB** loads the **Linux Kernel** (`vmlinuz`) and the **Initial RAM Filesystem** (`initramfs`) from `/boot` (`nvme0n1p2`).
3. **initramfs** loads temporary storage and encryption drivers (`dm_crypt`).
4. `initramfs` reads `/etc/crypttab` to unlock the encrypted partition (`nvme0n1p3`) as **cryptroot**.
5. **LVM** activates the volume group (`ubuntu-vg`) and logical volume (`ubuntu-lv`).
6. The system mounts the root filesystem (`ubuntu--vg-ubuntu--lv`) at `/` and boots into Ubuntu.

### The Root Cause: What Was Broken?

Your system broke due to two interrelated issues:

1. **Kernel Module Symbol Mismatch**: An interrupted update caused a version mismatch with the encryption driver (`dm_crypt: disagrees about version of symbol module_layout`). `initramfs` could not load the module required to open your encrypted drive.
2. **/etc/crypttab** **Name Mismatch**: Your system's encryption table mapped UUID `605ddf0d-c9bd-4974-806e-e082953b0cde` to `dm_crypt-0` instead of `cryptroot`. During boot, `initramfs` looked for `cryptroot`. Because it couldn't find `cryptroot`, it couldn't activate `ubuntu-vg` or find `/dev/mapper/ubuntu--vg-ubuntu--lv`, dropping you into a **BusyBox** emergency shell.

### Step-by-Step Breakdown of the Recovery Commands

#### Phase 1: Unlocking & Activating Storage from the Live USB

`sudo lsblk -f` 

- **What it does:** Displays all block devices, partition formats, UUIDs, and mount points. 
- **Why it was needed:** To identify which exact partition was the EFI system (`vfat`), `/boot` (`ext4`), and encrypted disk (`crypto_LUKS`).

`sudo cryptsetup luksOpen /dev/nvme0n1p3 cryptroot` 

- **What it does:** Uses the LUKS decryption driver to unlock partition `nvme0n1p3` with your password and maps the decrypted raw partition to `/dev/mapper/cryptroot`. 
- **Why it was needed:** Without unlocking LUKS manually, the Live USB couldn't access your files or LVM setup.

`sudo vgchange -ay` 

- **What it does:** Scans for Logical Volume Manager (LVM) volume groups and activates them (`-a` for activate, `y` for yes). 
- **Why it was needed:** Opening LUKS revealed the `ubuntu-vg` volume group. This command activated `ubuntu--vg-ubuntu--lv` under `/dev/mapper/` so it could be mounted.

#### Phase 2: Reassembling the System Hierarchy Under `/mnt`

`sudo mount /dev/mapper/ubuntu--vg-ubuntu--lv /mnt` 

- **What it does:** Mounts your main Ubuntu root filesystem onto `/mnt` inside the Live USB session.

`sudo mkdir -p /mnt/boot/efi` 

- **What it does:** Creates the directory structure `/mnt/boot/efi` if it doesn't already exist.

`sudo mount /dev/nvme0n1p1 /mnt/boot/efi` 

- **What it does:** Mounts the EFI system partition (`nvme0n1p1`) into the target root's `/boot/efi` directory.

`sudo mount /dev/nvme0n1p2 /mnt/boot` 

- **What it does:** Mounts your dedicated `/boot` partition (`nvme0n1p2`) containing kernel images (`vmlinuz`), boot configurations (`grub`), and `initramfs` files into `/mnt/boot`.

#### Phase 3: Setting Up the `chroot` Jail

```
sudo mount --rbind /dev /mnt/dev 
sudo mount --make-rslave /mnt/dev
``` 

(Repeated for /proc , /sys , /run _) 

- **What it does:** Recursively bind-mounts (`--rbind`) the Live environment's kernel API virtual filesystems (`/dev` for devices, `/proc` for processes, `/sys` for kernel parameters, `/run` for runtime data) into `/mnt`. The `--make-rslave` flag ensures mount events inside `/mnt` don't propagate back and disrupt the host Live USB system. 
- **Why it was needed:** To repair software inside your broken system, package managers like `apt` and boot generators like `update-initramfs` require direct access to real hardware devices and kernel interfaces.

`sudo chroot /mnt` 

- **What it does:** Changes the apparent root directory for the current terminal session to `/mnt`. 
- **Why it was needed:** Before this command, you were running terminal commands as the Live USB user (`ubuntu@ubuntu`). After `chroot`, your terminal operates directly inside your installed Ubuntu system as `root@ubuntu:/#`.

#### Phase 4: Repairing Packages & Handling Prompts

`dpkg --configure -a` 

- **What it does:** Scans for packages that were downloaded and unpacked but never finished configuring due to an interrupted upgrade.

`apt update` 

- **What it does:** Refreshes local package repository lists inside the chroot environment.

`apt install --reinstall linux-generic cryptsetup-initramfs lvm2` 

- **What it does:** Forces `apt` to clean reinstall the kernel (`linux-generic`), LUKS integration tools (`cryptsetup-initramfs`), and LVM binaries (`lvm2`). 
- **Why it was needed:** Reinstalls corrupted or missing driver modules (such as `dm_crypt`) that were causing version mismatch errors.

- **Choosing** **N** **at configuration prompts (e.g.,** **/etc/gdm3/custom.conf** **):** 
- **What it does:** Instructs `dpkg` to retain your existing configuration files rather than replacing them with default stock package versions. 
- **Why it was needed:** Selecting `N` preserved your existing custom display and login manager settings safely.

#### Phase 5: Fixing the Encryption Configuration Mismatch

`cat /etc/crypttab` 

- **What it does:** Reads the encryption table file that tells Ubuntu how to map encrypted UUIDs to device mapper names during boot.

`cp /etc/crypttab /etc/crypttab.backup` 

- **What it does:** Creates a backup copy of the original file prior to editing.

`printf 'cryptroot UUID=605ddf0d-c9bd-4974-806e-e082953b0cde none luks\n' > /etc/crypttab` 

- **What it does:** Overwrites `/etc/crypttab` with the corrected mapping string. 
- **Why it was needed:** The original file incorrectly named the target `dm_crypt-0`. Renaming it to **cryptroot** aligned `/etc/crypttab` with what the boot hooks and initramfs scripts expected.

#### Phase 6: Rebuilding Early-Boot Images & Bootloader

`update-initramfs -u -k all` 

- **What it does:** Rebuilds (`-u`) the initial RAM filesystem (`initrd.img`) archives located in `/boot` for all (`-k all`) installed kernels. 
- **Why it was needed:** This was the single most critical repair step. It packed the corrected `/etc/crypttab` and freshly reinstalled kernel modules (`dm_crypt`) directly into the early-boot RAM image.

`update-grub`

- **What it does:** Scans `/boot` for kernel images and rebuilds the GRUB bootloader script (`/boot/grub/grub.cfg`).

`grep -R "cryptroot" /etc/default/grub /etc/crypttab`

- **What it does:** Searches configuration files to verify that `cryptroot` references exist and match before rebooting.

#### Phase 7: Cleaning Up and Rebooting

`exit` 

- **What it does:** Exits the `chroot` jail environment and returns control back to the Live USB shell.

`sudo umount -R /mnt`

- **What it does:** Unmounts `/mnt` and all nested bind mounts (`/mnt/boot/efi`, `/mnt/boot`, `/mnt/dev`, `/mnt/proc`, `/mnt/sys`, `/mnt/run`) recursively and safely. 
- **Why it was needed:** Ensures all filesystem buffers are written to the SSD disk safely without data corruption before rebooting.

`sudo reboot`

- **What it does:** Restarts the computer so it can boot natively off the SSD.

# CHROOT other version

### 1. Boot Ubuntu Live USB

Use an Ubuntu USB and select: **Try Ubuntu**
Do **not** install Ubuntu.
### 2. Open Terminal

Run: `sudo lsblk -f`
You'll see something roughly like:

```
nvme0n1
├─nvme0n1p1  vfat
├─nvme0n1p2  ext4
└─nvme0n1p3  crypto_LUKS
```

Your encrypted partition will have: `crypto_LUKS`
Let's assume it is `/dev/nvme0n1p3`.
**Use the actual device shown by your `lsblk -f`; don't blindly use this example.**

### 3. Unlock the encrypted partition

```
sudo cryptsetup luksOpen /dev/nvme0n1p3 cryptroot
```

Enter your normal Ubuntu disk-encryption password.
Then: `sudo vgchange -ay`
You should see something like: `1 logical volume(s) in volume group "ubuntu-vg" now active`
Check: `ls /dev/mapper`
You should now see: `ubuntu--vg-ubuntu--lv`
That's the root filesystem that your broken boot was complaining about.

### 4. Mount your Ubuntu installation

```
sudo mount /dev/mapper/ubuntu--vg-ubuntu--lv /mnt
```

Then find your EFI partition: `lsblk -f`
It will normally be a small `vfat` partition, perhaps `/dev/nvme0n1p1`.
Mount it: `sudo mount /dev/nvme0n1p1 /mnt/boot/efi`
If you have a separate `/boot` partition, mount that too. If you don't, skip it.
#### IF ERROR OCCURS create the EFI mount point

Run: `sudo mkdir -p /mnt/boot/efi`
Then: `sudo mount /dev/nvme0n1p1 /mnt/boot/efi`
There should be **no error**.
#### mount your separate `/boot`

if it shows: `nvme0n1p2   ext4`
This is very likely your `/boot` partition.
Run: `sudo mount /dev/nvme0n1p2 /mnt/boot`
Then check: `ls /mnt/boot`
You should see things such as:

```
vmlinuz-...
initrd.img-...
grub
```

### 5. Prepare the chroot

Run: 

```
sudo mount --rbind /dev /mnt/dev
sudo mount --make-rslave /mnt/dev
sudo mount --rbind /proc /mnt/proc
sudo mount --make-rslave /mnt/proc
sudo mount --rbind /sys /mnt/sys
sudo mount --make-rslave /mnt/sys
sudo mount --rbind /run /mnt/run
sudo mount --make-rslave /mnt/run
```

Then: `sudo chroot /mnt`
Your prompt should change to something like: `root@ubuntu:/#`
### 6. Repair the interrupted upgrade

First: `dpkg --configure -a`
Then: `apt update`
Then reinstall the important boot/kernel packages: `apt install --reinstall linux-generic cryptsetup-initramfs lvm2`
If it asks whether to continue, answer: Y

### 7. Rebuild ALL initramfs images

This is the particularly important part given your error: `dm_crypt: disagrees about version of symbol module_layout`
Run: `update-initramfs -u -k all`
Watch for errors.
Then: `update-grub`

### 8. Exit and reboot

```
exit
```

Then: `sudo umount -R /mnt`
Then: `sudo reboot`
Remove the USB when the computer restarts.

# CHROOT other version

 **`chroot` = run a program as if a directory is the root filesystem (`/`)**

### Where `chroot` is ACTUALLY used in real life
#### Boot repair & disaster recovery (what you’re doing now)

**Most common real-world use.** Scenarios:
- System won’t boot
- Kernel panic
- Black screen after LUKS password
- Broken GRUB
- Initramfs missing

Flow: `Live USB  → mount real root  → chroot into it  → fix bootloader / packages / configs`

Real sysadmins do this **weekly**.
This is literally how:
- Ubuntu recovery guides
- Arch Wiki
- Red Hat rescue mode
#### Installing Linux without booting it (classic sysadmin trick)

You can install a full OS **without ever booting it**.
Example: `debootstrap jammy /mnt chroot /mnt`
Now `/mnt` is a fresh Ubuntu system.
Used in:
- Cloud image creation (AWS, Azure AMIs)
- Custom ISO builders
- Minimal server images

Your cloud VM OS?  
Built using **chroot**, not a GUI installer.
#### Package repair when system is half-dead

Sometimes:
- `apt` is broken
- Python is broken
- `/bin/bash` still works
- But login fails

You boot rescue mode and: `mount root chroot apt --fix-broken install`
This fixes:
- libc issues
- broken upgrades
- interrupted dist-upgrades

No reinstall needed.
#### Cross-architecture builds (very common in DevOps)

Example: Build ARM OS on x86 laptop
`chroot arm-rootfs` Combined with: `qemu-user-static`
Used in:
- Raspberry Pi images
- Embedded Linux
- Router firmware
- Android system images

This is **industry-standard**.
#### Secure build environments (before containers existed)

Before Docker:  chroot was isolation
Example: `chroot /var/chroot/build-env`
Used by:
- Package maintainers
- CI systems (old-school but still used)
- Build servers

Docker is basically:  chroot + namespaces + cgroups + magic
#### Malware analysis / forensic analysis

Security teams use chroot to:
- Inspect compromised systems
- Analyze infected disks safely

Mount disk: `mount compromised-root /mnt chroot /mnt` You can:
- Inspect startup scripts
- Disable malware
- Reset credentials

Without executing host OS.
#### Password recovery

Forgot root password? Boot Live USB: `mount root chroot passwd` Done. This works because:
- You’re “inside” the system
- Authentication files are accessible

## Where chroot is NOT used (important)

❌ Not a security sandbox (weak isolation)  
❌ Not a VM  
❌ Not a container replacement  
❌ Not safe against root escape
That’s why containers exist.
## Mental model (this will stick)
Imagine Linux like a **city**:
- `/` = city boundary
- Programs think that’s the whole world

`chroot` says: “Your world ends here now.”
Programs inside:
- Can’t see outside `/mnt`
- Think `/mnt` is `/`

But kernel is still shared.
### Why chroot is perfect for boot fixing

Because boot problems are:
- Filesystem problems
- Config problems
- Package problems

chroot gives you:
- Correct filesystem
- Correct paths
- Correct package database

Without needing a successful boot.

# 2️⃣3️⃣ TAR

To create a tar archive (tarball), the basic command is: `tar -cvf file.tar directory_or_files`
Where: -c = create, -v = verbose (optional, shows files), -f = filename
#### Examples
**Tar a directory** `tar -cvf backup.tar myfolder/`
**Tar multiple files** `tar -cvf config.tar file1 file2 file3`
**With compression**
**gzip (.tar.gz / .tgz)**: `tar -czvf backup.tar.gz myfolder/` -z = gzip compression
**bzip2 (.tar.bz2):** `tar -cjvf backup.tar.bz2 myfolder/` -j = bzip2 compression
**xz (.tar.xz):** `tar -cJvf backup.tar.xz myfolder/` -J = xz compression (best compression)
**Extracting** `tar -xvf backup.tar`
**or compressed:** `tar -xzvf backup.tar.gz`
# 2️⃣4️⃣ Compiling Source code
## Why not just install a .deb file? Why compile from source?

Installing a .deb package is the easiest way to get software on a Debian/Ubuntu system. It’s fast, convenient, and designed for normal users. But .deb packages are limited to pre-built binaries that someone else compiled for you with their own choices — their own features, libraries, optimizations, and versions. When you install from a .deb, you get exactly what the distro maintainers decided, no more, no less.

Compiling from source code gives you much more control. Software can have optional features like GPU acceleration, codec support, hardware capabilities, debugging tools, and performance flags. Distributions often disable some of these features in binary packages because of licensing issues, dependencies, patents, or size. For example, Ubuntu’s FFmpeg often ships without some codecs because of legal reasons, while compiling from source allows you to enable them.

Another big reason is optimization. When building from source, you can compile code tuned specifically for your hardware (CPU architecture, instruction sets, optimization level). Distro maintainers build for the lowest common denominator, so their packages run on many machines, not just yours. If you compile yourself, you can enable flags like -march=native to use all your CPU features.

There are also cases where software evolves faster than your distro’s repositories. For example, Ubuntu LTS might have a version from 2 years ago, while the source project releases updates monthly. Developers, testers, and advanced users compile from source to get new versions, bug fixes, or experimental features long before they land in .deb form.

Finally, source compilation is essential for developers, because they need access to symbols, debugging info, internal structure, and build logs. A .deb package hides this. Source teaches you how software is structured, how dependencies interact, how libraries are linked, and how build systems work — skills valuable for engineering and debugging in the real world.

In summary: .deb is convenience; source is control, customization, and learning. Normal users install .deb files because they just need the software to work. Developers, power users, and system integrators compile from source because they need performance, features, modern versions, and understanding.

### 1. Install C Compiler & Build Tools on Ubuntu

Before compiling anything, you need the essential compiler tools:

#### Install GCC and Build Tools

```
sudo apt update
sudo apt install build-essential
```

This installs GCC (GNU C Compiler), make, and other tools required for building software from source.
#### Check the Compiler Version

`gcc --version` This confirms GCC is installed and working.

### 2. Compile a Simple C Program

#### Example hello.c

Create a file: `nano hello.c`
Put this inside:

```
#include <stdio.h>
int main() {
    printf("Hello, World!\n");
    return 0;
}
```

#### Compile & Run

```
gcc hello.c -o hello
./hello
```

This compiles the C source file hello.c into an executable named hello, then runs it.

### 3. Compile & Install Software from Source

When you download software source code (usually .tar.gz, .zip, or from GitHub), the typical steps are:

#### Step A — Prepare Environment

Install necessary tools: `sudo apt install build-essential checkinstall git`
Extract the source: `tar -xzvf [software.tar.gz](http://software.tar.gz)`
cd software Or clone from GitHub: `git clone https://github.com/author/project.git`
`cd project`

#### Step B — Build Process 

Most open-source C/C++ software uses a 3-step build pattern:

```
./configure
make
sudo make install
```

Let’s go through them one by one.

#### 1. ./configure — Prepare the Build

#### What it does

When you run: `./configure`
it checks:

✔ your system environment  
✔ compiler availability (gcc, clang, etc.)  
✔ required libraries & dependencies  
✔ CPU architecture  
✔ paths for installation (/usr/local/bin, /usr/lib, etc.)

Then it generates a file called Makefile which contains the exact instructions for compiling.
Example Output
You might see something like:

```
checking for gcc... yes
checking for OpenSSL... no
checking prefix... /usr/local
```

If dependencies are missing, it will fail with: `configure: error: OpenSSL headers not found`
Then you fix via apt: `sudo apt install libssl-dev`

Optional Arguments
You can customize install paths: `./configure --prefix=/opt/myapp`
Or enable/disable features: `./configure --enable-ssl --disable-tests`
Key idea: ./configure = create a build plan tailored to your system

#### 2. make — Compile Source Code

Once configure finishes, run: `make`
##### What it does

make reads the generated Makefile and compiles the .c or .cpp files into binaries.

Example:

```
gcc -c file1.c
gcc -c file2.c
gcc -o program file1.o file2.o
```

##### Why use make?

Because building manually would be crazy, imagine: `gcc -Wall -O2 -std=c11 src/*.c -o myprogram`
make automates and tracks dependencies.

##### Useful Variants

Multi-core compile (faster): `make -j$(nproc)`
nproc gives number of CPU cores, so build is parallel.

#### 3. sudo make install — Install to System

After binary is built, run: `sudo make install`

##### What it does

Copies built files into system directories like:

|           |                      |
| --------- | -------------------- |
| File Type | Directory            |
| binaries  | /usr/local/bin       |
| libraries | /usr/local/lib       |
| configs   | /usr/local/etc       |
| man pages | /usr/local/share/man |

##### Example install output:

```
install myapp /usr/local/bin
install libmyapp.so /usr/local/lib
```

Now you can run: `myapp`

##### Important Note

Uninstalling is difficult because system doesn’t track installed files.

#### Safer Alternative — checkinstall

Instead of: `sudo make install`
Use: `sudo checkinstall`
This creates a .deb package so you can later uninstall: `sudo apt remove myapp`

### Putting It All Together (Typical Workflow)

Example for a software package:

```
tar -xzf myapp-1.2.3.tar.gz
cd myapp-1.2.3
./configure --prefix=/usr
make -j$(nproc)
sudo checkinstall
```

### What is CMake?

CMake itself does not compile code. It generates build files for compilers like make, ninja, MSVC, etc.
Think of it like this: `CMake → generates → Makefiles → used by make → compiles code`

### Typical CMake Build Workflow

Almost all CMake projects follow:

```
mkdir build
cd build
cmake ..
make
sudo make install
```

Let's break it down:

### 1. mkdir build && cd build — Out-of-Source Build

Projects avoid building in the source directory.

✔ Keeps source clean  
✔ Easy to delete build artifacts  
✔ You can have multiple builds

### 2. cmake .. — Configure Build

This command reads the CMakeLists.txt from the parent directory.
Example output:

```
-- Checking for module 'zlib'
-- Found zlib: /usr/lib/x86_64-linux-gnu/libz.so
-- Configuring done
-- Generating done
```

This step:

✔ checks dependencies  
✔ detects compilers  
✔ sets install directories  
✔ generates Makefile or Ninja files

### 3. make — Compile the Source

Now compile with: `make -j$(nproc)`
This executes the generated Makefile.
Try : OpenCV, Git, HandBrake, Kernel

# 2️⃣5️⃣ Runlevels
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

# 2️⃣6️⃣ Prompt String

`PS1` is the **Primary Prompt String** in Bash.  
It controls **how your terminal prompt looks** (username, host, path, colors, etc.).
`${debian_chroot:+($debian_chroot)}`
This shows the **chroot name** if you are inside a chroot environment.
Syntax explanation : `${variable:+value}`
Meaning:

- If **variable exists and is not empty → print value**
- Otherwise → print **nothing**

So: `${debian_chroot:+($debian_chroot)}`
means:

|Condition|Output|
|---|---|
|debian_chroot exists|`(name)`|
|not in chroot|nothing|

Example:  `(mychroot) user@host:~$`
Normally you won't see this unless using **chroot containers**.
`\[\033[01;32m\]` This sets **text color**. `\033` means **escape character**.  `01;32m` means:

| Code | Meaning     |
| ---- | ----------- |
| 01   | Bold        |
| 32   | Green       |
| m    | apply color |

So this sets **bold green color**.
`\u` : Displays the **username**.
Example: `koushik`
`@` : Just prints the `@` symbol.
`\h` : Shows the **hostname** (computer name).
Example: `koushik@ubuntu`
`\[\033[00m\]` : This **resets the color** back to normal.
`00m` : means **reset formatting**.
`:` Just prints a colon.
Example: `koushik@ubuntu:`
`\[\033[01;34m\]` : Sets **bold blue color**.

|Code|Meaning|
|---|---|
|01|Bold|
|34|Blue|
`\w`
Displays the **current working directory**.
Example:  `~/projects/react`
So prompt becomes:  `koushik@ubuntu:~/projects/react`
`\[\033[00m\]` : Reset color again.
`\$` Shows:

|User type|Symbol|
|---|---|
|normal user|`$`|
|root user|`#`|

Example:  `koushik@ubuntu:~$` or `root@ubuntu:/#`
What `\[ \]` does
`\[` Start of **non-printing sequence** 
`\]`End of **non-printing sequence**
So Bash **ignores the length** of whatever is inside.

# 2️⃣7️⃣ GRUB issue
### **The Big Picture**

Your system is currently suffering from **two separate problems**:

1. **GRUB is partially broken** and cannot locate its configuration automatically, dropping you into a `grub&gt;` prompt.
2. **The graphical desktop environment (GDM3 / graphics stack) is broken**, causing your screen to go black during a normal boot.

Right now, you are using a **temporary recovery door** — manually pointing GRUB to Ubuntu and forcing a **text-mode command-line boot (no GUI)**.

### **Part 1: Decoding What You Just Did (Manual Boot Steps)**

Here is the breakdown of the manual GRUB commands you used to get into the terminal:

- **ls**: Lists all disks and partitions so GRUB can see where files might live.
- **ls (hd0,gpt2)/**: Checks partition contents for Linux files like `/boot`, `vmlinuz`, and `initrd.img` to identify your Ubuntu drive.
- **set root=(hd0,gpt2)**: Explicitly tells GRUB which partition holds Ubuntu.
- **set prefix=(hd0,gpt2)/boot/grub**: Directs GRUB to its configuration files and modules.
- **insmod normal** **&** **normal**: Loads GRUB's "normal mode" module to bring back the boot menu.

When editing the boot entry (`e`):

- **linux /boot/vmlinuz-...**: Directly loads the Linux kernel file.
- **root=UUID=xxxx**: Tells the kernel where the root filesystem is located using its unique UUID.
- **nomodeset**: Disables high-performance GPU driver initialization early in boot to bypass the black screen crash.
- **systemd.unit=multi-user.target**: Forces Ubuntu to start in **text mode** rather than launching the broken graphical desktop.
- **Ctrl + X**: Boots the kernel immediately using these temporary parameters.

### **Part 2: Permanent Fix (Step-by-Step Terminal Commands)**

Once logged into the text terminal, run these commands in order to permanently repair GRUB and rebuild the graphics stack:

#### **Step 1: Fix the GRUB Bootloader**

Update GRUB's configuration and reinstall it to your drive's boot sector:

```
sudo update-grub
sudo grub-install /dev/nvme0n1
```

_(Note: Replace_ _/dev/nvme0n1_ _with your actual boot drive identifier if different)._

#### **Step 2: Stop GDM3 from Running**

Stop and disable the display manager so it won't crash while you repair it:

```
sudo systemctl disable gdm3
sudo systemctl stop gdm3
```

#### **Step 3: Reinstall Display Manager & Shell**

Reinstall corrupted GDM3 and GNOME binaries (this leaves your user files completely intact):

```
sudo apt install --reinstall gdm3 gnome-shell
sudo apt install ubuntu-drivers-common
```

#### **Step 4: Install the Recommended GPU Driver**

Detect and auto-install the correct graphics drivers for your hardware: `sudo ubuntu-drivers autoinstall`
#### **Step 5: Disable Wayland & Force Xorg**

Prevent driver crashes by turning off Wayland in GDM's settings: `sudo nano /etc/gdm3/custom.conf`
Find the line `#WaylandEnable=false` and change it to: `WaylandEnable=false`
Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

#### **Step 6: Restore Graphical Boot Target**

Re-enable GDM3 and tell systemd to boot into the graphical desktop interface automatically:

```
sudo systemctl enable gdm3
sudo systemctl set-default graphical.target
```

#### **Step 7: Reboot**

Restart your computer to test the full boot chain: `sudo reboot`

### **What to Expect**

- **Best case**: The boot chain (GRUB → Kernel → GPU Driver → Xorg → GDM → Desktop) completes and displays your regular graphical login screen.
- **If it still goes black**: Boot back into text mode (`nomodeset systemd.unit=multi-user.target`) and inspect system error logs using `journalctl -b -p err` or `journalctl -u gdm3`.

# 2️⃣8️⃣ Add user to docker group
### Step-by-step fix

1. **Create the docker group (if it doesn’t exist):** `sudo groupadd docker`
2. **Add your user to the docker group:** `sudo usermod -aG docker $USER`
3. **Apply the changes (choose one):** Log out and log back in  **OR** Run: `newgrp docker`

### Verify it worked

Run: `docker run hello-world`, If it runs without `sudo`, you're good 👍

### What’s happening behind the scenes?

When you run a command like: `docker ps`
You’re not directly running Docker itself—you’re talking to the **Docker daemon** (`dockerd`), which runs as **root**.

### Why does `sudo` work?

When you do: `sudo docker ps`
You’re basically saying: “Run this command as root”
Since the Docker daemon only accepts requests from **root or authorized users**, it works.

### So what does the `docker` group do?

Linux uses **groups** to control access to things like files and sockets.
Docker exposes a special file (a Unix socket): `/var/run/docker.sock`
Check its permissions: `ls -l /var/run/docker.sock`
You’ll see something like: `srw-rw---- 1 root docker ...`
Meaning:

- Owner: `root`
- Group: `docker`
- Permissions: read/write for owner and group

### Key idea

If your user is in the **docker group**, you can access that socket **without sudo**.
So when you run: `docker ps`
It works because:

- You → belong to `docker` group
- Socket → allows `docker` group access
- Docker daemon → accepts your request

### Why do you need to log out?

When you add yourself to a group: `sudo usermod -aG docker $USER`
Your current session doesn’t immediately “know” about it.
Logging out and back in refreshes your group memberships.


# 2️⃣9️⃣ Mount an LVM Volume Partition

Introduction: LVM is an acronym for Logical Volume Manager. LVM is a device mapper that provides logical volume management for the Linux kernel. You can access LVM partitions from an external USB hard disk or second hard disk installed in your system. 

If lvm2 not installed on your system, install it as per your Linux distro package manager utility. For example: `sudo apt install lvm2`

### How to mount LVM partition in Linux

The procedure to mount LVM partition in Linux as follows:
1. Run `vgscan` command scans all supported LVM block devices in the system for VGs.
2. Execute `vgchange` command to activate volume.
3. Type `lvs` command to get information about logical volumes.
4. Create a mount point using the `mkdir` command.
5. Mount an LVM volume under Linux using `sudo mount /dev/mapper/DEVICE /path/to/mount` command.

Let us see all steps in details to mount LVM partition or volume on Ubuntu Linux.
### How to mount an LVM volume

Type the following command to find info about LVM devices: `sudo vgscan` OR `sudo vgscan --mknodes`
Above output indicates that I have “fedora_localhost-live” LVM group. To activate it run: `sudo vgchange -ay` 
You can run the following command to list it: `sudo lvdisplay` OR `sudo lvs`
You can get good look at using the ls command: `ls -l /dev/fedora_localhost-live/`
Sample outputs:

```
total 0
lrwxrwxrwx 1 root root 7 Aug 17 15:47 home -> ../dm-1
lrwxrwxrwx 1 root root 7 Aug 17 15:47 root -> ../dm-2
lrwxrwxrwx 1 root root 7 Aug 17 15:47 swap -> ../dm-0
```

### Mount an LVM partition

Create a mount point using the mkdir command: `sudo mkdir -vp /mnt/fedora/{root,home}`
Sample outputs:

```
mkdir: created directory '/mnt/fedora/root'
mkdir: created directory '/mnt/fedora/home'
```

### Mount lvm volume using the mount command

Next, mount both home and root logical volume from LV path using the following syntax:

```
sudo mount {LV_PATH} /path/to/mount/point/
sudo mount /dev/fedora_localhost-live/home /mnt/fedora/home
sudo mount /dev/fedora_localhost-live/root /mnt/fedora/root
```

Verify it with the help of df command or grep command:

```
df -T
$ df -T | grep -i fedora
$ ls /mnt/fedora/root
$ ls /mnt/fedora/home
```

# 3️⃣0️⃣ chntpw

If you are using **Linux and have connected a Windows SSD as a secondary drive**, `chntpw` can be used to inspect or modify the **local Windows account database (SAM)**.

This applies to a Windows installation where you have authorization to recover the account.

### 1. Find the Windows SSD

Run: `lsblk -f`
You might see something like:

```
nvme0n1
├─nvme0n1p1   vfat
├─nvme0n1p2   ntfs
└─nvme0n1p3   ntfs
```

Identify the large **NTFS partition containing Windows**.
You can also check: `sudo blkid`

### 2. Mount the Windows partition

For example, if the Windows partition is `/dev/nvme0n1p3`:

```
sudo mkdir /mnt/windows
sudo mount /dev/nvme0n1p3 /mnt/windows
```

Check that it is actually Windows: `ls /mnt/windows`
You should see folders such as:

```
Windows
Users
Program Files
ProgramData
```

### 3. Locate the SAM file

The local Windows account database is normally: `/mnt/windows/Windows/System32/config/SAM`
There is also a related SYSTEM registry hive: `/mnt/windows/Windows/System32/config/SYSTEM`
You can verify: `sudo ls -l /mnt/windows/Windows/System32/config/SAM`

### 4. Use `chntpw`

First install it if necessary: `sudo apt install chntpw`
Then point it at the Windows SAM: `sudo chntpw -l /mnt/windows/Windows/System32/config/SAM`
The `-l` option lists the local accounts contained in the SAM.

For example, you may see:

```
RID          Username
--------------------------------
01f4         Administrator
03e8         Koushik
```

For an authorized recovery, `chntpw` can then be used to modify a **local account**.

### What `chntpw` can modify

When you run: `sudo chntpw -i /mnt/windows/Windows/System32/config/SAM`
you enter its interactive account-management interface. The exact menu can vary by version.
For a selected **local account**, it can generally provide options such as:

|Operation|What it means|
|---|---|
|Clear password|Removes the local account's password requirement|
|Set/change password|Allows a new password to be assigned in versions/setups that support it|
|Unlock account|Removes certain account-lock/disabled state|
|Enable account|Enables a disabled local account|
|Disable account|Disables the account|
|Promote to Administrator|Changes the account's local administrative membership/status in supported cases|
|Edit account information|Modify certain account flags/properties|
|List accounts|View accounts and their RIDs without changing them|

`chntpw` performs these through its **interactive menu** rather than giving each operation a simple one-line command.
First identify the Windows partition and SAM: `sudo mount /dev/nvme0n1p3 /mnt/windows`
Then list the accounts: `sudo chntpw -l /mnt/windows/Windows/System32/config/SAM`
For the actual account editor: `sudo chntpw -i /mnt/windows/Windows/System32/config/SAM`
You'll get a menu similar to:

```
===== chntpw Main Interactive Menu =====

1 - Edit user data and passwords
2 - Syskey status & change
3 - RecoveryConsole settings
4 - Registry editor
q - Quit
```

Choose: 1
Then select the user by its RID/username. The subsequent account menu contains the available password/account operations.

### Important correction

Some of the items in the table I gave you earlier are **not separate `chntpw` commands**. In particular, `chntpw` is primarily an **offline SAM/password/account-data editor**; it is not the normal tool for managing Windows local-group membership.

For example, for legitimate IT administration, these are normally done **from Windows itself**:

```
net user
net user username *
net user username /active:yes
net user username /active:no
net localgroup administrators username /add
net localgroup administrators username /delete
```

# 3️⃣1️⃣ Building a Tiny Linux From Scratch

### NEED

A custom-compiled Linux kernel  
    - `BusyBox` as the entire `userspace`  
    - A hand-written `init` script  
    - `Initramfs` as the root `filesystem`  
    - `UEFI` (`systemd`-boot) booting  
    - `QEMU` for testing

### 1. Reproducible Build Environment (Docker)
### 2. Compiling a Minimal Linux Kernel

Before we create any userspace environment, we need the most fundamental component: the Linux kernel itself. This section guides you through compiling a clean, upstream kernel the same codebase “Linus Torvalds” maintains free from distribution-specific patches or vendor modifications.
#### 2.1 Clone the Linux Kernel Source.

**_Why Clone from Torvalds’ Repository?_**  
Unlike downloading pre-packaged kernels from distributions like Ubuntu or Fedora, cloning directly from Linus Torvalds’ repository ensures you get:

- **Pure upstream code:** No distribution-specific patches or modifications
- **Latest features:** Access to the newest kernel development
- **Transparency:** Exactly what the kernel developers are working on
- **Control:** Complete understanding of every component in your system

```
# Navigate to the shared workspace inside the container  
cd /workspace/  
# Clone the latest Linux kernel (--depth 1 for minimal download)  
git clone --depth 1 https://github.com/torvalds/linux.git  
cd linux/
```

#### 2.2 Configure the Kernel.

**What is Kernel Configuration?**  
The Linux kernel is a massive project supporting countless hardware architectures, devices, and features. Configuration lets you select exactly which components to include. Think of it as building a custom car: you choose the engine, seats, and features you need, leaving out unnecessary weight.

```
# Launch the interactive configuration menu inside the container  
make menuconfig
```

**Note:** For this minimal system, you can simply exit and save without modifications. This creates a default x86_64 configuration.

#### 2.3 Add VGA support

**Why VGA Support Matters**  
Without proper display drivers, your kernel might boot successfully but show nothing on screen. This silent failure is frustrating for beginners. The commands below ensure the kernel can output to QEMU’s virtual VGA display.

- **Frame Buffer (FB):** Low-level graphics interface
- **Framebuffer Console**: Display console output on framebuffer
- **VGA Console**: Classic VGA text mode support
- **Virtual Terminal (VT)**: Multiple console sessions
- **Direct Rendering Manager (DRM)**: Modern graphics framework
- **Simple Framebuffer**: Minimal graphics for early boot

```
sed -i 's/# CONFIG_FB is not set/CONFIG_FB=y/' .config  
sed -i 's/# CONFIG_FRAMEBUFFER_CONSOLE is not set/CONFIG_FRAMEBUFFER_CONSOLE=y/' .config  
sed -i 's/# CONFIG_VGA_CONSOLE is not set/CONFIG_VGA_CONSOLE=y/' .config  
sed -i 's/# CONFIG_VT is not set/CONFIG_VT=y/' .config  
sed -i 's/# CONFIG_VT_CONSOLE is not set/CONFIG_VT_CONSOLE=y/' .config  
sed -i 's/# CONFIG_DRM is not set/CONFIG_DRM=y/' .config  
sed -i 's/# CONFIG_DRM_I915 is not set/CONFIG_DRM_I915=y/' .config  
sed -i 's/# CONFIG_FB_SIMPLE is not set/CONFIG_FB_SIMPLE=y/' .config
```

**Alternative Approach:**  
Instead of sed commands, you could use:

```
# Enable all necessary options in one go  
make menuconfig  
# Navigate to: Device Drivers → Graphics support  
# Enable: Framebuffer, VGA text console, Simple framebuffer
```

#### 2.4 Compile the Kernel.

Compiling the kernel transforms millions of lines of C code into a single executable binary. This process involves:

1. **Preprocessing**: Handling includes and macros
2. **Compilation**: Converting C to assembly
3. **Assembly**: Converting to machine code
4. **Linking**: Combining all objects into one binary

```
# Compile with parallel processing (adjust -j number based on your CPU cores)  
make -j$(nproc)  # Alternative: make -j16 for 16 threads
```

This produces an extremely small kernel configuration.

#### 2.5 Prepare the Boot Kernel.

The kernel is now compiled, producing a single file in folder “_arch/x86/boot/”_ with name “_bzImage”_.

Now we need to copy file to “_/workspace/boot-files”_ so that we can test it with qemu.

```
# Create boot directory for kernel image  
mkdir -p /workspace/boot-files  
# Copy the compressed kernel image  
cp arch/x86/boot/bzImage /workspace/boot-files/  
# Return to workspace root  
cd ..
```

#### 2.6 Test the Kernel (Host Machine).

**Why Test Without a Root Filesystem?**  
Testing the kernel alone confirms:

1. Successful compilation without linker errors
2. Basic hardware initialization works
3. Kernel can reach the point of mounting a root filesystem
4. No immediate crashes or hardware incompatibilities

```
# On your host machine (outside Docker)  
cd ~/Desktop/docker/boot-files  
qemu-system-x86_64 -kernel bzImage
```

**Expected Result:** The kernel should boot and panic with:

```
Kernel panic - not syncing: No working init found  
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

- **“No working init found”** = Kernel can’t find `/sbin/init`
- **“Unable to mount root fs”** = Kernel can’t find ANY filesystem

Both mean: **Your kernel works perfectly but has nothing to run!**

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*cWllAwyS-MmQ34bKyiaKdw.png)

**Next Steps:** The kernel is now ready for pairing with a minimal root filesystem to create a complete bootable system.
### 3. BusyBox: The Entire Userspace

When building a minimal Linux system, you quickly realize that a kernel alone is like an engine without a car. You need a **userspace** — the collection of utilities that make an operating system usable. This is where **BusyBox** shines as the ultimate minimalist solution.
#### What is BusyBox?

BusyBox is often called “The Swiss Army Knife of Embedded Linux.” It’s a single executable that replaces hundreds of standard Unix utilities (ls, cp, grep, init, sh, etc.) in one compact package. While a typical GNU/Linux distribution might have thousands of separate binaries totaling hundreds of megabytes, BusyBox provides the same core functionality in a single binary that can be as small as **1–2 MB**.

BusyBox is compiled as a **static binary**, which is critical. This allows the **initramfs** to work without shared libraries or dynamic linking.

Once built, BusyBox is installed into a directory that will become the _initramfs_ root.

```
# Clone BusyBox source  
cd /workspace  
git clone --depth 1 https://github.com/mirror/busybox  
cd busybox
```

Now that we have the BusyBox source code, we need to configure it for our minimal system and compile it into a static binary. This process is crucial for creating a self-contained userspace.

#### Essential Configuration //.

```
# Launch the BusyBox configuration interface  
make menuconfig
```

This opens an ncurses-based menu where we’ll make two critical changes:
#### 1. Enable Static Compilation //.

Navigate to **Settings** → **Build Options** and enable: `[*] Build static binary (no shared libs)`

**Why this matters:** A static binary contains all necessary libraries within the executable itself. This means:

- No external dependencies
- Works on any compatible Linux system
- Perfect for initramfs where no libraries exist
- Eliminates “shared library not found” errors at boot

### 2. Disable Problematic Features //.

Some `BusyBox` features may cause compilation errors with certain `toolchains`. To ensure a clean build:
Navigate to **Networking Utilities** and disable: `[ ] tc (traffic control utility)`

tc is a powerful network traffic shaping tool that depends on kernel networking extensions. For our minimal system, we don’t need it, and disabling it prevents potential compilation issues.

#### Compiling BusyBox.

```
# Build BusyBox with parallel compilation  
make -j$(nproc)  
# Or specify the number of cores:  
# make -j16
```
#### The Result .

After successful compilation, you’ll have:

- A single `busybox` executable (typically 1-2 MB)
- All core Unix utilities embedded within
- Zero external dependencies
- A binary ready for inclusion in our `initramfs`
### 4. Creating the `Initramfs`

With `BusyBox` compiled, we now need to create a home for our kernel to live in at boot time. This is where the **`initramfs`** (initial RAM `filesystem`) comes in it’s a temporary root `filesystem` loaded into memory during the early boot process.

#### What is an `Initramfs`?

Think of `initramfs` as a **minimal, self-contained environment** that the kernel mounts immediately after starting. It’s:

- **Temporary**: Lives in RAM, disappears after reboot
- **Essential**: Contains the bare minimum to continue booting
- **Flexible**: Can include drivers, tools, and setup scripts
- **Critical**: Without it, the kernel has nowhere to go

#### Creating the Structure.

```
# Create the initramfs directory structure  
mkdir -p /workspace/boot-files/initramfs
```

This single directory will become our entire root `filesystem`. Every file, directory, and binary needed for boot will live here.

#### Installing `BusyBox`.

```
# Install BusyBox into our initramfs directory  
make CONFIG_PREFIX=/workspace/boot-files/initramfs install
```

This command does something beautiful: it takes our single `busybox` binary and creates a **complete Unix environment** by generating hundreds of symbolic links.

#### The Magic of Symbolic Links.

After installation, look inside your `initramfs` directory: `ls -la /workspace/boot-files/initramfs/bin/`
You’ll see something like:

```
sh -> busybox  
ls -> busybox  
cp -> busybox  
mkdir -> busybox  
cat -> busybox  
echo -> busybox  
init -> busybox  
```


#### ... and hundreds more

Each symbolic link points to the same `busybox` binary. When you run ls, `BusyBox` checks `argv[0]` (how it was invoked) and executes the appropriate functionality.
##### What You Get //.

Your `initramfs` now contains:

- **`/bin/busybox`** — The main executable
- **/bin/** — Directory with all utility `symlinks`
- **`/usr/bin/`** — Additional `symlinks` for compatibility
- **`/sbin/`** — System administration tools
- **`/linuxrc`** — Traditional `initramfs` entry point

But we’re not done yet! This `filesystem` is still **empty** of the critical directories and configuration files needed for a functioning system. In the next step, we’ll:

1. Create essential directories (`/dev`, `/proc`, `/sys`, etc.)
2. Write an `init` script to set up the environment
3. Package everything into a format the kernel can load
### 5. Writing the `init` Script: Becoming PID 1

Every Linux system needs an `init` process—the very first program the kernel launches after boot. In our minimal system, we're creating this process ourselves, and it carries the sacred **PID 1** designation. This isn't just another program; it's the ancestor of all future processes and the guardian of the entire system.

#### Creating the Essentials.

```
# Navigate to our initramfs  
cd /workspace/boot-files/initramfs/  
  
# Create critical kernel interface directories  
mkdir -p proc sys dev
```

These directories aren’t just folders — they’re **communication channels** with the kernel:

- **`/proc`**: Live system information (processes, memory, hardware)
- **`/sys`**: Kernel objects and device information
- **`/dev`**: Device files representing hardware

#### The init Script.

```
# Navigate to our initramfs  
cd /workspace/boot-files/initramfs/  
  
# Edit file with vim  
vim init
```

```
#!/bin/busybox sh  
  
# Mount essential filesystems  
mount -t proc proc /proc  
mount -t sysfs sysfs /sys  
mount -t devtmpfs devtmpfs /dev  
  
# Wait for device initialization  
sleep 0.2  
  
# CRITICAL: Create VGA console if it doesn't exist  
# Only create once (devtmpfs may have already created it)  
if [ ! -c /dev/tty0 ]; then  
    echo "Creating VGA console /dev/tty0..."  
    mknod -m 620 /dev/tty0 c 4 0  
fi  
  
# Create other essential devices if they don't exist  
if [ ! -c /dev/tty ]; then  
    mknod -m 666 /dev/tty c 5 0  
fi  
  
if [ ! -c /dev/null ]; then  
    mknod -m 666 /dev/null c 1 3  
fi  
  
if [ ! -c /dev/zero ]; then  
    mknod -m 666 /dev/zero c 1 5  
fi  
  
# Set up /dev/pts for pseudo-terminals  
mkdir -p /dev/pts  
mount -t devpts devpts /dev/pts  
  
# CRITICAL: Force /dev/console to point to VGA (tty0)  
# This ensures the system uses VGA instead of serial  
ln -sf /dev/tty0 /dev/console  
  
# CRITICAL: Redirect all standard I/O to VGA console  
exec 0</dev/tty0  
exec 1>/dev/tty0  
exec 2>/dev/tty0  
  
# Set up terminal environment  
export TERM=linux  
export HOME=/root  
  
# Create home directory  
mkdir -p /root  
  
# Clear screen and display welcome message  
clear  
cat <<!  
  
========================================  
Welcome to Micro Linux!  
Boot took $(cut -d' ' -f1 /proc/uptime) seconds  
Running on VGA console (/dev/tty0)  
========================================  
  
!  
  
# CRITICAL: Start shell with proper terminal control for VGA  
# setsid cttyhack ensures proper job control on VGA console  
exec setsid cttyhack /bin/sh  
  
# If shell exits (shouldn't reach here with exec)  
echo ""  
echo "System shutting down..."  
poweroff -f
```

#### Finalizing the Setup.

```
# Make init executable  
chmod +x init  
  
# Remove the default linuxrc symlink  
rm linuxrc
```

The `linuxrc` symlink is BusyBox's default init, but we want full control over our PID 1 process.
#### Why This Architecture Matters.

Our `init` represents the **absolute minimum** viable userspace:

- No login prompts
- No services
- No background daemons
- No multi-user support
- Just you and a root shell

This is Linux distilled to its essence: kernel + shell = functional system.
### 6. Packaging the Initramfs

We’ve built all the pieces: a kernel, BusyBox utilities, and our init script. Now we need to package everything into a single file the kernel can load at boot time. This is where `cpio` (Copy In, Copy Out) comes in—an ancient but perfect archiving format for initramfs.

#### The CPIO Format: Why Old-School Works Best.

While you might be familiar with `tar` or `zip`, the kernel expects initramfs in **cpio format** because:

- **Kernel-native**: The Linux kernel has built-in cpio extraction
- **Simple structure**: No complex compression headers
- **Streamable**: Can be decompressed on-the-fly during boot
- **Proven reliability**: Used since the early days of Linux

#### Creating the Archive.

```
# Navigate to our initramfs directory  
cd /workspace/boot-files/initramfs/  
  
# Package everything into a cpio archive  
find . | cpio -o -H newc > ../initramfs.cpio
```

#### What’s Happening Internally?

The kernel will:

1. Load `initramfs.cpio` into memory during boot
2. Automatically extract it to a temporary in-memory filesystem
3. Execute `/init` as the first process
4. Mount this as the initial root filesystem

#### Testing Our Complete System.

Now for the moment of truth — booting our entire homemade Linux:

```
# On your host machine (outside Docker)  
cd ~/Desktop/docker/boot-files  
  
# Launch QEMU with our kernel and initramfs  
qemu-system-x86_64 \  
    -kernel bzImage \  
    -initrd initramfs.cpio \  
    -append "console=tty0"
```

![](https://miro.medium.com/v2/resize:fit:648/1*WwY_vCkeXXQdb9FSJpg5GA.png)

**Expected Result:** Within seconds, you should see:

```
Welcome to Micro Linux!  
Boot took 0.23 seconds  
Running on VGA console (/dev/tty0)  
#
```

#### The Complete Boot Chain.

Let’s visualize what you’ve created:

```
Kernel (bzImage) → Initramfs (cpio) → init Script → BusyBox Shell  
    ↓                    ↓                   ↓            ↓  
  Hardware          Root Filesystem      PID 1        User Interface  
  Initialization   (in memory)         Process         (Your Shell)
```
### 7. SYSTEMD-BOOT: A Modern UEFI Boot Loader

When moving beyond simple QEMU testing to real hardware or more sophisticated virtual setups, you’ll need a proper bootloader. **systemd-boot** (formerly **gummiboot**) is a minimalist UEFI boot manager that’s perfect for our lightweight system.

#### Why systemd-boot?

Unlike traditional BIOS bootloaders (GRUB, LILO), systemd-boot is:

- **UEFI-native**: Directly EFI executable, no intermediate stages
- **Minimal**: ~100KB vs GRUB’s ~10MB
- **Simple**: Plaintext configuration files
- **Secure**: Supports Secure Boot (with proper signing)
- **Fast**: Boots in milliseconds

#### Installation Prerequisites.

```
# Inside docker  
apt install systemd-boot-efi
```

#### Locating the EFI Binary.

The core EFI executable you need is:

- **Filename**: `systemd-bootx64.efi` (for x86_64 systems)
- **Standard Location**: `/usr/lib/systemd/boot/efi/systemd-bootx64.efi`

```
# Copy to our boot directory  
cp /usr/lib/systemd/boot/efi/systemd-bootx64.efi /workspace/boot-files/
```
### 8. Creating a UEFI Disk Image

Now we transform our scattered boot files into a **professional, self-contained bootable disk image** that can run on real hardware, virtual machines, or be written to USB drives.

#### The Anatomy of a Bootable UEFI Disk

Our goal is to create a FAT32-formatted disk image containing everything needed to boot:

```
uefi.img (FAT32 filesystem)  
├── EFI/BOOT/BOOTx64.EFI           # UEFI bootloader (systemd-boot)  
├── loader/                        # Bootloader configuration  
│   ├── loader.conf                # Global settings  
│   └── entries/mll-x86_64.conf    # Boot menu entry  
└── minimal/x86_64/                # Our system files  
    ├── kernel                     # Compressed Linux kernel  
    └── rootfs                     # Initramfs with BusyBox
```
##### 1. Create the Disk Image

```
# Create a 64MB empty disk image (adjust size as needed)  
dd if=/dev/zero of=uefi.img bs=1M count=64  
  
# Setup loop device (virtual block device)  
sudo losetup -fP uefi.img  
# Find the loop device (usually /dev/loop0)  
sudo losetup -a  
  
# Format as FAT32 (UEFI requires FAT filesystem)  
sudo mkfs.vfat -F 32 /dev/loop0
```
##### 2. Mount and Populate the Filesystem

```
# Create mount point and mount the image  
sudo mkdir -p /mnt/uefi  
sudo mount /dev/loop0 /mnt/uefi
```
##### 3. Install Systemd-Boot

```
# Create UEFI directory structure  
sudo mkdir -p /mnt/uefi/EFI/BOOT  
  
# Copy systemd-boot EFI binary (renamed for automatic detection)  
sudo cp systemd-bootx64.efi /mnt/uefi/EFI/BOOT/BOOTx64.EFI
```

**Note**: UEFI firmware automatically looks for `EFI/BOOT/BOOTx64.EFI` on FAT partitions.
##### 4. Copy Kernel and Root Filesystem

```
# Create organized directory for our system  
sudo mkdir -p /mnt/uefi/minimal/x86_64  
  
# Copy kernel and initramfs  
sudo cp bzImage /mnt/uefi/minimal/x86_64/kernel  
sudo cp initramfs.cpio /mnt/uefi/minimal/x86_64/rootfs
```
##### 5. Configure the Bootloader

```
# Create loader directory  
sudo mkdir -p /mnt/uefi/loader/entries  
  
# Global loader configuration  
sudo tee /mnt/uefi/loader/loader.conf > /dev/null << EOF  
default mll-x86_64  
timeout 3  
EOF  
  
# Boot entry configuration  
sudo tee /mnt/uefi/loader/entries/mll-x86_64.conf > /dev/null << EOF  
title   Minimal Linux (BusyBox)  
linux   /minimal/x86_64/kernel  
initrd  /minimal/x86_64/rootfs  
options console=tty0 vga=791   # VGA text mode 1024x768  
EOF
```
**Boot Options Explained:**

- `console=tty0`: Use VGA console (your monitor)
- `vga=791`: Set 1024×768 text mode (see Linux VGA modes)

##### 6. Finalize and Cleanup

```
# Ensure all writes are flushed to disk  
sync  
  
# Unmount the image  
sudo umount /mnt/uefi  
  
# Detach the loop device  
sudo losetup -d /dev/loop0  
  
# Optional: Remove mount point  
sudo rmdir /mnt/uefi
```
### 10. Testing with `QEMU` (`UEFI`)

The final image is tested using `QEMU` with `OVMF` firmware.

```
sudo qemu-system-x86_64 -bios OVMF.fd -m 1024 -drive format=raw,file=uefi.img
```

If the `systemd`-boot menu appears and the `BusyBox` shell starts, the system is complete.

Press enter or click to view image in full size
## Real-World Applications

The same image can then be written directly to a USB drive using `dd` and booted on real hardware.

```
sudo dd if=uefi.img of=/dev/sdX bs=4M status=progress conv=fsyn
```
## Conclusion

This tiny Linux system proves how little is actually required to boot Linux:

- One kernel  
- One statically linked binary  
- One small script

No systemd.  
No package manager.  
No disk installation required.

Just Linux — stripped down to its core.

## Continue
### 1) Kernel (the brain)

Compiled file: `bzImage`. It:

- talks to CPU, RAM, hardware
- mounts first filesystem
- runs **first process (PID 1)**

But kernel **cannot give you shell or commands**.
So when you boot kernel alone → panic: "No working init found"
Because kernel says:   "I am alive… but what program should I run?"
### 2) Initramfs (temporary mini-OS in RAM)

The article’s biggest idea.

It is NOT disk  
It is NOT `/` partition  
It is a **RAM filesystem loaded at boot**

Kernel extracts it in memory and mounts as root `/`. So at boot:

```
There is NO hard disk yet  
There is NO ext4 yet  
There is ONLY RAM
```

### 3) BusyBox (the entire userspace)

Single binary providing: `ls, sh, mount, cp, mkdir, init, echo, cat, etc`
Instead of 3000 programs → 1 file (~1-2MB)
And compiled STATIC → means:    it does not need glibc or libraries
So your Linux can run with **zero packages installed**.
### 4) `/init` (the real first program)

This line in article is the most important Linux concept ever: `exec setsid cttyhack /bin/sh`. This literally means:

```
> Start the first shell in existence  
> This shell becomes the OS
```

No login  
No systemd  
No services
Just: `You + Kernel`

### What you actually built (philosophically)

Normal distro: `Linux = Kernel + systemd + glibc + packages + services + desktop`
Article Linux: `Linux = Kernel + shell`
That’s it.

### 1) Power on

Your motherboard firmware starts:

- old PCs → **BIOS**
- modern PCs → **UEFI**

Firmware already knows how to access:
- SATA
- NVMe
- USB
- partitions

So at this moment the disk _is accessible_ — just not using Linux.
### 2) Firmware loads GRUB

Firmware reads a file from disk:
BIOS: `MBR → stage1 → stage2 → GRUB`
UEFI: `EFI partition → /EFI/BOOT/grubx64.efi`
GRUB is now running **using firmware disk read functions**.
Think of it like:

```
GRUB: "Hey firmware, give me sector 2048"  
Firmware: "Here, bytes"
```

No kernel involved.
### 3) GRUB loads the kernel

GRUB parses filesystem itself:
- ext2/ext4
- fat32
- btrfs (limited)

Then loads:

```
/boot/vmlinuz  
/boot/initramfs.img
```

into RAM. So the disk is accessed like: `Disk → Firmware → GRUB → RAM → Kernel`
### 4) Kernel starts — NOW drivers matter

Only after kernel starts:

```
Kernel initializes SATA/NVMe drivers  
Kernel detects disks again  
Kernel mounts real root filesystem
```

This is the **second disk detection** — real Linux detection.
### Why your tiny Linux still boots

In that article:

GRUB loads kernel + initramfs into memory  
After that → disk can disappear → system still runs

Because root filesystem is RAM.

### Key insight (important)

There are **two different disk worlds**:

|Stage|Who reads disk|How|
|---|---|---|
|Bootloader|Firmware|BIOS/UEFI services|
|Linux|Kernel|Hardware drivers|

So when people say:   “GRUB can't detect disk”
They actually mean:

```
> Kernel can't detect disk  
> (missing driver)
```

But GRUB already read it earlier using firmware.

### Think of initramfs as a “temporary emergency OS”

Your computer has a big OS on disk: `/dev/sda2  → Ubuntu / Arch / whatever`
But at boot time…  **Kernel cannot read it yet**
Why?
Because to read the disk it needs:
- filesystem driver (ext4, btrfs…)
- disk driver (NVMe/SATA)
- sometimes LVM
- sometimes encryption unlock

But those drivers live **inside the OS you are trying to load** 🤯
So we have a chicken-egg problem:

```
> To load OS → need drivers  
> Drivers → inside OS
```

### The solution Linux invented: initramfs

We first load a **tiny fake OS into RAM**
This RAM OS contains only enough tools to make the real OS accessible.
That RAM OS = **initramfs**
### Boot flow (real world)

```
BIOS/UEFI  
   ↓  
GRUB loads kernel + initramfs into RAM  
   ↓  
Kernel starts  
   ↓  
Kernel mounts initramfs as /  
   ↓  
Runs /init   ← FIRST PROGRAM EVER  
   ↓  
/init prepares real root filesystem  
   ↓  
Switch root → real OS  
   ↓  
systemd starts
```

So Linux boots **TWICE**
1. Temporary RAM Linux
2. Real Linux

### What is inside initramfs?

Very small tools (usually BusyBox):

```
/init  
/bin/sh  
/bin/mount  
/bin/ls  
drivers
```

Its only job:   make the real disk usable

### Example: encrypted laptop

Without initramfs the kernel sees: `nvme0n1p2 → encrypted garbage`
Kernel:   "I see a disk but I can't read it"
Initramfs runs:

```
ask password  
unlock luks  
activate LVM  
mount ext4  
switch_root
```

Now kernel can continue booting.
### The MOST IMPORTANT LINE (magic moment)

Inside initramfs: `switch_root /newroot /sbin/init`
This literally means:

```
> Throw away the RAM OS  
> Continue life inside real OS
```

After this — initramfs disappears forever.

### What the article you read did

It never switched to a real disk.
So the RAM OS became the real OS.
You booted a computer that lives entirely in memory.
That’s why removing the disk wouldn’t crash it 😄
#### Mental model (remember this)

|Thing|Purpose|
|---|---|
|Kernel|Can run programs but can't access disk yet|
|Initramfs|Makes disk accessible|
|Real rootfs|Actual operating system|


#### 1) Kernel vs Userspace (the biggest separation)

A running Linux system is split into two worlds:

```
KERNEL SPACE  (God mode — hardware control)  
USER SPACE    (Normal programs — what you interact with)
```

Kernel (drivers, memory, cpu scheduling)  
Programs (ls, bash, python, nginx) live outside it.
The kernel **never prints text, shows prompt, or runs commands**.
So when Linux boots, the kernel’s first job is: “Start a program so humans can control me”
That first program = `/init`
Everything after that = **userspace**

#### 2) What is a Shell?

A shell is simply:  A program that lets humans talk to the kernel
You type: `ls`
Shell translates that into:

```
start program /bin/ls  
wait for result  
print output
```

So yes:   **A shell is just a C program**
Examples:
- **Bash**
- **Zsh**
- **Dash**
- BusyBox `sh`

Without shell → Linux runs but you cannot control it.
#### 3) What is BusyBox?

**BusyBox is a special program.
Normally Linux has thousands of programs:

```
/bin/ls  
/bin/cat  
/bin/mount  
/bin/sh  
/bin/echo  
...
```

BusyBox compresses ALL of them into ONE executable: `/bin/busybox`
And creates fake names:

```
ls → busybox  
cat → busybox  
mount → busybox  
sh → busybox
```

Same binary — different behavior depending on filename.
So BusyBox is:    100+ command line tools + a shell inside one file
Perfect for initramfs because tiny.
#### 4) What is Bash then?

**Bash is just a _feature-rich shell program_.
Think:

| Program      | Type                   |
| ------------ | ---------------------- |
| bash         | luxury shell           |
| sh (busybox) | minimal survival shell |

BusyBox shell supports basic commands only:

```
if  
for  
cd  
ls  
mount
```

Bash supports:

```
arrays  
history  
autocomplete  
aliases  
job control  
scripting features
```

Your tiny Linux used BusyBox shell, not Bash.
#### 5) Putting it all together

When your tiny Linux boots:

```
Kernel starts  
↓  
Runs /init  
↓  
/init launches BusyBox sh  
↓  
You see prompt
```

So what you see on screen is:
NOT Linux  
NOT Kernel
It is:  a small C program talking to the kernel for you

# 3️⃣2️⃣ SHELL & Cttyhack

Yes, a shell can run **without a TTY**, but you cannot interact with it directly without some kind of terminal device.
Now let’s break this properly.
### What is a TTY?

TTY = terminal device.
Examples:

- Physical console: `/dev/tty1`
- Virtual terminals: Ctrl+Alt+F1–F6
- Pseudo terminals (SSH, terminal emulator)

The kernel provides these devices.
### Can a shell run without a TTY?

Yes.
A shell is just a normal userspace program.  
For example: `bash script.sh`
Here:

- No interactive terminal needed
- Shell reads commands from a file
- It exits when done

This is called **non-interactive mode**.
### Interactive vs Non-Interactive Shell

|Mode|Needs TTY?|Example|
|---|---|---|
|Interactive|Yes|typing commands|
|Non-interactive|No|running a script|
|Piped|No|`echo hi|

Example without TTY: `echo "ls" | bash`
Shell runs. No terminal attached.
### What happens in your tiny Linux?

In your build (kernel + BusyBox):
If you only run: `/bin/sh`
without attaching it to a console device like `/dev/tty1`,
then:

- It runs
- But you can’t type into it
- It has no input/output device

That’s why tutorials use something like: `exec setsid cttyhack /bin/sh`
BusyBox provides `cttyhack` to attach the shell to a controlling TTY.
(BusyBox is from BusyBox)
### Can Linux work without TTY at all?

Yes. Many systems run without any terminal:

- Containers
- Background servers
- Embedded devices
- PID 1 running silently

They just: `shell/script → kernel → hardware`
No human interaction needed.
### Key insight

TTY is not required for:

- The kernel
- The shell
- The OS to function

TTY is only required for **interactive human input/output**.
### Final Answer

A shell can run without a TTY, but you cannot interact with it unless it is connected to a terminal device.
Why can’t I run commands in a shell if there is no TTY?
The answer is about **input and output streams**.

### 1️⃣ A shell is just a program

The shell (bash, ash, etc.) is a normal userspace program. Like every Unix program, it has:

```
stdin   (file descriptor 0)  
stdout  (file descriptor 1)  
stderr  (file descriptor 2)
```

It reads commands from **stdin**.

### 2️⃣ What happens in interactive mode

When you log in normally:

```
keyboard → TTY device → stdin of shell  
shell → stdout → TTY → screen
```

The shell waits for input from stdin. If stdin is connected to a TTY, you can type.

### 3️⃣ What happens without a TTY

If the shell starts and:

- stdin is not connected to anything
- or connected to a closed pipe
- or redirected from /dev/null

Then:

shell tries to read from stdin  
→ nothing available  
→ read() returns EOF  
→ shell exits

It doesn’t know where to get commands from. That’s why you “can’t run commands”. Not because the shell refuses.
But because: There is no input source.
### 4️⃣ Important concept: Controlling terminal

Interactive shells need:

- A controlling terminal
- Job control
- Signal handling (Ctrl+C, Ctrl+Z)

Without a TTY:

- No job control
- No signal from keyboard
- No prompt behavior

So the shell switches to non-interactive mode or exits.

### 5️⃣ Why tutorials use `cttyhack`

In your tiny Linux (using BusyBox): They do: `exec setsid cttyhack /bin/sh`
Why? Because:

- PID 1 has no controlling TTY
- The shell needs one to behave interactively
- `cttyhack` attaches `/dev/console` as the controlling terminal

Without that, the shell runs but you cannot type.

### 6️⃣ Deep kernel reason

The kernel does not automatically connect programs to a keyboard. Programs read from file descriptors.
TTY devices are special character devices that:

- Translate keyboard input
- Handle line editing
- Send signals (SIGINT, SIGTSTP)

Without that device:  
The shell is blind and deaf.
### Final simple explanation

You cannot run commands interactively without a TTY because:

```
> The shell needs stdin connected to something that provides input.  
> A TTY provides that input.  
> Without it, the shell has nothing to read.
```

## Imagine the shell is like this:

The shell does only one main thing:

```
while (true) {  
    read a line  
    execute it  
}
```

The important part is: **read a line**
Read from where? It reads from something called **stdin** (standard input).
### Case 1️⃣ Normal login (with TTY)

When you open terminal: `keyboard → TTY → shell stdin`
So when shell says: `read()`
The kernel gives it what you typed.
That’s why it works.
### Case 2️⃣ No TTY

Now imagine:
shell starts  
stdin = nothing
Shell says: `read()`
Kernel replies: `EOF (no input available)`
Shell thinks: "No input? Then I’m done."
So it exits immediately.
That’s why you “can’t run commands”.
Not because commands are disabled.  
But because **the shell has nothing to read from.**

### Why `cttyhack` was needed in your tiny Linux

In your minimal system using BusyBox:
When kernel starts PID 1: There is **no terminal attached yet**.
So shell has: `stdin = empty`
So it exits.
`cttyhack` connects: `/dev/console → stdin/stdout`
Now shell can read.
### One sentence answer

You can’t run commands without a TTY because:

```
> The shell needs input to know what command to run.  
> Without a TTY (or some other input source), it receives nothing and exits.
```

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

# 3️⃣4️⃣ lib
### What “dynamically linked” means

A Linux program needs many helper functions to run:

- printing text (`printf`)
- memory allocation (`malloc`)
- file operations
- threading, math, etc.

Those helpers live inside **libraries** (like `libc.so`). There are **two ways** a program can use libraries:

### 1) Static linking (everything packed inside the program)

All library code is copied into the executable when it’s built.
Think of it like:   A fully offline app — carries everything inside itself.
**Result:**

- Big file size 📦
- Runs even if system libraries missing
- Faster startup
- No updates automatically

Example: `hello_static = 2–5 MB`

### 2) Dynamic linking (what your `make` uses)

The executable **does NOT contain the library code**.
Instead it says:   "When I run, load required code from the system libraries."
So Linux loads libraries from:

```
/lib  
/usr/lib  
/lib64
```

Your output shows this: `interpreter /lib64/ld-linux-x86-64.so.2`
That is the **dynamic loader** — the program that connects your binary to system libraries at runtime.

### Real-life analogy

|Type|Analogy|
|---|---|
|Static|Movie downloaded to phone|
|Dynamic|Netflix streaming from server|

### Why Linux prefers dynamic linking

Because thousands of programs share the same libraries.
If `glibc` updates → **all programs automatically get fixes**  (no reinstall needed)
Saves huge disk space.
### How to check libraries used by `make`

Run: `ldd /bin/make`
You’ll see something like:

```
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6  
libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2
```

That means `make` borrows code from those files when running.
### Quick summary

**Dynamically linked = program loads libraries at runtime instead of containing them.**

✔ Smaller binaries  
✔ Shared updates  
✔ Less RAM overall  
❌ Needs system libraries present

### What is a `.so` file?

`.so` = **Shared Object**
It is a **Linux shared library file** — basically the library that a _dynamically linked_ program loads while running.
So when a program starts, Linux says:  “I need extra functions → load them from `.so` files”

### Think of it like this

|File type|Meaning|
|---|---|
|`.exe` (Windows)|program|
|`.dll` (Windows)|shared library|
|`.so` (Linux)|shared library|

`.so` = Linux version of `.dll`
### Example on your system

Run: `ldd /bin/make`
You’ll see something like:

```
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6  
libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2
```

That means:

**`make` does NOT contain printf(), malloc(), etc**  
→ It borrows them from `libc.so.6`

### What is inside a `.so`

Compiled functions:

```
printf()  
malloc()  
fopen()  
pthread_create()  
sin()
```

Thousands of programs share the SAME file in memory.
So RAM usage becomes:

|Without .so|With .so|
|---|---|
|100 programs load libc 100 times|libc loaded once and shared|

Huge memory saving 🚀

## Why it's called _shared_

Because:
- shared between programs
- shared in RAM
- shared on disk

### Where they live

```
/lib  
/lib64  
/usr/lib  
/usr/lib64
```

### Static vs Shared

|Static `.a`|Shared `.so`|
|---|---|
|copied into program|loaded at runtime|
|big executable|small executable|
|portable|depends on system|
|faster startup|flexible updates|

One-line definition  A `.so` file is a compiled library that Linux programs load dynamically at runtime to use common functions.
Your `cat` says:

> **statically linked**  
> but still shows an **interpreter** → `/lib/klibc-xxxx.so`

At first that looks contradictory. Let’s break it properly.

### Normally

You usually see:

|Type|Interpreter|
|---|---|
|Dynamic binary|`/lib64/ld-linux-x86-64.so.2`|
|Static binary|**no interpreter**|

But your binary is special — it comes from **klibc**.

### What is klibc?

**klibc = tiny C library used in early boot environment** Used by:

- initramfs
- recovery shell
- early userspace before real Linux starts

It exists so Linux can run programs **before glibc is available**.

### So why “statically linked” but still interpreter?

Because this is not normal dynamic linking.
This is:  **self-contained mini runtime loader**
The interpreter `/lib/klibc-<random>.so` is bundled as part of the boot environment.
So:

|glibc program|klibc program|
|---|---|
|loads big system libraries|loads tiny boot library|
|depends on full OS|works before OS exists|
|real dynamic linking|boot-time pseudo-static linking|

In simple terms:

👉 The program already contains almost everything  
👉 But still needs a tiny loader to start execution

### Why klibc does this

During boot:

1. Kernel starts
2. No `/usr/lib`, no full filesystem yet
3. Need `cat`, `sh`, `mount`, `mkdir`

So Linux ships **micro-binaries**. They are:

- almost static
- tiny (~20–60 KB)
- portable inside initramfs

### Compare sizes

Try:

```
ls -lh /bin/cat  
ls -lh /lib/klibc/bin/cat
```

You’ll see:
- normal `cat` → ~50-100 KB (dynamic)
- klibc `cat` → ~10-30 KB

### What that weird interpreter really is

`klibc-xxxxx.so` is basically:   a tiny startup runtime that replaces the normal dynamic linker
So execution flow:

```
kernel  
  ↓  
klibc loader (.so)  
  ↓  
cat program
```

Instead of: `kernel → ld-linux → glibc → program`
### One-line explanation

Your `cat` is **boot-time minimal executable** — mostly static, but uses a tiny klibc runtime loader instead of the full system dynamic linker.

# 3️⃣5️⃣ BusyBox GNU USERSPACE
### What _userspace_ actually is

**Userspace = everything that runs _outside the kernel_** and talks to the kernel using **system calls**.
So yes — commands, binaries and libraries are part of it… but `userspace` is _much broader_.

### It includes:

1. **Shells (command interpreters)**
    - `bash`, `zsh`, `sh`
    - They let humans interact with the system

2. **User programs / applications**
    - Commands: `ls`, `cp`, `grep`
    - Editors: `vim`, `nano`
    - Browsers: Firefox, Chrome
    - Servers: nginx, apache, mysql
    - GUI apps: file manager, terminal, etc.

3. **Binaries (executables)**
    - Files in `/bin`, `/usr/bin`, `/usr/sbin`
    - Any compiled program you run

4. **Libraries**
    - Shared libs like `libc.so`
    - Provide functions so programs don’t talk to kernel directly

5. **System services / daemons**
    - `systemd`
    - `sshd`
    - `cron`
    - Network managers

6. **Graphical environment**
    - X11 / Wayland
    - Desktop environments (GNOME, KDE)
### Key idea

Userspace **cannot directly access hardware**.
Instead it requests the kernel: `userspace program → system call → kernel → hardware`
Example:

```
cat file.txt  
   ↓  
read() system call  
   ↓  
kernel reads disk  
   ↓  
returns data to program
```

Simple definition: Userspace is all software running on top of the kernel that interacts with hardware only through system calls.

### Quick contrast

|Kernel space|User space|
|---|---|
|Drivers|Applications|
|Memory management|Commands|
|Process scheduling|Shell|
|Hardware control|Libraries|

So your statement:   "userspace is nothing but commands, binaries, libraries"

✔ Partially correct  
❌ Missing: services, GUI, daemons, interpreters, servers, applications — basically **everything you interact with in Linux**

### What BusyBox actually is

BusyBox is a **single executable program** that bundles _hundreds of Linux utilities_ into one binary.

Think of it like:  a miniature Linux userspace packed into one file

It is heavily used in:
- embedded systems (routers, TVs, IoT)
- initramfs
- recovery environments
- containers

### Does it include a shell?

Yes. BusyBox provides a shell called:   **`ash` (Almquist shell)**
When you run: `/bin/sh`
On many minimal systems → you are actually running **BusyBox ash**
### What BusyBox provides

Inside one binary:

|Category|Examples|
|---|---|
|Shell|`sh` (ash)|
|Commands|`ls`, `cp`, `mv`, `cat`, `grep`|
|System tools|`mount`, `ifconfig`, `ps`|
|Network|`ping`, `wget`, `telnet`|
|Init|init system (sometimes)|

All symlink to the same binary:

```
ls -l /bin/ls  
→ /bin/ls -> busybox
```

BusyBox checks `argv[0]` to decide what command to behave as.
### Important concept

**BusyBox = multi-call binary** `1 program → acts like 300+ commands`
### So final answer

```
> BusyBox is not a shell.  
> It is a collection of utilities that _includes_ a shell (ash).
> YES — in that custom Linux you build, you really are using BusyBox `ash` as your actual shell.
```

And not just briefly — it is your _entire user environment_.
### What that tutorial actually creates

The article builds a Linux system made of only:

- Linux kernel
- initramfs
- BusyBox
- your handwritten `init`

No GNU tools, no bash, no systemd, no coreutils.
The author even says the goal is a system that _boots straight into a BusyBox shell_.
### The boot chain in your mini-Linux

From the article: `Kernel → Initramfs → init script → BusyBox shell`
So after boot:  **PID 1 (init)** runs your script  
then your script runs: `exec setsid cttyhack /bin/sh`
And `/bin/sh` is a symlink: `sh -> busybox`
Meaning: `/bin/sh = BusyBox ash`
Because BusyBox provides the shell implementation.
### So what shell are you actually typing into?

|System|Shell you interact with|
|---|---|
|Ubuntu/Fedora|bash/zsh|
|Alpine Linux|ash (BusyBox)|
|Your tiny Linux|**ash (BusyBox)**|

In your build:   the operating system literally **is kernel + BusyBox + your script**
There is no other shell available.
### Why this matters (the big realization)

You’ve basically built:  the _minimum possible Unix_: `kernel + shell = operating system`
Everything else in normal distros is just extra layers.
 `ash` _is itself a userspace program_ that lets _you_ interact with the kernel.
### Correct chain

```
You (human)  
   ↓  
Shell (ash)  
   ↓  
Programs / system calls  
   ↓  
Kernel  
   ↓  
Hardware
```

So:   ash = command interpreter (UI between human and OS)
### What the shell actually does

The `ash` shell from BusyBox:
1. Reads your command
2. Parses it
3. Decides what to run
4. Starts a program OR makes system calls

Example: `cat file.txt`
### What really happens

```
You type command  
→ ash parses "cat"  
→ ash launches /bin/cat  
→ cat calls read() syscall  
→ kernel reads disk  
→ hardware returns data  
→ kernel → cat → ash → you
```

### Key understanding

```
The shell does NOT talk to hardware  
The shell does NOT control the kernel
```

It only:   **requests the kernel to run programs**
### So refine your sentence

Your sentence:  ash is a way to interact with userspace in turn with kernel in turn with hardware
Better: ash is a userspace program that lets the user request services from the kernel, which then controls hardware.

### One-line mental model

**Kernel = power**  
**Userspace programs = workers**  
**Shell = receptionist between you and workers**

You never talk to workers directly — only through the receptionist.> **In userspace, the shell is just one program, and we use it to run other userspace utilities.**
### Structure of userspace

```
Userspace  
 ├─ Shell (ash/bash)  
 ├─ Utilities (ls, cat, mount, cp…)  
 ├─ Libraries (libc)  
 └─ Applications (editors, servers, etc.)
```

The shell itself does almost nothing alone — it mainly **starts other programs**.
### Example in your tiny Linux

Your shell is `ash` from BusyBox
You type:  ls /dev
What happens:

```
ash → runs "ls" utility  
ls  → asks kernel for directory entries  
kernel → reads filesystem → disk
```

So:

> shell = launcher  
> utilities = workers

### Important insight

The shell is **not special to the kernel**. To Linux, `ash` is just another normal program like `cat` or `grep`.
The kernel doesn’t know:
- commands
- pipes
- redirection
- scripting


Those are all features implemented by the shell.
### Final refined statement

> Userspace contains many programs.  
> The shell is a userspace program that we use to run other userspace utilities, which then request services from the kernel.

### In normal distros (Ubuntu, Fedora, Arch…)

We do **NOT** rely on BusyBox for userspace.
Instead we use the **GNU userspace** — mainly the project:   GNU Project
So a typical system is actually: `Linux kernel  +  GNU userspace`
That’s why people often say **GNU/Linux**.

### Main components of normal userspace

#### 1️⃣ Shell

Most common:

|Shell|Where|
|---|---|
|bash|default in many distros|
|zsh|popular modern setups|
|fish|user-friendly shell|

Example:  GNU Bash is default in many installations
#### 2️⃣ Core utilities (NOT BusyBox ones)

Instead of BusyBox tools, you use:  GNU Core Utilities
These provide: `ls, cp, mv, rm, cat, chmod, chown, mkdir...`
Full-featured, bigger, more standards-compliant than BusyBox versions.

#### 3️⃣ System libraries

Programs don’t talk to kernel directly — they use:  glibc
This is the most important userspace component after the kernel.

#### 4️⃣ Init system

Modern distros boot with:   systemd
(not BusyBox init)
### Comparison

|Feature|Tiny Linux|Ubuntu/Fedora/Arch|
|---|---|---|
|Shell|ash|bash/zsh|
|Utilities|BusyBox|GNU coreutils|
|libc|musl/uclibc|glibc|
|Init|busybox init|systemd|
|Size|few MB|several GB|
|Purpose|embedded/minimal|general-purpose OS|
### Big picture

There are basically **two ecosystems of userspace**:

```
Embedded Linux → BusyBox world  
Desktop/Server Linux → GNU world
```

### Final answer

In Ubuntu and most distributions, userspace is mainly GNU tools (bash + coreutils + glibc + systemd), not BusyBox.

# 3️⃣6️⃣ Permanent DNS set

How to **`nameserver 10.60.10.3` to remain permanently in `/etc/resolv.conf`** on native Ubuntu.
Your `/etc/resolv.conf` is managed by `systemd-resolved`, so **don't edit that file directly**. Configure the DNS server underneath it.
### Permanent solution

Run: `sudo resolvectl dns wlp3s0 10.60.10.3`
Then verify: `resolvectl status`
You should see:

```
Link 3 (wlp3s0)
    Current DNS Server: 10.60.10.3
    DNS Servers: 10.60.10.3
```

However, `resolvectl dns` by itself can be lost when the network connection changes/reconnects.

### Make it permanent with NetworkManager

First get your Wi-Fi connection name: `nmcli connection show`
You'll see something like:

```
NAME              UUID                                  TYPE      DEVICE
Office WiFi       xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  wifi      wlp3s0
```

Then run: `sudo nmcli connection modify "Office WiFi" ipv4.dns "10.60.10.3"`
Also tell NetworkManager to use the DNS you specified: `sudo nmcli connection modify "Office WiFi" ipv4.ignore-auto-dns yes`
Reconnect:

```
sudo nmcli connection down "Office WiFi"
sudo nmcli connection up "Office WiFi"
```

Now check: `resolvectl status`
and: `cat /etc/resolv.conf`
You will still see: `nameserver 127.0.0.53`
**That is normal.** `127.0.0.53` is Ubuntu's local `systemd-resolved` DNS stub. The actual upstream DNS will be: `10.60.10.3`
So you don't actually need the literal line `nameserver 10.60.10.3` in `/etc/resolv.conf`; `systemd-resolved` forwards the requests to `10.60.10.3`.

# 3️⃣7️⃣ Manually adding a screen resolution

**manually adding a screen resolution that Ubuntu does not currently detect**.

### 1. Check the current display modes

```
xrandr
```

`xrandr` = **X Resize and Rotate**.
It asks the graphics/display system: "What displays do I have, what resolution are they using, and what resolutions are available?"
You saw: `eDP-1 connected primary 1366x768`. So:

- `eDP-1` = your laptop's internal screen
- `connected` = Ubuntu detects it
- `primary` = main display
- `1366x768` = current resolution

And importantly, `1920x1080` wasn't in the list.

### 2. Calculate a 1920×1080 display mode

```
cvt 1920 1080 60
```

`cvt` means **Coordinated Video Timings**.
You're telling it: "Calculate the technical timing parameters required to display 1920×1080 at 60 Hz."
It produced: `Modeline "1920x1080_60.00" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync`
The important part for you is: `1920 1080`
The rest are **timing parameters** used by the display system.
Think of the Modeline as: "Here are the exact instructions for how to generate a 1920×1080 @ 60 Hz video signal."

### 3. Create that mode inside xrandr

```
xrandr --newmode "1920x1080_60.00" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
```

This tells `xrandr`: "Create a new display mode called `1920x1080_60.00` using these timing parameters."
After this, the mode exists in xrandr, but it isn't attached to your screen yet.
Think:

```
1920x1080 mode
       ↓
created in xrandr
       ↓
but not assigned to eDP-1 yet
```

### 4. Attach the mode to your laptop display

```
xrandr --addmode eDP-1 "1920x1080_60.00"
```

This tells xrandr: "Make this newly-created mode available on my `eDP-1` display."
Now the relationship is:

```
eDP-1
 ├── 1366x768
 ├── 1280x720
 ├── 1024x768
 └── 1920x1080_60.00  ← manually added
```

You can check with: `xrandr`
### 5. Actually switch to it

```
xrandr --output eDP-1 --mode "1920x1080_60.00"
```

This tells Ubuntu: "Use the 1920×1080 mode on the eDP-1 display now."
So the complete process is:

```
xrandr
  ↓
Check available resolutions

cvt 1920 1080 60
  ↓
Calculate timing information

xrandr --newmode ...
  ↓
Create 1920×1080 mode

xrandr --addmode eDP-1 ...
  ↓
Attach that mode to your screen

xrandr --output eDP-1 --mode ...
  ↓
Actually switch the screen to it
```

# 3️⃣8️⃣ This app is not sandboxed

In Ubuntu, when you hear **“this app is not sandboxed”**, it means the application is **not isolated from the rest of your system**.
Think of it like this:

### 1. Normal app — not sandboxed

Suppose you install a program normally:

```
Ubuntu
│
├── Your files
├── System files
├── Network
├── Hardware
└── Application
       └── can potentially access many of these
```

The application runs as your user account, so it generally has the same permissions that **you** have.
For example, if you are logged in as `koushik`, an application might be able to access:

```
/home/koushik/Documents
/home/koushik/Downloads
/home/koushik/.config
```

It doesn't automatically have permission to modify protected system files such as:

```
/etc
/usr
/boot
```

but it can access a lot of your user-level files.
### 2. Sandboxed app

A sandbox puts the application inside a restricted environment:

```
Ubuntu
│
├── Your files
│
├── System
│
└── Sandbox
     └── Application
          ├── Limited files
          ├── Limited hardware
          └── Limited system access
```

The application may be prevented from accessing things it doesn't need. For example:

```
Application
     │
     ├── ✓ its own files
     ├── ✓ allowed folders
     ├── ✓ allowed network access
     │
     ├── ✗ your private documents
     ├── ✗ arbitrary system files
     └── ✗ certain hardware
```

This is mainly a **security mechanism**.
### Where you'll see this in Ubuntu

You may encounter this when installing applications as **Snap** or **Flatpak**.
For example: `snap install firefox`
Snap applications normally run with confinement/sandboxing.
You might see terminology such as: `confinement: strict`
which means the Snap is restricted by its security policy.
Traditional packages installed with: `sudo apt install <package>` are generally **not sandboxed in the same application-isolation sense**.
### Important distinction

**Not sandboxed ≠ dangerous.**

It simply means: "This application isn't being isolated from the operating system by a sandbox."
For example, many ordinary Linux programs are not sandboxed and are perfectly normal and safe.
The bigger issue is **what permissions the application has**.
A program running as your normal user has your user's permissions.
A program running as `root` has extremely powerful permissions:

```
normal user
    ↓
limited access

root
    ↓
almost complete system access
```

So when Ubuntu says an application **isn't sandboxed**, think:

```
> Sandbox = a restricted room.  
> Not sandboxed = the application is running in the normal Ubuntu environment, subject to its user/system permissions.
```

Installed **Google Chrome from a `.deb` file**, it is a traditional system package rather than a Snap/Flatpak application.

That does **not** mean Chrome has no security isolation at all—Chrome itself uses its own sandbox internally. But if you mean **Ubuntu-level application sandboxing**, you can put the `.deb`-installed Chrome inside another sandbox.
### Your options

**1. Keep your current `.deb` Chrome**

- Chrome already has its **own browser sandbox**.
- This is the simplest option.
- You don't need to sandbox it again for normal use.

**2. Install Chrome as a Flatpak**

- Flatpak provides stronger OS-level application isolation.
- You control which files, devices, and services the application can access.
- This is usually the easiest way to get a Linux desktop-app sandbox.

**3. Use Firejail**

- You can keep your existing `.deb` Chrome and launch it through **Firejail**.
- Example: `sudo apt install firejail`

Then: `firejail google-chrome`
Firejail creates a restricted environment around the application.

### One important thing

Don't simply run: `sudo firejail google-chrome`
unless you specifically understand why you're doing it. Running Chrome as root is unnecessary and undesirable.

# 3️⃣9️⃣ openssl

## First: the big picture

Suppose you own: `example.com`
You want NGINX to serve: `https://example.com`
You need an **SSL/TLS certificate**.
The basic flow is:

```text
                 You
                  │
                  │ Generate private key
                  ▼
            Private Key
                  │
                  │ Generate CSR
                  ▼
                CSR
                  │
                  │ Send to Certificate Authority
                  ▼
              Sectigo
                  │
                  │ Issues certificate
                  ▼
            SSL Certificate
                  │
                  │ Install on NGINX
                  ▼
          https://example.com
```

Now let's decode every term.

### 1. ECC

**ECC = Elliptic Curve Cryptography**
It's a type of cryptography used for creating **public/private key pairs**.
There are two common types you'll encounter:

```text
RSA
ECC
```

For example:

```text
RSA private key
RSA public key
```

or:

```text
ECC private key
ECC public key
```

ECC provides strong security with relatively small keys.
For example:

```text
RSA: 2048-bit
ECC: 256-bit
```

You should **not** think: ECC = certificate
Instead: ECC = cryptographic technology used by the key/certificate.

### 2. CSR

**CSR = Certificate Signing Request**
This is a request you create when you want a Certificate Authority to issue you a certificate.
For example:

```text
You
 │
 │ "I need a certificate for example.com"
 │
 ▼
CSR
 │
 ▼
Sectigo
```

A CSR contains information such as:

```text
Domain name
Organization
Country
Public key
Signature
```

Importantly:
**The private key is NOT sent to Sectigo.**
You keep the private key.

```text
Your server

Private Key
     │
     └── KEEP SECRET


CSR
     │
     └── Send to Sectigo
```

### 3. Sectigo

**Sectigo is a Certificate Authority (CA).**
A Certificate Authority is an organization trusted to issue SSL/TLS certificates.
Other examples include:

- Let's Encrypt
- DigiCert
- GlobalSign
- Sectigo

Think of Sectigo like a trusted authority saying: "I have verified that this certificate belongs to example.com."
So:

```text
You
 │
 │ CSR
 ▼
Sectigo
 │
 │ Certificate
 ▼
You
```

### 4. `.crt`

`.crt` generally means: Certificate file
For example: `example.crt`
It usually contains a certificate such as:

```text
-----BEGIN CERTIFICATE-----
MIID...
...
-----END CERTIFICATE-----
```

The certificate contains things like:

```text
Domain
Issuer
Validity period
Public key
Signature
```

A `.crt` file **does not normally contain your private key**.

### 5. `.pem`

This one causes a lot of confusion.
**PEM is a format/encoding, not necessarily a specific type of certificate.**
A PEM file is basically text that looks like:

```text
-----BEGIN CERTIFICATE-----
MIID...
...
-----END CERTIFICATE-----
```

You can have:

```text
certificate.pem
private-key.pem
chain.pem
```

So `.pem` doesn't tell you exactly what's inside.
For example: `certificate.pem`
could contain a certificate.
And: `private-key.pem`
could contain a private key.
The important part is: `-----BEGIN CERTIFICATE-----` or: `-----BEGIN PRIVATE KEY-----`

### 6. `.pfx`

`.pfx` is a **container**.
This is very different from `.crt`.
A PFX file can contain multiple things:

```text
                    .pfx
                     │
          ┌──────────┼──────────┐
          │          │          │
     Certificate  Private Key  Chain
```

For example:

```text
certificate
private key
intermediate certificates
```

That's why `.pfx` is convenient for moving an entire certificate setup between systems.
PFX is also commonly called: `PKCS#12` So: `.pfx = PKCS#12 container`
It is often password protected.

### 7. X.509

This is **not a file extension**.
X.509 is a standard that defines how certificates are structured.
Think:

```text
X.509
  │
  └── Rules for what an SSL certificate looks like
```

An X.509 certificate contains information such as:

```text
Subject
Issuer
Serial Number
Validity
Public Key
Extensions
Signature
```

So when someone says: "This is an X.509 certificate"
they're talking about the **certificate standard**, not necessarily a `.x509` file.

### 8. DER

DER is an **encoding format**.
There are two common ways you might encounter an X.509 certificate:

#### PEM

Human-readable text:

```text
-----BEGIN CERTIFICATE-----
MIID...
...
-----END CERTIFICATE-----
```

#### DER

Binary: `(binary data)`
So:

```text
X.509 certificate
       │
       ├── PEM encoded
       │
       └── DER encoded
```

You can think of it like:

```text
Same certificate
       │
       ├── PEM → text representation
       │
       └── DER → binary representation
```

### Put everything together

Here's the important relationship:

```text
                    SSL/TLS
                       │
              Certificate
                       │
                    X.509
                       │
              ┌────────┴────────┐
              │                 │
             PEM               DER
              │                 │
        certificate.pem    certificate.der
```

And separately:

```text
Cryptography
     │
     ├── RSA
     │
     └── ECC
```

Then:

```text
Private Key
     │
     └── used to create CSR
                    │
                    ▼
                   CSR
                    │
                    ▼
                 Sectigo
              (Certificate
               Authority)
                    │
                    ▼
              X.509 Certificate
                    │
             ┌──────┴──────┐
             │             │
            PEM           DER
```

And finally:

```text
Certificate + Private Key + Chain
              │
              ▼
             PFX
            (.pfx)
```


## openssl

`openssl` is a **command-line tool for working with SSL/TLS certificates, private keys, CSRs, encryption, and cryptographic operations**.

For example, if you're working with certificates like the **Sectigo `.pfx` / `.pem` / `.crt`** files, OpenSSL is commonly used to inspect and convert them.

### Basic OpenSSL commands

**1. Check OpenSSL version** `openssl version`
**2. View a certificate** `openssl x509 -in certificate.crt -text -noout`
This shows:

- Subject / domain
- Issuer
- Validity dates
- SANs
- Public key
- Signature algorithm

**3. Check a private key** `openssl rsa -in private.key -check`
For an ECC key: `openssl ec -in private.key -check`
**4. Create a CSR** `openssl req -new -key private.key -out certificate.csr`
**5. Convert PEM certificate to DER** `openssl x509 -in certificate.pem -outform DER -out certificate.der`
**6. Extract certificates/private key from a PFX** `openssl pkcs12 -in certificate.pfx -info`
Or extract the private key: `openssl pkcs12 -in certificate.pfx -nocerts -out private.key`
Extract the certificate: `openssl pkcs12 -in certificate.pfx -clcerts -nokeys -out certificate.crt`

### **Checking the SSL/TLS configuration of a website**.

The most important command is: `openssl s_client -connect example.com:443`
For a real website, for example: `openssl s_client -connect google.com:443`
This connects to the website's **HTTPS port 443** and performs a TLS handshake.

### What are you actually checking?

Think of it like:

```text
Your Linux machine
       │
       │ HTTPS / TLS
       ▼
   example.com
       │
       ├── Certificate
       ├── Certificate chain
       ├── TLS version
       ├── Cipher
       ├── Public key
       └── Server response
```

## Check a website's certificate

```bash
openssl s_client -connect example.com:443
```

You'll see information including:

```text
Certificate chain
Server certificate
SSL handshake
TLS version
Cipher
```

The output can be huge.
A very useful version is: `openssl s_client -connect example.com:443 -servername example.com`

#### Why `-servername`?

Because many websites share the same IP address.
For example:

```text
                    1 IP address
                         │
             ┌───────────┼───────────┐
             │           │           │
          site1.com   site2.com   site3.com
```

NGINX can use **SNI (Server Name Indication)** to determine which certificate to return.
So: `-servername example.com`
tells the server: "I want the TLS certificate for example.com."

### Show only the certificate

You can pipe the output to `openssl x509`: `openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -text -noout`

Now you can inspect:

```text
Subject
Issuer
Validity
Public Key
SAN
Signature Algorithm
```

For example:

```text
Issuer: Sectigo
Subject: example.com
Not Before: ...
Not After : ...
DNS:example.com
DNS:www.example.com
```

### Check certificate expiry

A much simpler command: `openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -dates`
You'll get something like:

```text
notBefore=Jan 10 00:00:00 2026 GMT
notAfter=Jan 10 23:59:59 2027 GMT
```

So you can quickly check: When does this website's certificate expire?

### Check who issued the certificate

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -issuer
```

Example: `issuer=C=GB, O=Sectigo Limited, CN=Sectigo ...`
This tells you the **Certificate Authority**.

### Check the domain on the certificate

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -subject
```

And: `openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -ext subjectAltName`
The second command is particularly useful because modern certificates use **SAN (Subject Alternative Name)**.
You might see:

```text
DNS:example.com
DNS:www.example.com
DNS:api.example.com
```

### Check TLS version

You can explicitly test TLS versions.
TLS 1.2 `openssl s_client -connect example.com:443 -tls1_2`
TLS 1.3 `openssl s_client -connect example.com:443 -tls1_3`
If the connection succeeds, the server supports that protocol version.

### Check the certificate chain

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

You may see:

```text
Certificate chain
 0 s:CN=example.com
   i:CN=Intermediate CA

 1 s:CN=Intermediate CA
   i:CN=Root CA
```

This is particularly important when you're troubleshooting things like:

```text
Sectigo
    ↓
Intermediate certificate
    ↓
Root certificate
```

If the chain is incorrectly configured on NGINX, some clients may report certificate errors.

### Check a website and see the negotiated TLS details

A useful command: `openssl s_client -connect example.com:443 -servername example.com </dev/null`
Look near the bottom for something like:

```text
Protocol  : TLSv1.3
Cipher    : TLS_AES_256_GCM_SHA384
```

That tells you: **Protocol** `TLSv1.3`
**Cipher suite** `TLS_AES_256_GCM_SHA384`

