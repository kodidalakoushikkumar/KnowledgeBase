**

# What is RAS (Remote Access Service)?

RAS = Remote Access Service, used for users to dial-in or VPN into a company network.

Earlier it literally meant modem dial-in (telephone line), later extended for:  
✔ VPN (PPTP/L2TP/SSTP)  
✔ RRAS in Windows Server  
✔ Remote RADIUS authentication

# What does “Dial-In RAS permissions for the user” mean?

It means: Does this AD user have permission to connect remotely through RAS/VPN ?

In Actve Directory, each user object has a Dial-in tab that controls this.

There are typically 3 options:

1. Allow access
    
2. Deny access
    
3. Control access through NPS policies (common in domain setups)
    

# Where is this used?

Typically when:  
✔ setting up VPN access  
✔ RADIUS server authentication  
✔ RRAS (Routing & Remote Access Services)  
✔ NPS policies

Example scenario:  User → VPN client → RRAS → NPS/RADIUS → AD → Accept/Reject

If the user doesn’t have dial-in permission, authentication fails.

# When you see error like

Dial-In RAS permissions for the user

It usually means:  
→ The user is not allowed for VPN/dial-in remote access in AD/NPS

# Practical Real-world Example

ISSUE: User cannot connect to VPN.

CHECK: In AD Users & Computers → User → Properties → Dial-In tab

OR in NPS: Network Policies → “Remote Access Permission: Allow”

If set to Deny, authentication fails even if password is correct.

# Why do companies still care today?

RAS permissions still matter if environment uses:

- RRAS VPN (Windows)
    
- NPS RADIUS
    
- Wi-Fi 802.1x with RADIUS auth
    
- Network device login via RADIUS
    

# Mapping to other components

User must pass: User Credentials → RADIUS/NPS → AD Policy → RAS Dial-in Permission → Accept

If any part rejects → fail.

# WHAT YOU KNOW TODAY (normal Wi-Fi)

Normal Wi-Fi at home uses:  WPA2/WPA3-PSK → One shared password

Everyone uses the same Wi-Fi password

Easy, but not secure for enterprise.

# WHAT ENTERPRISE USES

Enterprises don’t want a shared password, so they use:

802.1X + RADIUS + individual user authentication

So even Wi-Fi login becomes like logging into domain.

# Breaking the phrase

802.1x  => authentication framework for network access

RADIUS  => backend authentication server

# FLOW: HOW Wi-Fi 802.1x WORKS

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

# Credentials Used in Enterprises

802.1x can use:  
✔ user/password (PEAP/MSCHAPv2)  
✔ certificates (EAP-TLS) → most secure  
✔ smartcards (optional)

  

# WHY COMPANIES USE IT

Benefits:  
✔ No shared Wi-Fi password  
✔ Per-user authentication  
✔ Can revoke access per user  
✔ AD integrated  
✔ VLAN assignment based on user  
✔ Can enforce NAC / posture checks

# REAL WORLD SCENARIO

Company Wi-Fi:

- Corp-WiFi → 802.1x (RADIUS + AD accounts)
    
- Guest-WiFi → simple password or captive portal  
      
    

# SEQUENCE (DETAILED FLOW)

Supplicant → Access-Request → RADIUS

RADIUS → AD → check credentials

RADIUS → Accept/Reject

AP → allow or deny port

Packets:

EAPOL

RADIUS Access-Request

RADIUS Access-Accept

  

# WHERE RAS PERMISSIONS ENTER

If RAS permissions (Dial-In) are disabled for a user:

User joins corporate Wi-Fi → AUTH FAIL

Because:

Wi-Fi 802.1x → RADIUS → AD → RAS permission check

# COMMON RADIUS SERVERS

Enterprise uses:

- Microsoft NPS (most common in AD shops)
    
- Cisco ISE
    
- Aruba ClearPass
    
- FreeRADIUS (Linux)
    
- FortiNAC
    
- Palo Alto GlobalProtect (in some modes)
    

# SWITCH PORT VERSION (same tech!)

802.1x is also used on switches:

Don’t allow Ethernet port traffic until user/device authenticates

Useful for secure buildings.

# In Human language:

“Wi-Fi 802.1x with RADIUS lets users log into Wi-Fi using their domain account instead of a shared password.”

# id output breakdown

uid=1988112744(kodidala.kumar)

gid=1988112744(kodidala.kumar)

groups=1988112744(kodidala.kumar),

27(sudo),

1988100609(domain_users),

1988100649(ras_and_ias_servers),

1988103132(remotewifi)

  

Now let’s understand each one:

# Domain Relevant Groups

### 1. domain_users

Default AD group for all domain accounts.

Meaning = you’re a normal AD user.

### 2. ras_and_ias_servers

This is interesting.

RAS = Remote Access Service  
IAS = Internet Authentication Service (old name for NPS before Server 2008)

If you are in this group, it typically means:

✔ Allowed for RADIUS-based services  
✔ Allowed for remote access authentication  
✔ Allowed for RRAS/NPS restrictions

This is exactly linked with:

Dial-In remote access permissions

and

Remote Wi-Fi 802.1x

### 3. remotewifi

This one gives it away.

This group is most likely:

✔ an AD security group used by NPS policy to allow Wi-Fi 802.1x access

Common scenario in companies:

If user ∈ remotewifi group → allow Wi-Fi 802.1x

Else → reject

  

So the NPS rule might look like:

Condition:

  Windows-Group = remotewifi

Action:

  Access-accept

# Putting it in a practical chain

Your login path likely is:

Device → AP (802.1x EAPOL)

AP → RADIUS/NPS

NPS → AD

Check:

   ✔ Credentials valid?

   ✔ RAS permission allowed?

   ✔ Member of remotewifi?

If all true → Access-Accept

  

# sudo group

Unrelated to RADIUS/Wi-Fi.

Means locally on this Linux system you have admin privilege.

# Why this matters in Infra / Networking

This is how enterprises:

✔ control Wi-Fi  
✔ control VPN  
✔ control remote access  
✔ integrate with AD

without sharing passwords.

# If you removed yourself from remotewifi

You would fail Wi-Fi login.

If you removed yourself from ras_and_ias_servers

You might fail RADIUS-based authentication.

Typical Company Policy Structure

AD Groups map to:

|   |   |
|---|---|
|AD Group|Purpose|
|remotewifi|Access corporate Wi-Fi|
|vpn_users|Allow VPN access|
|radius_users|802.1x authentication|
|remote_desktop_users|RDP permissions|
|domain_users|default|

  
  
  
  
  
  

# INFRA COMPONENTS

Your target stack contains:

1. AD (Active Directory)
    
2. NPS (Network Policy Server)
    
3. RADIUS
    
4. AP (Access Point)
    

Now let’s break them down.

# (1) ACTIVE DIRECTORY (AD)

### ROLE / WHAT IT DOES

✔ Stores user accounts  
✔ Stores passwords  
✔ Stores groups & permissions  
✔ Verifies credentials (auth)  
✔ Provides centralized policy

### USED FOR

- Domain login
    
- VPN login
    
- Wi-Fi 802.1x login
    
- Authorization checks
    

### WHY NEEDED

Without AD → no identity control. Otherwise you'd need local accounts on every system (unmanageable).

### REQUIRES

- Domain Controllers (DC)
    
- DNS working properly
    
- Time sync
    

### OUTPUT

Auth-Result (valid/invalid user) + group membership

# (2) NPS (Network Policy Server)

NPS = Microsoft’s RADIUS server role.

### ROLE / WHAT IT DOES

✔ Receives authentication requests from AP/VPN  
✔ Queries AD to verify user  
✔ Applies Network Access Policies  
✔ Decides ALLOW or DENY  
✔ Sends RADIUS Access-Accept / Access-Reject

### NPS IS WHERE YOU CONFIGURE

- Wi-Fi 802.1X policies
    
- VPN policies
    
- RAS permissions
    
- Group policies
    
- VLAN assignment
    
- Certificate based auth (EAP-TLS)
    

### WHY NEEDED

APs and VPN devices cannot talk to AD directly.

So NPS acts as translator: AP → RADIUS Request → NPS → AD

### REQUIRES

- AD membership (NPS must join domain)
    
- Certificate (if using PEAP / TLS)
    
- RADIUS Shared Secret with AP
    

# (3) RADIUS (Protocol)

RADIUS itself is not a server, it’s a protocol.

### ROLE / WHAT IT DOES

- Transports credentials to NPS
    
- Carries Access-Request / Accept / Reject
    
- Carries accounting logs
    
- Supports EAP (Extensible Auth Protocol)
    

### 3 Main Functions

✔ Authentication – Are you legit?  
✔ Authorization – What are you allowed to access?  
✔ Accounting – What did you do? (logs)

### WHY NEEDED

Because Wi-Fi/VPN devices don’t understand AD.

They only understand RADIUS.

### USED PORTS

UDP 1812 → Authentication

UDP 1813 → Accounting

# (4) AP (Access Point)

In 802.1x environment, AP = Authenticator

### ROLE / WHAT IT DOES

✔ Sends user credentials to RADIUS (NPS)  
✔ Blocks traffic until authentication is successful  
✔ Enforces allow/deny  
✔ Optionally applies dynamic VLAN  
✔ Optionally logs accounting

### WHY NEEDED

Because without 802.1x AP, any device could connect to Wi-Fi freely.

### REQUIRES

- SSID configured as WPA2/WPA3-Enterprise
    
- RADIUS IP + Shared Secret
    
- EAP support (PEAP or TLS)
    
- Connectivity to NPS
    

# HOW THEY ALL TALK (AUTH FLOW)

This is the real wiring:

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

Then return path:

AD → Success/Fail → NPS → Access-Accept/Reject → AP → Allow/Deny Network

# WHO DOES WHAT (RESPONSIBILITY TABLE)

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

# WHAT EACH NEEDS TO WORK

### AP Needs

- IP of NPS
    
- Shared secret
    
- Ports 1812/1813 open
    
- Enterprise SSID
    
- EAP Support (PEAP/TLS)
    

### NPS Needs

- Domain joined server
    
- Certificate (for PEAP or TLS)
    
- Network Policy
    
- CRL reachable (if certs)
    
- Time sync with DC
    

### AD Needs

- Users and Groups
    
- Password policy
    
- Dial-in / RAS permissions (optional)
    

### Client Needs

- CA certificate trusted
    
- EAP method set (PEAP or TLS)
    
- Domain credentials or certificate
    

# WHY COMPANIES USE THIS STACK

Because it solves:

✔ Identity-based network access  
✔ No shared Wi-Fi passwords  
✔ Automatic removal on termination  
✔ Group-based segmentation  
✔ Logs for compliance  
✔ Zero trust foundation

# WITHOUT NPS/AD/RADIUS

Wi-Fi = not secure  
VPN = not manageable  
No logs  
No revocation  
No identity control  
Everyone uses same key = bad

**