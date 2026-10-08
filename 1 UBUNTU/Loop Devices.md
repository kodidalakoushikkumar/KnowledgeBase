

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