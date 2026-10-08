
# 8️⃣ Partitioning the 32 GB MicroSD Card

➖ Open DiskPart
➖ Press **Win + X → Terminal (Admin)** and run: `diskpart`
➖ Find your microSD :  `list disk`
➖ Identify the **32 GB** card.
➖ Select it
➖ For example, if it's Disk 1: `select disk 1`
➖ Then verify: `detail disk`
➖ Make sure it is actually your microSD.
➖ Clear any DiskPart attributes
➖ Run: `attributes disk clear readonly`
➖ Erase the partition table : `clean`
➖ Then make it **MBR**: `convert mbr`
➖ Create exactly a 2 GB partition : `create partition primary size=2048`
➖ Format it as FAT16 : `format fs=fat quick label=CAMERA`
➖ On Windows, `fs=fat` creates the older **FAT filesystem (FAT16)** for a partition this size.
➖ Give it a drive letter : `assign`
➖ Then: `exit`
➖ Your card should now appear in Windows as approximately **2 GB**, formatted FAT16, with the remaining ~30 GB unallocated.

