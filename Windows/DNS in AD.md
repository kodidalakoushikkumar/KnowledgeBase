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