

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
