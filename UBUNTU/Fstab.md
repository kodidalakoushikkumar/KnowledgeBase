---
Title: fstab
Type:
Topic:
tags:
cover: "[[/IMAGES/fstab.png]]"
---
# What is `fstab`?

`fstab` = File System Table
Location: `/etc/fstab`
It tells Linux: Which disk/partition, Where to mount it, Which filesystem, With what options, At boot time
Think of it as Linux’s auto-mount configuration file.
#### Example `fstab` entry
`/dev/mynew_vg/vol01   /foobar   ext4   defaults   0 0`
This line means: “Mount this disk at `/foobar` every time the system boots.”
#### `fstab` has 6 columns (VERY IMPORTANT)
`<device>   <mount point>   <fs type>   <options>   <dump>   <fsck>`
Let’s break them
#### 1️ Device (WHAT to mount)
##### First field – The block device
The first field in each fstab entry holds information about the local or remote block device which should be mounted. The most typical way to reference a block device is by using its node inside the `/dev` directory, so for example to reference the first partition of the `sda` block device we use `/dev/sda1` as value.
Examples: `/dev/sda1 /dev/mynew_vg/vol01 UUID=3f2a-9c1e`
Best practice: **UUID**   ➖ Why? Device names can change.
#### 2️ Mount point (WHERE)
##### Second field – The mountpoint
In each fstab entry, the second field specifies the `mountpoint` for the filesystem: what directory in the system should be used to access its content. This should be always provided except if the block device we are referencing it’s used as swap. In that case `"none"` should be used. `/  /home  /var  /foobar` . Must already exist (`mkdir`).
#### 3️ Filesystem type
##### Third field – The filesystem type
The third field of an fstab entry specifies the type of filesystem in use on the raw block device or partition. The filesystem must be among the ones supported by the operating system like, for example ext4, xfs etc. In case of a remote filesystem we can use, for example `cifs` as the value of this field if the filesystem is shared via samba or `nfs` if it is shared via the `Network File System`.
Examples: `ext4 xfs ntfs vfat swap`
#### 4️ Options (HOW)
##### Fourth field – Mount options
The fourth field of each entry in the fstab file is used to provide a list of options to be used when mounting the filesystem. To use the default set of mount options we specify `default` as a value. Default options are:
- `rw` (read-write);
- `suid` (respect setuid and setgid bits);
- `dev` (interpret characters and block devices on the filesystem);
- `exec` (allow executing binaries and scripts);
- `auto` (mount the filesystem when the -a option of the mount command is used);
- `nouser`(make the filesystem not mountable by a standard user);
- `async` (perform I/O operations on the filesystem asynchronously).
Common ones: `defaults ro        (read-only) rw        (read-write) noexec nosuid nodev`
Most beginners use: `defaults`
#### 5️ Dump (backup)
##### Fifth field – Should the filesystem be dumped ?
The fifth field in each entry can be either 0 or 1. The value is used by the dump backup program (if installed) to know what filesystem should be dumped. `0 = don’t backup 1 = backup`
Almost always: `0`
#### 6️ Fsck (filesystem check order)
##### Sixth field – Fsck order
The sixth field is used to establish the order by which another utility, `fsck`, should check filesystems on boot. The value of `1` must always be used for the root filesystem; for all the others we can use `2`. If this value is not provided it defaults to 0, and the filesystem will not be checked. `0 = never check 1 = check first (root /) 2 = check later`
Typical: `/      → 1 others → 2`
