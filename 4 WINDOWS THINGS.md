# 1️⃣ Network Issue

```
ipconfig /release  
ipconfig /renew  
ipconfig /flushdns  
netsh winsock reset  
netsh int ip reset
```

# 2️⃣ slmgr (Software Licensing Management Tool)

```
slmgr /dlv      :: Full licensing details
slmgr /xpr      :: Activation expiration/permanent status
slmgr /dli      :: Basic license information
slmgr /ato      :: Activate Windows
slmgr /ipk KEY  :: Install a product key
slmgr /upk      :: Uninstall the current key
slmgr /cpky     :: Remove the key from the registry
wmic path softwarelicensingservice get OA3xOriginalProductKey
```

# 3️⃣ GPO

There are two different types of Group Policy in Windows:

1. **Local Group Policy (LGPO)** – Stored on the laptop itself.
2. **Domain Group Policy (GPO)** – Stored in Active Directory on the Domain Controller.

This distinction is very important.
### Case 1: Local Group Policy

If the setting is configured using: `gpedit.msc` on the laptop itself, then it is a **Local Group Policy**. A **local administrator** can:

- Open `gpedit.msc`
- Change the settings
- Run `gpupdate /force`

The changes apply immediately.
Example:

```
Computer Configuration
    Administrative Templates
        Windows Components
            BitLocker Drive Encryption
```

A local admin can change every setting here if it's only a Local GPO.

### Case 2: Domain Group Policy

If your organization configured BitLocker from: `Group Policy Management (gpmc.msc)` on the Domain Controller, then the policy is downloaded from Active Directory.
Example:

```
Domain Controller
        │
        │
        ▼
    Domain GPO
        │
        ▼
    Laptop
```

The laptop only receives the policy. A local administrator **cannot permanently change** those settings. Suppose the Domain GPO says: `Require BitLocker = Enabled` , You log in as Local Administrator.
You open `gpedit.msc`. You change it to `Disabled` It may appear changed temporarily. But after: `gpupdate /force` or `Computer Restart` or `90–120 minutes (default background refresh)` the Domain GPO reapplies and overwrites your change.

```
Domain Controller
        │
        ▼
Require BitLocker = Enabled
        │
        ▼
	Laptop

You changed it to Disabled
↓
gpupdate
↓
Enabled again
```

### Can Local Admin edit gpedit.msc?

Yes. Being in the local Administrators group gives permission to edit the Local Group Policy. It does **not** give permission to modify the Domain GPO stored in Active Directory.
### Can a user in the local Administrators group change Domain GPO?

No. They can only:

- View the policies
- Attempt to override Local Policy
- Disable services temporarily (if permitted)
- Modify registry values (if not protected)

But the Domain GPO will reapply and win.
### How do I know whether a policy is Local or Domain?

Run: `gpresult /r` or `gpresult /h C:\gp.html` Open `gp.html`. You'll see something like:

```
Applied Group Policy Objects

Default Domain Policy
BitLocker Policy
Security Baseline
```

If you see named domain GPOs, the setting came from Active Directory. If it only shows: `Local Group Policy`
then it's local.

### Where are these policies stored?

Local GPO :  `C:\Windows\System32\GroupPolicy\` and `C:\Windows\System32\GroupPolicyUsers\`

Domain GPO

Stored on the Domain Controller under the SYSVOL share, for example: `\\domain.com\SYSVOL\domain.com\Policies\`
Each GPO has its own GUID folder, and clients download them during policy refresh.
# 4️⃣ Windows Hello for Business PIN

Many people think the PIN is a backup to the domain password, but it actually works differently.
## How it works

When your laptop is joined to an Active Directory domain or Microsoft Entra ID (Azure AD), you normally sign in with:

- **Domain username + password**
- 
    ```
    COMPANY\john
    Password123!
    ```
    

You can also configure a **PIN**, for example: `PIN: 562481`
At the login screen you'll have options like:

- PIN
- Password
- Fingerprint (if available)
- Face Recognition (if available)

## Is the PIN a replacement for the password?

**No.** The PIN is tied to **that specific device only**.

Example:
Laptop A

- Domain Password: `Welcome@123`
- PIN: `562481`

Laptop B

- Same domain account
- PIN `562481` **will NOT work**

Each device has its own PIN.
## If the domain is having issues, can I still log in?

### Scenario 1: Domain Controller is offline ✅

Yes.  If you've logged into the laptop before, Windows stores cached domain credentials. You can log in using: Domain Password, PIN
even if the domain controller isn't reachable.

### Scenario 2: Network disconnected

Still works. Windows uses cached credentials.
### Scenario 3: Password recently changed

If the laptop hasn't contacted the domain controller after the password change:

- Old cached password may still work.
- PIN usually continues to work because it's tied to the device.

## How to enable a PIN on a domain laptop

### Method 1 (Windows Hello for Business enabled)

1. Open **Settings**
2. **Accounts**
3. **Sign-in options**
4. Click **PIN (Windows Hello)**
5. Click **Set up**
6. Enter your domain password
7. Create your PIN

Done.

## What if "PIN" is missing?

In many organizations, it's disabled by Group Policy. The IT administrator must enable **Windows Hello for Business**. Common Group Policy settings are:

```
Computer Configuration
 └ Administrative Templates
     └ Windows Components
         └ Windows Hello for Business
             ├ Use Windows Hello for Business = Enabled
             ├ Configure PIN Complexity
             └ Use Biometrics
```

Or under:

```
Computer Configuration
 └ Administrative Templates
     └ System
         └ Logon
```

depending on the Windows version and deployment.

## Can I set it manually using CMD or PowerShell?

No. There is **no command like**: `net user pin 1234` or `Set-PIN`. The PIN is created through Windows Hello after the feature has been enabled by policy.

## If you are the Domain Administrator

If you want **all domain users** to have the option to use a PIN:

1. Enable **Windows Hello for Business** in Group Policy.
2. Configure PIN policies (length, complexity, expiration if desired).
3. Run: `gpupdate /force`
4. Users then go to:

```
Settings
→ Accounts
→ Sign-in options
→ PIN (Windows Hello)
```

and create their own PIN.

### Important distinction

A Windows Hello PIN is **not** an "IT emergency PIN" that lets you bypass the domain. It's a secure, device-specific sign-in method backed by cryptographic keys. If the laptop has previously authenticated the user, the PIN can still be used when the domain controller is temporarily unavailable, but it does **not** replace the domain account or provide access if the device has never been set up for that user.

# 5️⃣ Bitlocker

### **Prerequisites and Verification**

Before starting, ensure your system has **TPM 2.0 enabled**, is in **UEFI Boot Mode**, and you have **Local Administrator privileges**.

1. **Verify TPM Status:**
    - Press `Windows + R`, type **tpm.msc**, and press Enter.
    - Confirm the status is "The TPM is ready for use" and the version is 2.0.
2. **Verify UEFI Mode:**
    - Press `Windows + R`, type **msinfo32**, and press Enter.
    - Confirm "BIOS Mode" is set to **UEFI**.
3. **Check Current BitLocker Status:**
    - Open **Command Prompt as Administrator** and run: `manage-bde -status`
    - The "Protection Status" should be **Off** before you begin.

### **Configuration Steps**

1. **Create a Recovery Key:**
    - Open **PowerShell as Administrator** and run: `Add-BitLockerKeyProtector -MountPoint "C:" -RecoveryPasswordProtector`
    - **Crucial:** Save the 48-digit recovery key provided in a secure location.
    - Verify it was added by running: `manage-bde -protectors -get C:`.
2. **Configure Group Policy for PIN:**
    - Run **gpedit.msc** and navigate to: `Computer Configuration > Administrative Templates > Windows Components > BitLocker Drive Encryption > Operating System Drives`.
    - Open **"Require additional authentication at startup"**, set it to **Enabled**, and select **"Require startup PIN with TPM"**.
    - Apply the changes and force a policy update by running: **gpupdate /force**.
3. **Add the TPM + PIN Protector:**
    - In **Command Prompt (Admin)**, run: `manage-bde -protectors -add C: -TPMAndPIN`
    - Enter and confirm your chosen PIN (an 8-digit numeric PIN is recommended).
4. **Enable BitLocker Encryption:**
    - Run the following command to start encryption (using "used space only" for speed): `manage-bde -on C: -usedspaceonly`.
5. **Reboot and Validate:**
    - Restart the laptop: `shutdown /r /t 0`.
    - The BitLocker pre-boot screen should appear; enter your PIN to boot into Windows.
6. **Final Verification:**
    - Once logged in, check the status: `manage-bde -status C:`
    - Success is confirmed if "Percentage Encrypted" is **100%** and "Protection Status" is **On**.

### **Additional Commands**

- **Encrypt a Secondary Drive (D:):**
    - `manage-bde -on D: -usedspaceonly`
    - `manage-bde -protectors -add D: -RecoveryPassword`.
- **Change your BitLocker PIN:**
    - `manage-bde -changepin C:`.

# 6️⃣ Delete the User Profile in WINDOWS
### Method 1 (GUI) - Delete the User Profile (Recommended)

Log in as **another administrator** (not the user you're deleting).

1. Press **Win + R**
2. Type: `sysdm.cpl`
3. Go to `Advanced`
4. Under **User Profiles**, click `Settings`
5. Select the domain user's profile, for example: `DOMAIN\john`
6. Click **Delete**

This removes:

- `C:\Users\john`
- Desktop
- Documents
- Downloads
- AppData
- Registry profile

Basically the equivalent of **Delete User + Home Folder** on macOS.

### Method 2 (Settings)

```
Settings
→ System
→ About
→ Advanced system settings
→ User Profiles
```

Same thing.

### Method 3 (Command Line)

If the profile still exists:
List profiles: `Get-CimInstance Win32_UserProfile | Select LocalPath, SID`
Delete one: `Get-CimInstance Win32_UserProfile Where-Object {$_.LocalPath -eq "C:\Users\john"} Remove-CimInstance`
or `Remove-WmiObject -Class Win32_UserProfile`
This removes the profile completely.

### Method 4 (If the profile is already gone but the folder remains)

Delete the folder: `rmdir /s /q "C:\Users\john"` or `Remove-Item "C:\Users\john" -Recurse -Force`

### If "Delete" button is greyed out

This usually means:

- You're logged in as that user.
- The user is still signed in.
- Some process is using the profile.

Check: `query user`
Log them off: `logoff <sessionID>`
or restart the laptop and log in as another admin.

### Does this remove the domain user?

No. It only removes the **local cached profile**. The domain account still exists in Active Directory. If the user signs in again, Windows will simply create a brand-new profile under: `C:\Users\john`

# 7️⃣ Windows Junk CleanUp

If `C:\Windows\WinSxS` and `C:\Windows\SoftwareDistribution\Download` together are using around **100 GB**, that's much larger than expected in most cases. The good news is that you **shouldn't manually delete WinSxS contents**, but there are safe ways to reclaim space.
### 1. Check the actual size of WinSxS \(recommended first\)

Windows Explorer often **overstates** the size of `WinSxS` because of hard links.
Open **Command Prompt as Administrator** and run: `Dism.exe /Online /Cleanup-Image /AnalyzeComponentStore`
This tells you:

- Actual size
- Whether cleanup is recommended
- Potential reclaimable space
### 2. Clean up WinSxS safely

If cleanup is recommended: `Dism.exe /Online /Cleanup-Image /StartComponentCleanup`
For a more aggressive cleanup \(prevents uninstalling old updates\): `Dism.exe /Online /Cleanup-Image /StartComponentCleanup /ResetBase`
**Use **`/ResetBase`** only if you're comfortable giving up the ability to uninstall currently installed Windows updates.**
### 3. Clean `SoftwareDistribution\Download`

This folder stores downloaded Windows Update files. You can safely clear it.
Method 1 \(recommended\)
Open Command Prompt as Administrator:

```
net stop wuauserv
net stop bits
```

Then delete everything inside: `C:\Windows\SoftwareDistribution\Download` , Do **not** delete the `SoftwareDistribution` folder itself.
Then restart services:

```
net start bits
net start wuauserv
```

Windows will recreate anything it needs.
### 4. Run Disk Cleanup

Press **Win + R** RUN: `cleanmgr`
Choose **C:**
Click **Clean up system files**.
Check:

- Windows Update Cleanup
- Delivery Optimization Files
- Temporary Windows installation files
- Temporary files
- Previous Windows installation \(Windows.old\), if present

This often frees several GB.
### 5. Check for Windows.old

Look for: `C:\Windows.old`
This alone can consume **20–40 GB**.
Remove it through Disk Cleanup rather than deleting it manually.
### 6. Check if updates are stuck

If `SoftwareDistribution\Download` alone is tens of GB, Windows Update may have repeatedly downloaded failed updates.
Run: `Get-WindowsUpdateLog`
or check: `Settings → Windows Update` for failed updates.

If WinSxS is still huge \(50–80 GB\)
Run: `Dism /Online /Cleanup-Image /RestoreHealth`
followed by: `sfc /scannow`
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

# 9️⃣ Collect info (Service, Process, Startup)

**Windows system enumeration**. Collect information about:

1. **Running processes**
2. **Whether process executables are digitally signed**
3. **Programs configured to start automatically**
4. **Currently running services**

This is useful for **troubleshooting, malware investigation, incident response, and general system auditing**.

```
Get-CimInstance Win32_Process |
Select-Object Name, ProcessId, ParentProcessId, ExecutablePath, CommandLine |
Format-List |
Out-File "$env:USERPROFILE\Desktop\processes.txt" -Width 300
```

```
Get-CimInstance Win32_Process |
Where-Object { $_.ExecutablePath } |
ForEach-Object {
    $sig = Get-AuthenticodeSignature $_.ExecutablePath

    [PSCustomObject]@{
        Process = $_.Name
        PID     = $_.ProcessId
        Path    = $_.ExecutablePath
        Status  = $sig.Status
        Signer  = $sig.SignerCertificate.Subject
    }
} |
Out-File "$env:USERPROFILE\Desktop\process_signatures.txt" -Width 300
```

```
Get-CimInstance Win32_StartupCommand |
Select-Object Name, Command, Location, User |
Format-List |
Out-File "$env:USERPROFILE\Desktop\startup.txt" -Width 300
```

```
Get-CimInstance Win32_Service |
Where-Object { $_.State -eq "Running" } |
Select-Object Name, DisplayName, State, StartMode, PathName |
Format-List |
Out-File "$env:USERPROFILE\Desktop\services.txt" -Width 300
```

# 1️⃣0️⃣ Windows Activation Transfer

**OLD LAPTOP**
Open **Command Prompt as Administrator**: `slmgr /dlv`
If it's an OEM key stored in the laptop's firmware, run: `wmic path softwarelicensingservice get OA3OriginalProductKey`
You can also use PowerShell: `(Get-CimInstance -ClassName SoftwareLicensingService).OA3OriginalProductKey`
Open **Command Prompt as Administrator**: `slmgr /upk`
Then: `slmgr /cpky`
This removes the installed product key from the old Windows installation.

**NEW LAPTOP**
Then open **Command Prompt as Administrator**: `slmgr /ipk XXXXX-XXXXX-XXXXX-XXXXX-XXXXX`
Then activate: `slmgr /ato`
Check activation: `slmgr /xpr`

# 1️⃣1️⃣ DNS in AD

**Active Directory literally cannot function without DNS.**  
It’s not optional—it’s the backbone that lets everything find everything else.
#### Think of it like this

**Active Directory (AD)** = a big office with employees (users, computers, services)
**DNS** = the office directory / contact list
Without a directory, nobody knows: who is where, how to reach them, which service to talk to
#### The core reason: AD is “DNS-driven”

When a Windows machine tries to do anything in AD (login, join domain, find a service), it **does NOT search randomly**.
It asks DNS: “Hey, where is the Domain Controller?”
#### What actually happens (real flow)

Let’s say your laptop joins a domain:
##### Step 1: It queries DNS

It looks for special records called **SRV records**, like: `_ldap._tcp.dc._msdcs.yourdomain.com`
##### Step 2: DNS replies

DNS says: “Your Domain Controller is at 10.50.10.5”
##### Step 3: Client connects

Now your system can:

- authenticate
- apply group policies
- access resources

Without DNS → it doesn’t even know where the Domain Controller is.
#### What breaks without DNS

If DNS is missing or wrong:

- ❌ Domain login fails
- ❌ Group Policy won’t apply
- ❌ Domain join fails
- ❌ Kerberos authentication breaks
- ❌ Services like LDAP can’t be located

Basically: **AD becomes useless**
#### Why not use IP instead?

Good question—and this is where many get confused.
You _could_ hardcode IPs, but:

- Domain Controllers can change
- Multiple DCs exist (load balancing)
- Services are dynamic

DNS gives:

-   flexibility
-  load distribution
-  service discovery
#### Key concept you must remember

Active Directory doesn’t “find things” itself. It **depends on DNS to locate everything**
That’s why when you install AD DS: DNS role is installed automatically (or required)
#### Real-world example (your work scenario)

When you log into your office laptop:

1. System asks DNS → “Where is DC?”
2. DNS responds
3. System contacts DC
4. You get logged in

If DNS is broken:  You’ll see **“domain not available” errors**
## How does DNS “know” where AD is?

DNS doesn’t magically know. **Active Directory registers itself in DNS.**
When you install a Domain Controller, It automatically creates DNS records like:
A record → maps name to IP, SRV record → tells “what service is where”
Example: `dc1.company.local  → 10.50.10.5   (A record)_ldap._tcp.dc._msdcs.company.local → dc1.company.local  (SRV record)`
That SRV record literally means: “If you need LDAP (AD service), go to this server”
So DNS is like:

- Not just a phonebook
- More like a **service registry**
## What is special about these SRV records?

This is the key piece most articles don’t explain well.
Normal DNS: `google.com → IP`
AD DNS: `Service → Server → IP`
So instead of asking: “What is the IP of AD?”
The client asks: “Who provides LDAP / Kerberos for this domain?”
That’s why DNS is essential—it answers _service-level questions_, not just names.
## Why can’t Windows just talk directly to AD?

Because it doesn’t know:

- which server is the Domain Controller
- how many DCs exist
- which one is closest / available
### Imagine this situation:

You have:

- 3 Domain Controllers
- Different locations
- Some may be down

If you hardcode one IP:

- ❌ No failover
- ❌ No load balancing
- ❌ Breaks easily

DNS solves this by:

- returning multiple DCs
- prioritizing based on site/location
## What actually happens during login

Let’s walk it simply:
Step 1: Your PC asks DNS: `Where is LDAP for company.local?`
Step 2: DNS replies: `dc1.company.localdc2.company.local`
Step 3: Now your PC connects to one of them. DNS is just the **discovery step**
## The real reason you cannot skip DNS

Even if you _know_ the IP:
AD still depends on:

- Kerberos (needs proper names, not just IP)
- Service discovery (LDAP, GC, etc.)
- Domain-based communication

These rely on **names + DNS structure**, not raw IP.
### Key insight

DNS is not just resolving names in AD. It is how clients _discover services dynamically_
## Does DHCP “have” DNS?

**No — they are separate services**, but they usually **work together**.

- **DHCP** → gives you: IP address, Subnet, Gateway, **DNS server IP**
- **DNS** → answers: “What is the IP of this name?”, “Where is this service?”

So DHCP doesn’t replace DNS.  It just tells your computer _which DNS server to use_.
## What actually happens when your PC starts

### Step 1: DHCP process (first thing)

Your PC broadcasts: “I need an IP!”
DHCP replies:

- IP: `192.168.1.50`
- Gateway: `192.168.1.1`
- DNS: `192.168.1.10`

Now your PC knows:  “If I need name resolution, go to this DNS server”
### Step 2: DNS comes into play

Now when your PC needs AD:
It asks DNS: “Where is the Domain Controller?”
DNS responds → gives DC details

## Will DNS tell me the IP of DHCP?

Will DNS tell me the IP of DHCP?  **No — DNS is NOT used to find DHCP**
Because DHCP works differently:

- DHCP uses **broadcast**
- It doesn’t need a name or IP to start
- It shouts on the network: “Any DHCP server out there?”

So:

- ❌ No DNS lookup for DHCP
- DHCP finds clients via broadcast
### Big Picture

When a client gets an IP from DHCP:
- It can automatically create/update its DNS record  
- So others can find it by name

This is called: **Dynamic DNS (DDNS)**

## What is “Auto DNS Registration”?

It means: When a machine gets an IP → its **hostname is automatically added to DNS**
Example: `PC1 → gets IP 192.168.1.50  DNS entry created → PC1.company.local → 192.168.1.50`
Now:

- You can ping `PC1` instead of IP
- AD and other systems can locate it

## Who actually updates DNS?

There are **2 possible ways**:
### Option A: Client updates DNS (default)

- Windows client registers itself in DNS
- Uses its own credentials

✔ Works fine in simple setups  
❌ Problem: permissions & stale records in larger environments
### Option B (Recommended): DHCP updates DNS

This is what enterprises use. Flow:

1. Client gets IP from DHCP
2. DHCP server updates DNS **on behalf of client**

✔ Central control  
✔ Secure  
✔ Works for non-Windows devices too

## How to Configure DHCP + DNS Integration (Windows Server)

### Step 1: Open DHCP console

Go to: `Server Manager → Tools → DHCP`
### Step 2: Configure DNS settings in DHCP

Right-click: `IPv4 → Properties → DNS tab`
Enable: **Enable DNS dynamic updates**  
Choose: “Always dynamically update DNS records”
Check: “Discard A and PTR records when lease is deleted”
### Step 3: Set DHCP credentials (IMPORTANT)

This is what most people miss
In DHCP console: `IPv4 → Properties → Advanced → Credentials`
Add a **domain user account** (not admin, just normal user)
Why? DHCP will use this account to securely update DNS
### Step 4: Configure DNS Server

Open DNS: `Server Manager → Tools → DNS`
Right-click your zone (e.g. `company.local`)
Properties → General
Set:  **Dynamic updates = Secure only**
### What happens now (real flow)

Let’s trace it:
Step 1: Client → “I need IP”
Step 2: DHCP → gives IP + DNS info
Step 3: DHCP → updates DNS: `PC1.company.local → 192.168.1.50`
Step 4: Other systems can now find PC1 via DNS
### How to verify (very important for you)

On client: `ipconfig /all`
Check: DNS server IP, Hostname
Check DNS record: `nslookup PC1.company.local`
Force registration (client side): `ipconfig /registerdns`
Check DHCP logs:  Located in: `C:\Windows\System32\dhcp`
## Common real-world issues

These are things you’ll actually face:
❌ DNS record not updating -> DHCP credentials missing
❌ Duplicate records ->  Client + DHCP both updating Fix:  Let **only DHCP handle updates**
❌ Old/stale records ->  Not scavenging DNS ->Enable:  DNS scavenging (cleanup old entries)
## What Auto DNS Registration actually solves

It keeps this mapping always updated: `Hostname  ↔  Current IP`
Without it, that mapping becomes:

- ❌ missing
- ❌ outdated
- ❌ inconsistent
## So what if you DON’T register DNS?

Let’s break it practically.
### Case 1: You don’t care about names (only IPs)

You _can_ operate like this:

- Use IPs everywhere
- Manually track devices

Works in very small setups

❌ Problems:

- Not scalable
- Hard to manage
- Breaks AD-dependent features
### Case 2: Machine IP changes (very common)

DHCP gives dynamic IPs: `Yesterday: PC1 → 192.168.1.50  Today:     PC1 → 192.168.1.80`
If DNS is NOT auto-updated: `DNS still says → PC1 = 192.168.1.50 ❌`
Now:

- Ping fails
- Remote access fails
- Scripts fail

You get **random, confusing issues**

### Case 3: Active Directory environment

This is where it really matters.
AD uses:

- Hostnames (not IPs)
- Kerberos (name-based authentication)

If a machine isn’t properly in DNS:

- ❌ Other machines can’t find it
- ❌ Group Policy targeting may fail
- ❌ Remote tools (RDP, SCCM, etc.) break
- ❌ Reverse lookups fail (important for logs/security)

### Case 4: Admin / IT operations (your job)

Imagine you’re troubleshooting:
Instead of: `ping PC1`
You must: `find IP manually → then ping`
Multiply that by:

- 100+ systems
- changing IPs

It becomes a nightmare

## Real-world problems without auto DNS

These are actual tickets you’ll see:

- “Hostname not resolving”
- “RDP not working but IP works”
- “Application cannot connect to server”
- “Duplicate IP / wrong system responding”

## The BIG issue: stale records

Worst case: `PC1 old IP → now given to PC2DNS still says → PC1 = old IP`
Now:  You connect to **wrong machine**
That’s dangerous.

## Why auto registration exists

Because in modern networks:

- IPs are dynamic (DHCP)
- Devices are many
- Changes are frequent

Manual DNS is not practical

# 1️⃣2️⃣ KDC TGT RADIUS

- **Kerberos** is the **authentication protocol/system** that defines how users prove their identity securely on a network.
- **KDC (Key Distribution Center)** is a **component inside Active Directory** that acts like the “ticket office.” It **issues tickets** needed for authentication.
- **TGT (Ticket-Granting Ticket)** is the **first ticket** the KDC gives you when you log in successfully. It proves to the network that you are a valid user.
- After getting a TGT, the **Kerberos protocol** uses it to get **service tickets** for internal resources without asking for your password again.

**Flow in simple terms (your network):**

```
User enters AD credentials 
	↓ 
Kerberos protocol triggers
	↓   
KDC (AD server) verifies credentials 
	↓
KDC issues TGT 
	↓   
Device uses TGT to request service tickets 
	↓
Access internal resources
```

So yes, **KDC provides TGT**, and **Kerberos is the system/protocol that uses it** for authentication.

## Is there any protocol other than Kerboros For auth?

Yes! Kerberos is the default authentication protocol in **Windows/Active Directory environments**, but there are several **other authentication protocols** used in networks depending on the system, use case, or vendor. Here’s a detailed breakdown:

### **1. RADIUS (Remote Authentication Dial-In User Service)**

- **Purpose:** Centralized authentication for network devices (Wi-Fi, VPN, switches).
- Often used with **802.1X / PEAP** for Wi-Fi authentication.
- Works with AD to verify credentials.
- **Flow:** Device → Network Device (AP/Firewall) → RADIUS server → AD

### **2. LDAP / LDAPS (Lightweight Directory Access Protocol / Secure)**

- **Purpose:** Access directory services for authentication and user info.
- AD supports LDAP for applications that need to verify credentials.
- **LDAPS** adds encryption for security.

### **3. NTLM (NT LAN Manager)**

- **Purpose:** Legacy Windows authentication protocol.
- Used when Kerberos is unavailable (like across non-AD domains or certain legacy applications).
- Less secure than Kerberos because it doesn’t use tickets; instead, it relies on challenge/response hashing.

### **4. SAML (Security Assertion Markup Language)**

- **Purpose:** Web-based Single Sign-On (SSO) for cloud or internal applications.
- Often used with **Okta, Azure AD, or Google Workspace**.
- Allows users to authenticate once and access multiple web apps.

### **5. OAuth / OpenID Connect**

- **Purpose:** Modern token-based authentication for web and mobile apps.
- Common in cloud services (Google, Microsoft 365, GitLab SaaS).

### **6. EAP Variants (for Wi-Fi / VPN)**

- **PEAP (Protected EAP)** – encapsulates authentication in a TLS tunnel; works with AD + Kerberos or LDAP.
- **EAP-TLS** – certificate-based authentication without passwords.
- **EAP-TTLS** – similar to PEAP, supports various inner authentication methods.

### **Summary Table**

| Protocol               | Use Case          | Notes                               |
| ---------------------- | ----------------- | ----------------------------------- |
| Kerberos               | AD, LAN, SSO      | Ticket-based, secure, default in AD |
| NTLM                   | Legacy Windows    | Challenge-response, less secure     |
| RADIUS                 | Wi-Fi, VPN        | Centralized auth, works with AD     |
| LDAP / LDAPS           | Directory lookups | Can authenticate apps via AD        |
| SAML                   | Web apps / SSO    | Token-based, cloud-focused          |
| OAuth / OpenID Connect | Web / mobile apps | Modern token-based auth             |
| EAP variants           | Wi-Fi/VPN         | PEAP, EAP-TLS, EAP-TTLS             |


RADIUS is middle man btw AD and user for passing cred and used by aruba(which is RADIUS CLIENT or Server)  if user connected to AP.

# 1️⃣3️⃣ RAS

## What is RAS (Remote Access Service)?

RAS = Remote Access Service, used for users to dial-in or VPN into a company network.

Earlier it literally meant modem dial-in (telephone line), later extended for:  
✔ VPN (PPTP/L2TP/SSTP)  
✔ RRAS in Windows Server  
✔ Remote RADIUS authentication

### What does “Dial-In RAS permissions for the user” mean?

It means: Does this AD user have permission to connect remotely through RAS/VPN ?
In Actve Directory, each user object has a Dial-in tab that controls this.
There are typically 3 options:

1. Allow access
2. Deny access
3. Control access through NPS policies (common in domain setups)

### Where is this used?

Typically when:  
✔ setting up VPN access  
✔ RADIUS server authentication  
✔ RRAS (Routing & Remote Access Services)  
✔ NPS policies

Example scenario:  User → VPN client → RRAS → NPS/RADIUS → AD → Accept/Reject
If the user doesn’t have dial-in permission, authentication fails.

### When you see error like

Dial-In RAS permissions for the user
It usually means:  The user is not allowed for VPN/dial-in remote access in AD/NPS

### Practical Real-world Example

ISSUE: User cannot connect to VPN.
CHECK: In AD Users & Computers → User → Properties → Dial-In tab
OR in NPS: Network Policies → “Remote Access Permission: Allow”
If set to Deny, authentication fails even if password is correct.

### Why do companies still care today?

RAS permissions still matter if environment uses:

- RRAS VPN (Windows)
- NPS RADIUS
- Wi-Fi 802.1x with RADIUS auth
- Network device login via RADIUS

### Mapping to other components

User must pass: `User Credentials → RADIUS/NPS → AD Policy → RAS Dial-in Permission → Accept`
If any part `rejects → fail.`

### WHAT YOU KNOW TODAY (normal Wi-Fi)

Normal Wi-Fi at home uses:  WPA2/WPA3-PSK → One shared password
Everyone uses the same Wi-Fi password
Easy, but not secure for enterprise.

### WHAT ENTERPRISE USES

Enterprises don’t want a shared password, so they use:
802.1X + RADIUS + individual user authentication
So even Wi-Fi login becomes like logging into domain.

### Breaking the phrase

802.1x  => authentication framework for network access
RADIUS  => backend authentication server

### FLOW: HOW Wi-Fi 802.1x WORKS

3 components always involved:

1️⃣ Supplicant → Device (Laptop/Phone)  
2️⃣ Authenticator → Access Point / Switch  
3️⃣ Authentication Server → RADIUS (NPS, FreeRADIUS, ISE, etc.)

Process:

User enters AD credentials
Device → AP (802.1x)
AP → RADIUS
RADIUS → AD (verify user)
If OK → AP gives network access

### Credentials Used in Enterprises

802.1x can use:  
✔ user/password (PEAP/MSCHAPv2)  
✔ certificates (EAP-TLS) → most secure  
✔ smartcards (optional)

### WHY COMPANIES USE IT

Benefits:  
✔ No shared Wi-Fi password  
✔ Per-user authentication  
✔ Can revoke access per user  
✔ AD integrated  
✔ VLAN assignment based on user  
✔ Can enforce NAC / posture checks

### REAL WORLD SCENARIO

Company Wi-Fi:

- Corp-WiFi → 802.1x (RADIUS + AD accounts)
- Guest-WiFi → simple password or captive portal  

### SEQUENCE (DETAILED FLOW)

Supplicant → Access-Request → RADIUS
RADIUS → AD → check credentials
RADIUS → Accept/Reject
AP → allow or deny port

Packets:

EAPOL
RADIUS Access-Request
RADIUS Access-Accept

### WHERE RAS PERMISSIONS ENTER

If RAS permissions (Dial-In) are disabled for a user:
User joins corporate Wi-Fi → AUTH FAIL
Because: Wi-Fi 802.1x → RADIUS → AD → RAS permission check

### COMMON RADIUS SERVERS

Enterprise uses:

- Microsoft NPS (most common in AD shops)
- Cisco ISE
- Aruba ClearPass
- FreeRADIUS (Linux)
- FortiNAC
- Palo Alto GlobalProtect (in some modes)

### In Human language:

“Wi-Fi 802.1x with RADIUS lets users log into Wi-Fi using their domain account instead of a shared password.”

## id output breakdown

uid=1988112744(kodidala.kumar) gid=1988112744(kodidala.kumar) groups=1988112744(kodidala.kumar), 27(sudo), 1988100609(domain_users), 1988100649(ras_and_ias_servers), 1988103132(remotewifi)

### Domain Relevant Groups

#### 1. domain_users

Default AD group for all domain accounts.
Meaning = you’re a normal AD user.

#### 2. ras_and_ias_servers

This is interesting.

RAS = Remote Access Service  
IAS = Internet Authentication Service (old name for NPS before Server 2008)

If you are in this group, it typically means:

✔ Allowed for RADIUS-based services  
✔ Allowed for remote access authentication  
✔ Allowed for RRAS/NPS restrictions

This is exactly linked with: Dial-In remote access permissions and Remote Wi-Fi 802.1x

#### 3. remotewifi

This one gives it away.
This group is most likely: ✔ an AD security group used by NPS policy to allow Wi-Fi 802.1x access
Common scenario in companies:
If user ∈ remotewifi group → allow Wi-Fi 802.1x
Else → reject
So the NPS rule might look like:
Condition: Windows-Group = remotewifi
Action: Access-accept

### Putting it in a practical chain

Your login path likely is:

Device → AP (802.1x EAPOL)
AP → RADIUS/NPS
NPS → AD

Check:

   ✔ Credentials valid?
   ✔ RAS permission allowed?
   ✔ Member of remotewifi?

If all true → Access-Accept

### sudo group

Unrelated to RADIUS/Wi-Fi.
Means locally on this Linux system you have admin privilege.

### Why this matters in Infra / Networking

This is how enterprises:

✔ control Wi-Fi  
✔ control VPN  
✔ control remote access  
✔ integrate with AD

without sharing passwords.

### If you removed yourself from remotewifi

You would fail Wi-Fi login.
If you removed yourself from ras_and_ias_servers
You might fail RADIUS-based authentication.
Typical Company Policy Structure
AD Groups map to:


| AD Group             | Purpose                |
| -------------------- | ---------------------- |
| remotewifi           | Access corporate Wi-Fi |
| vpn_users            | Allow VPN access       |
| radius_users         | 802.1x authentication  |
| remote_desktop_users | RDP permissions        |
| domain_users         | default                |

## INFRA COMPONENTS

Your target stack contains:

1. AD (Active Directory)
2. NPS (Network Policy Server)
3. RADIUS
4. AP (Access Point)

Now let’s break them down.

### (1) ACTIVE DIRECTORY (AD)

##### ROLE / WHAT IT DOES

✔ Stores user accounts  
✔ Stores passwords  
✔ Stores groups & permissions  
✔ Verifies credentials (auth)  
✔ Provides centralized policy

##### USED FOR

- Domain login
- VPN login
- Wi-Fi 802.1x login
- Authorization checks

##### WHY NEEDED

Without AD → no identity control. Otherwise you'd need local accounts on every system (unmanageable).

##### REQUIRES

- Domain Controllers (DC)
- DNS working properly
- Time sync

##### OUTPUT

Auth-Result (valid/invalid user) + group membership

### (2) NPS (Network Policy Server)

NPS = Microsoft’s RADIUS server role.

##### ROLE / WHAT IT DOES

✔ Receives authentication requests from AP/VPN  
✔ Queries AD to verify user  
✔ Applies Network Access Policies  
✔ Decides ALLOW or DENY  
✔ Sends RADIUS Access-Accept / Access-Reject

##### NPS IS WHERE YOU CONFIGURE

- Wi-Fi 802.1X policies
- VPN policies
- RAS permissions
- Group policies
- VLAN assignment
- Certificate based auth (EAP-TLS)

##### WHY NEEDED

APs and VPN devices cannot talk to AD directly.
So NPS acts as translator: AP → RADIUS Request → NPS → AD

##### REQUIRES

- AD membership (NPS must join domain)
- Certificate (if using PEAP / TLS)
- RADIUS Shared Secret with AP

### (3) RADIUS (Protocol)

RADIUS itself is not a server, it’s a protocol.

##### ROLE / WHAT IT DOES

- Transports credentials to NPS
- Carries Access-Request / Accept / Reject
- Carries accounting logs
- Supports EAP (Extensible Auth Protocol)

##### 3 Main Functions

✔ Authentication – Are you legit?  
✔ Authorization – What are you allowed to access?  
✔ Accounting – What did you do? (logs)

##### WHY NEEDED

Because Wi-Fi/VPN devices don’t understand AD.
They only understand RADIUS.

##### USED PORTS

UDP 1812 → Authentication
UDP 1813 → Accounting

### (4) AP (Access Point)

In 802.1x environment, AP = Authenticator

##### ROLE / WHAT IT DOES

✔ Sends user credentials to RADIUS (NPS)  
✔ Blocks traffic until authentication is successful  
✔ Enforces allow/deny  
✔ Optionally applies dynamic VLAN  
✔ Optionally logs accounting

##### WHY NEEDED

Because without 802.1x AP, any device could connect to Wi-Fi freely.

##### REQUIRES

- SSID configured as WPA2/WPA3-Enterprise
- RADIUS IP + Shared Secret
- EAP support (PEAP or TLS)    
- Connectivity to NPS

### HOW THEY ALL TALK (AUTH FLOW)

This is the real wiring:

```
[Client Device]
     |
     | EAPOL (802.1x)
     ↓
[AP / Controller]
     |
     | RADIUS Access-Request
     ↓
[NPS Server]
     |
     | LDAP/Kerberos
     ↓
[Active Directory]
```

Then return path:  `AD → Success/Fail → NPS → Access-Accept/Reject → AP → Allow/Deny Network`

### WHO DOES WHAT (RESPONSIBILITY TABLE)

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|Component|Auth|Authorization|Directory|Policy|Enforcement|
|Client|Sends creds|none|none|none|none|
|AP|forward EAP|applies VLAN|none|none|Yes|
|RADIUS (NPS)|Yes|Yes|talks to AD|Yes|No|
|AD|verifies creds|sends groups|Yes|partly|No|

So:

✔ AD = identity  
✔ NPS = decision brain  
✔ AP = enforcer  
✔ RADIUS = transport

### WHAT EACH NEEDS TO WORK

#### AP Needs

- IP of NPS
- Shared secret
- Ports 1812/1813 open
- Enterprise SSID
- EAP Support (PEAP/TLS)

#### NPS Needs

- Domain joined server
- Certificate (for PEAP or TLS)
- Network Policy
- CRL reachable (if certs)
- Time sync with DC

### AD Needs

- Users and Groups
- Password policy
- Dial-in / RAS permissions (optional)

#### Client Needs

- CA certificate trusted
- EAP method set (PEAP or TLS)
- Domain credentials or certificate

### WHY COMPANIES USE THIS STACK

Because it solves:

✔ Identity-based network access  
✔ No shared Wi-Fi passwords  
✔ Automatic removal on termination  
✔ Group-based segmentation  
✔ Logs for compliance  
✔ Zero trust foundation

### WITHOUT NPS/AD/RADIUS

Wi-Fi = not secure  
VPN = not manageable  
No logs  
No revocation  
No identity control  
Everyone uses same key = bad