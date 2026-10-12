

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


