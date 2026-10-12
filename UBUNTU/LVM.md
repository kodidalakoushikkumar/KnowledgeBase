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

