# 3️⃣0️⃣ chntpw

If you are using **Linux and have connected a Windows SSD as a secondary drive**, `chntpw` can be used to inspect or modify the **local Windows account database (SAM)**.

This applies to a Windows installation where you have authorization to recover the account.

### 1. Find the Windows SSD

Run: `lsblk -f`
You might see something like:

```
nvme0n1
├─nvme0n1p1   vfat
├─nvme0n1p2   ntfs
└─nvme0n1p3   ntfs
```

Identify the large **NTFS partition containing Windows**.
You can also check: `sudo blkid`

### 2. Mount the Windows partition

For example, if the Windows partition is `/dev/nvme0n1p3`:

```
sudo mkdir /mnt/windows
sudo mount /dev/nvme0n1p3 /mnt/windows
```

Check that it is actually Windows: `ls /mnt/windows`
You should see folders such as:

```
Windows
Users
Program Files
ProgramData
```

### 3. Locate the SAM file

The local Windows account database is normally: `/mnt/windows/Windows/System32/config/SAM`
There is also a related SYSTEM registry hive: `/mnt/windows/Windows/System32/config/SYSTEM`
You can verify: `sudo ls -l /mnt/windows/Windows/System32/config/SAM`

### 4. Use `chntpw`

First install it if necessary: `sudo apt install chntpw`
Then point it at the Windows SAM: `sudo chntpw -l /mnt/windows/Windows/System32/config/SAM`
The `-l` option lists the local accounts contained in the SAM.

For example, you may see:

```
RID          Username
--------------------------------
01f4         Administrator
03e8         Koushik
```

For an authorized recovery, `chntpw` can then be used to modify a **local account**.

### What `chntpw` can modify

When you run: `sudo chntpw -i /mnt/windows/Windows/System32/config/SAM`
you enter its interactive account-management interface. The exact menu can vary by version.
For a selected **local account**, it can generally provide options such as:

|Operation|What it means|
|---|---|
|Clear password|Removes the local account's password requirement|
|Set/change password|Allows a new password to be assigned in versions/setups that support it|
|Unlock account|Removes certain account-lock/disabled state|
|Enable account|Enables a disabled local account|
|Disable account|Disables the account|
|Promote to Administrator|Changes the account's local administrative membership/status in supported cases|
|Edit account information|Modify certain account flags/properties|
|List accounts|View accounts and their RIDs without changing them|

`chntpw` performs these through its **interactive menu** rather than giving each operation a simple one-line command.
First identify the Windows partition and SAM: `sudo mount /dev/nvme0n1p3 /mnt/windows`
Then list the accounts: `sudo chntpw -l /mnt/windows/Windows/System32/config/SAM`
For the actual account editor: `sudo chntpw -i /mnt/windows/Windows/System32/config/SAM`
You'll get a menu similar to:

```
===== chntpw Main Interactive Menu =====

1 - Edit user data and passwords
2 - Syskey status & change
3 - RecoveryConsole settings
4 - Registry editor
q - Quit
```

Choose: 1
Then select the user by its RID/username. The subsequent account menu contains the available password/account operations.

### Important correction

Some of the items in the table I gave you earlier are **not separate `chntpw` commands**. In particular, `chntpw` is primarily an **offline SAM/password/account-data editor**; it is not the normal tool for managing Windows local-group membership.

For example, for legitimate IT administration, these are normally done **from Windows itself**:

```
net user
net user username *
net user username /active:yes
net user username /active:no
net localgroup administrators username /add
net localgroup administrators username /delete
```
