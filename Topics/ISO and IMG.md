# 6️⃣ ISO and IMG

|Type|What it contains|Think of it as|Common uses|
|---|---|---|---|
|**`.txt`, `.jpg`, `.pdf`**|**Data**|A normal file|Storing text, images, documents, etc.|
|**`.zip`**|**Multiple files/directories + compression + archive metadata**|A package/container of files|Easy transfer, storage, backups|
|**`.iso`**|**Data + filesystem/disc structure + boot information if present**|A **disc image**|OS installation media, bootable USB creation, Virtual Machines, DVD/CD backup|
|**`.img`**|**Raw data + filesystem(s) + partitions/partition table + boot information + other disk-level structures** _(depending on the image)_|A **disk/storage image**|Disk cloning, full-disk backup, OS deployment, recovery, Raspberry Pi/SD-card images|
|**Physical USB/SSD/SD card**|Data + filesystem + partitions + boot information + disk structures|Actual storage device|Store and boot operating systems, applications, files|

#### The simplest mental model

|File|Think|
|---|---|
|`.txt`|**"Here is data."**|
|`.jpg`|**"Here is image data."**|
|`.zip`|**"Here are multiple files packaged together."**|
|`.iso`|**"Here is a disc."** → Can become a **virtual DVD/CD** when mounted|
|`.img`|**"Here is a storage device."** → Can reproduce a **USB/SD/HDD/SSD**|

#### Example

```text
Ubuntu ISO

ubuntu.iso
    │
    ├── Data
    ├── Files/directories
    ├── Filesystem structure
    └── Boot information
             │
             ▼
       Installation media
```

```text
Full Disk Image

disk.img
    │
    ├── Partition table
    ├── Partition 1
    │     └── Filesystem + data
    ├── Partition 2
    │     └── Filesystem + data
    ├── Boot information
    └── Other disk-level information
             │
             ▼
       Clone / Backup / Restore
```

**One correction:** don't assume every `.img` contains partitions. `.img` is a generic extension; it _can_ be a whole-disk image, a single-partition image, or another type of image.

## Making bootable usb from .iso
### 1. Making a bootable USB from an `.iso`

Suppose you have: `ubuntu.iso`
The ISO contains something like:

```text
ubuntu.iso
│
├── Boot information
├── EFI/
├── boot/
├── casper/
├── filesystem/data
└── other installation files
```

You use something like Rufus, Balena Etcher, or `dd` to **write the image to the USB**.
Conceptually:

```text
ubuntu.iso
     │
     │ WRITE IMAGE
     ▼
┌──────────────────────────┐
│ USB                      │
│                          │
│ Partition structure      │
│ EFI / boot information   │
│ Ubuntu installation data │
│ Filesystem               │
└──────────────────────────┘
```

Now the computer can boot from the USB.

#### What happens when you boot?

```text
Computer
   │
   ▼
BIOS/UEFI
   │
   ▼
USB
   │
   ▼
Bootloader
   │
   ▼
Ubuntu installer
   │
   ▼
Install Ubuntu onto internal SSD
```

**Important:** the ISO usually contains the **installer environment**, not your final installed computer.

### 2. Now consider your `.img` example

Suppose you have: `company-laptop.img`
And this image represents an already-installed Linux OS:

```text
company-laptop.img
│
├── Partition table
│
├── EFI partition
│   └── Bootloader
│
└── Linux partition
    ├── /bin
    ├── /etc
    ├── /home
    ├── /usr
    ├── Applications
    └── User configuration
```

Maybe the user has:

```text
Ubuntu
├── Chrome
├── VS Code
├── Docker
├── Company applications
├── Configurations
└── User files
```

This is very different from an Ubuntu installation ISO.
The `.img` is essentially a **snapshot/image of an already-installed system**.

### 3. How do you install/restore that `.img`?

You generally don't "install" an `.img` the same way you install an ISO.
You **write/restore the image to a target disk**.
For example:

```text
company-laptop.img
        │
        │ RESTORE / WRITE
        ▼
     Target SSD
        │
        ▼
┌──────────────────────────┐
│ Partition table          │
│ EFI partition            │
│ Linux partition          │
│ OS                       │
│ Applications             │
│ Configurations           │
│ User data                │
└──────────────────────────┘
```

Afterward, assuming the image and hardware are compatible and the boot setup is correct:

```text
Power ON
   ↓
UEFI
   ↓
Bootloader
   ↓
Linux
   ↓
Applications already installed
```

You **don't need to run the Ubuntu installer again** because the OS is already inside the image.

### 4. Example using Linux

Suppose: `company-laptop.img` is your image.
Your target SSD is: `/dev/nvme0n1`
A raw image could be restored with something like: `sudo dd if=company-laptop.img of=/dev/nvme0n1 bs=4M status=progress`
Conceptually:

```text
company-laptop.img
       │
       │ bytes
       ▼
/dev/nvme0n1
       │
       ├── partition table
       ├── EFI
       └── Linux filesystem
```

**This completely overwrites the target disk**, so you have to be extremely careful about identifying the correct device.
Tools such as Clonezilla are often preferable for real deployment because they provide a safer workflow and additional cloning/recovery features.

### 5. Here's the BIG difference between ISO and IMG

Imagine you want to deploy Ubuntu to 20 laptops.

#### Method A — ISO

You have: `ubuntu.iso`
For each laptop:

```text
USB
 ↓
Boot installer
 ↓
Install Ubuntu
 ↓
Install applications
 ↓
Configure applications
 ↓
Configure settings
```

You repeat the configuration work.

#### Method B — Preconfigured disk image

You prepare **one perfect laptop**:

```text
Ubuntu
Chrome
VS Code
Docker
Company applications
Configurations
Settings
```

Then create: `company-laptop.img`
Now:

```text
             company-laptop.img
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Laptop 1   Laptop 2   Laptop 3
          │          │          │
          ▼          ▼          ▼
       Same OS    Same OS    Same OS
       Same apps  Same apps  Same apps
       Same config Same config Same config
```

That's the basic idea behind **disk imaging/deployment**.

### 6. But there is an important catch

You cannot blindly take one `.img` and put it on every computer.
For example:

```text
Image created on:
Dell Laptop A
```

may have:

```text
Dell-specific drivers
Hardware configuration
Boot configuration
Disk size assumptions
Machine-specific settings
```

When deploying to different hardware, you may need to account for:

- Different disk sizes
- Different hardware/drivers
- UEFI/BIOS configuration
- Partition layout
- Hostname
- Machine identity
- User accounts
- Encryption
- Network configuration

Enterprise imaging tools often handle some of these issues.

### 7. One more important distinction: ISO can also contain a complete OS

Don't think:

```text
ISO = installer only
IMG = installed OS only
```

That's not universally true.
An ISO **can contain a complete bootable/live OS**.
For example:

```text
Ubuntu Live ISO
       ↓
Boot USB
       ↓
Ubuntu runs directly from USB
```

You haven't necessarily installed it onto the SSD.
An `.img` can also contain an OS, but its structure may represent the actual disk/partition layout.

### 8. Think about it like this

#### ISO

```text
                 ubuntu.iso
                     │
                     ▼
              Installation/
               Live Media
                     │
                     ▼
                Target SSD
                     │
                     ▼
             Installed Ubuntu
```

**Purpose:** provide **media from which you install/boot**.

#### IMG

```text
             company.img
                  │
                  │ restore
                  ▼
             Target SSD
                  │
                  ▼
       Already-installed system
                  │
          ┌───────┼────────┐
          ▼       ▼        ▼
         OS      Apps    Config
```

**Purpose:** reproduce an **existing disk/system**.

### The easiest way to remember

**ISO → "I want to install/boot something."**
*IMG → "I already have a configured storage/system and want to reproduce/restore it."**


1