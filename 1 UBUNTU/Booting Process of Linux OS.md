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