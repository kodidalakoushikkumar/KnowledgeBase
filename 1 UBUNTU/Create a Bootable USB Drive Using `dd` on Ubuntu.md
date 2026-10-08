# Create a Bootable USB Drive Using `dd` on Ubuntu

Find your USB drive : `lsblk`
Unmount the USB : `sudo umount /dev/sdb1`
Write the ISO to the USB : `sudo dd if=/path/to/file.iso of=/dev/sdb bs=4M status=progress conv=fsync`
Example: `sudo dd if=ubuntu.iso of=/dev/sdb bs=4M status=progress conv=fsync`
Wait until it finishes, Then run: `sync`
Safely eject : `sudo eject /dev/sdb
**Important:** Replace `/dev/`