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


