---
Title:
Type:
Topic:
tags:
ee: "[[/IMAGES/BUS.png]]"
---
# 1. First forget the old telephone idea

A traditional phone system might look like:

```
Telephone
    │
    │ Copper telephone cable
    │
Telephone Exchange / PBX
    │
    │ PSTN
    │
Other Telephone
```

With VoIP, your office phone is connected to the **IP network**:

```
                 ┌─────────────┐
                 │   Internet  │
                 └──────┬──────┘
                        │
                  ISP / SIP Trunk
                        │
                ┌───────▼────────┐
                │ PBX / IP-PBX    │
                │ Phone Server    │
                └───┬────┬────┬───┘
                    │    │    │
                 Network Network Network
                    │    │    │
                  Phone Phone Phone
                  1001  1002  1003
```

Your Beetel phone is **not directly calling another Beetel phone** in most office setups.
Instead:

```
Phone 1001
    │
    ▼
	PBX
    │
    ▼
Phone 1002
```

The PBX is the **brain / telephone exchange of the company**.
# 2. The three things you need to understand

For almost every office VoIP system, think about these three components:

|Component|Example|Job|
|---|---|---|
|IP Phone|Beetel phone|User's physical telephone|
|PBX / IP-PBX|Asterisk, FreePBX, 3CX, Yeastar, Cisco CUCM, etc.|Controls calls|
|SIP Trunk / Telecom Provider|Airtel, Jio, Tata, BSNL, etc.|Connects office to external telephone network|

So your architecture probably looks something like:

```
                        OUTSIDE WORLD
      Mobile  ────────┐
                      │
      Landline ───────┼──── PSTN / Telecom Network
                      │
      Other Company ──┘
                            │
                            │ SIP Trunk
                            ▼
                    ┌─────────────────┐
                    │    IP-PBX       │
                    │                 │
                    │ Extensions      │
                    │ 1001            │
                    │ 1002            │
                    │ 1003            │
                    │ 1004            │
                    └────────┬────────┘
                             │
                       Office Network
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
          Beetel          Beetel         Beetel
          Phone           Phone          Phone
          Ext 1001        Ext 1002       Ext 1003
```

# 3. What is an Extension?

This is probably the first thing you should check in your company. Suppose there are 50 employees:

```
Reception       → 1000
HR              → 1001
Koushik         → 1002
Manager         → 1003
IT              → 1004
Conference Room → 1005
```

These numbers are usually **extensions**. They are not necessarily public telephone numbers.
When you dial: 1003 ., Your phone is basically saying to the PBX:  "I am extension 1002. Please connect me to extension 1003."
The PBX checks:

```
Is extension 1002 registered? Yes
Is extension 1003 registered? Yes
Is 1003 allowed to receive calls? Yes
```

Then it connects both phones.

# 4. The most important process: Phone Registration

When your Beetel IP phone starts, something like this happens.

## Step 1: Phone gets power

Depending on the setup:
### Option A: PoE

```
Network Switch
       │
       │ Ethernet + Power
       ▼
    IP Phone
```

The switch provides power through the Ethernet cable.
This is called: **Power over Ethernet (PoE)**.
Typical office setup:

```
PoE Switch
   │
   ├── PC
   ├── IP Phone
   ├── IP Phone
   └── IP Phone
```
### Option B: Power Adapter

```
Electricity
     │
     ▼
Power Adapter → IP Phone
                   │
                   ▼
                 LAN
```

# 5. The phone gets an IP address

Your Beetel phone is basically another network device.
For example:

```
Phone Extension: 1002
IP Address:      192.168.10.51
Gateway:         192.168.10.1
Subnet Mask:     255.255.255.0
DNS:             192.168.10.10
PBX Server:      192.168.10.20
```

Usually DHCP gives the phone this information automatically. So:

```
IP Phone
    │
    │ DHCP Discover
    ▼
DHCP Server
    │
    │ "Here is your IP address"
    ▼
IP Phone
```

For example: `DHCP Server → 192.168.10.51` , Now the phone is reachable on the network.
# 6. How does the phone know where the PBX is?

This is where configuration comes in. The phone needs to know something like:
SIP Server / Registrar: `192.168.10.20` or: `pbx.company.local`
It also needs credentials.
For example:

```
Extension:        1002
Authentication ID: 1002
Password:         ********
SIP Server:       192.168.10.20
```

Then the phone sends a SIP registration request.
Conceptually:

```
Phone 1002
IP: 192.168.10.51

        REGISTER
            │
            ▼
       PBX Server
       192.168.10.20
```

The PBX checks:

```
Extension: 1002
Password:  Correct
```

Then:

```
PBX
 │
 └── 200 OK
```

Now the phone is registered.

You may see something on the phone like: Registered or the extension number appears on the screen.
The PBX now knows: Extension 1002 is currently online at IP address 192.168.10.51.
This is extremely important.
Think of the PBX as maintaining a table like this:

|Extension|Status|Current IP|
|---|---|---|
|1001|Registered|192.168.10.50|
|1002|Registered|192.168.10.51|
|1003|Registered|192.168.10.52|
|1004|Offline|—|

So when someone calls 1002, the PBX knows where to send the call.

# 7. What is SIP?

SIP means: **Session Initiation Protocol**
Don't get scared by the name. For practical IT work, think:  **SIP is mainly responsible for setting up and controlling the call.**
Example:
You call extension 1003. Your phone sends: "Please call 1003"
The PBX processes that request. This communication happens using SIP.
Simplified:

```
Phone 1002
     │
     │ SIP INVITE
     ▼
     PBX
     │
     │ SIP INVITE
     ▼
Phone 1003
```

Phone 1003 starts ringing. When the person answers: `1003 → 200 OK`
The PBX completes the call setup.

# 8. But SIP is NOT usually carrying your voice

This is another important concept. There are generally two major parts of a VoIP call:

## 1. Signaling

Used for:

- Registering phones
- Starting calls
- Ringing
- Answering
- Hanging up
- Hold
- Transfer

Usually: SIP
## 2. Media

Used for: Your actual voice
Usually: RTP
So the architecture can be:

```
             SIP Signaling
Phone 1001 ───────► PBX ───────► Phone 1002
  
  
              Voice / RTP
Phone 1001 ◄──────────────────► Phone 1002
```

Sometimes RTP goes directly between phones. Sometimes it goes through the PBX. That depends on the PBX and network configuration.

# 9. Internal call architecture

Let's take a real example.

You are extension: 1002
Your colleague is: 1005
You pick up the phone and dial: 1005
The process is approximately:
### Step 1

Your phone sends: INVITE 1005 to the PBX.

```
Beetel Phone 1002
        │
        │ SIP INVITE
        ▼
      PBX
```

### Step 2

The PBX checks: Does extension 1005 exist? Yes.
Is extension 1005 registered? Yes.
Then:

```
PBX
 │
 │ INVITE
 ▼
Phone 1005
```

### Step 3

Phone 1005 rings.

```
☎ Incoming Call
From: 1002
```

### Step 4

The user answers.

Now RTP voice communication starts.

```
       SIP CONTROL
1002 ───── PBX ───── 1005
  
  
       RTP VOICE
1002 ◄──────────────► 1005
```

Or:

```
1002
 │
 │ RTP
 ▼
PBX
 │
 │ RTP
 ▼
1005
```

Again, the exact path depends on the configuration.

# 10. Now the interesting part: How can you call a mobile number?

Let's say extension 1002 wants to call: 9876543210
Your phone cannot magically connect directly to the mobile network.
The call goes through the PBX.
Your Beetel Phone

```
Extension 1002
       │
       ▼
       PBX
       │
       │ "This is an external number"
       ▼
SIP Trunk
       │
       ▼
Telecom Provider
       │
       ▼
PSTN / Mobile Network
       │
       ▼
Mobile Phone
```

The PBX decides:  "1002 dialed an external number, so I need to send this call through my telecom provider."
This is controlled using something called a **dial plan** or **outbound route**.
# 11. What is a Dial Plan?

A dial plan tells the PBX:  Depending on the number the user dials, where should the call go?
For example: 1000 - 1999
means: Internal Extension
So: 1001, 1002, 1003
→ Send internally.
But: 0XXXXXXXXXX
might mean: External call
The PBX might remove the first `0` and send the number to the SIP provider.

Example:

You dial: 09876543210
PBX logic: User dialed 0XXXXXXXXXX
Then: Remove 0
Result: 9876543210
Then:

```
PBX
 ↓
SIP Trunk
 ↓
Telecom Provider
```

This is why sometimes you need to dial a prefix like: 0 or: 9 before making an external call.
For example: `9 + Mobile Number`
The `9` is not part of the actual mobile number.
It's basically telling the PBX: "Bro, I want an outside line."

# 12. Complete outgoing call architecture

Let's visualize it properly.

```
                 OFFICE
  
┌──────────────────────────────┐
│                              │
│  Employee                    │
│      │                       │
│      ▼                       │
│  Beetel IP Phone             │
│  Extension: 1002             │
│  IP: 192.168.10.51           │
│      │                       │
│      │ Ethernet              │
│      ▼                       │
│  ┌───────────────┐           │
│  │ Network Switch│           │
│  └───────┬───────┘           │
│          │                   │
│          ▼                   │
│     ┌───────────┐            │
│     │   IP-PBX  │            │
│     │           │            │
│     │ Ext 1001  │            │
│     │ Ext 1002  │            │
│     │ Ext 1003  │            │
│     └─────┬─────┘            │
└───────────┼──────────────────┘
            │
            │ SIP Trunk
            ▼
      Telecom Provider
            │
            ▼
      PSTN / Mobile Network
            │
            ▼
       9876543210
```

# 13. How does an incoming external call work?

Suppose someone outside calls your company number:

040-12345678

The flow might be:

```
Customer
    │
    │ Calls
    ▼
040-12345678
    │
    ▼
Telecom Provider
    │
    │ SIP Trunk
    ▼
Company PBX
    │
    │ Incoming Call Rule
    ▼
Reception / IVR / Ring Group
```

Then depending on configuration:

```
"Press 1 for Sales"
"Press 2 for HR"
"Press 3 for Support"
Or directly:
Incoming Number
       │
       ▼
Reception Phone
Extension 1000
```

# 14. What is an IVR?

You've definitely heard this:  "Welcome to XYZ company. Press 1 for Sales. Press 2 for Support."
That system is called: **IVR — Interactive Voice Response**
Architecture:

```
External Caller
       │
       ▼
Telecom Provider
       │
       ▼
SIP Trunk
       │
       ▼
	 PBX
       │
       ▼
      IVR
       │
       ├── Press 1 → Sales
       │
       ├── Press 2 → HR
       │
       └── Press 3 → Support
```

For example:

```
Press 1
   │
   ▼
Ring Group: Sales
   │
   ├── Extension 1010
   ├── Extension 1011
   └── Extension 1012
```

All phones might ring simultaneously.

Or:

```
1010 rings for 15 seconds
        │
        ▼
1011 rings
        │
        ▼
1012 rings
```

This is called a **ring strategy**.

# 15. Ring Groups

Suppose your IT team has:

```
Koushik  → 1101
Person B → 1102
Person C → 1103
```

Instead of remembering everyone's extension, the PBX can create: IT Support → 5000
When someone calls: 5000
The PBX can do:

```
             ┌── Phone 1101
             │
Call 5000 ───┼── Phone 1102
             │
             └── Phone 1103
```

All ring at once. Or sequentially:

```
1101
 ↓ no answer
1102
 ↓ no answer
1103
```

# 16. What is Call Transfer?

Let's say someone calls Reception.

```
External Caller
      │
      ▼
Reception
Extension 1000
```

The caller wants IT.
Reception presses: Transfer → 1101
The PBX changes the call path.
There are usually two common types.

## Blind Transfer

Reception transfers directly:

```
Caller
   │
   ▼
Reception
   │
   │ Transfer
   ▼
IT 1101
```

Reception disconnects immediately.

## Attended Transfer

Reception first talks to IT:

```
Caller
   │
   ▼
Reception
   │
   │ Hold caller
   ▼
IT 1101
```

"Bro, someone from XYZ company is calling."
Then: Transfer
The caller is connected to IT.

# 17. What actually happens when the phone boots?

This is where you, as an IT guy, should start visualizing the process. A typical boot process:

```
POWER ON
   │
   ▼
Ethernet Link Up
   │
   ▼
Get IP from DHCP
   │
   ▼
Find VLAN
   │
   ▼
Find Provisioning Server / PBX
   │
   ▼
Download Configuration
   │
   ▼
Configure SIP Account
   │
   ▼
REGISTER with PBX
   │
   ▼
READY
```

Let's go deeper.

# 18. VLAN — Very important in corporate VoIP

Your company may separate voice traffic from normal computers.
For example:

```
Data VLAN
VLAN 10
192.168.10.0/24
```

and:

```
Voice VLAN
VLAN 20
192.168.20.0/24
```

Architecture:

```
                    Switch
                      │
              ┌───────┴────────┐
              │                │
         VLAN 10           VLAN 20
           DATA             VOICE
              │                │
              │                │
             PC             IP Phone
```

Why? Because voice is sensitive.
A normal website loading slowly: No big problem.
But voice: Hello Koushik... can you h.... me?
That's a problem.
So companies often separate: PC Traffic from: Voice Traffic
Example:

```
                    Core Switch
                         │
          ┌──────────────┴──────────────┐
          │                             │
      VLAN 10                        VLAN 20
      DATA                           VOICE
          │                             │
          ▼                             ▼
      Computers                     IP Phones
```

# 19. How can one Ethernet cable support both Phone and PC?

Look at the back of many IP phones. You may see two ports.
Something like: LAN / Network Port PC Port
Architecture:

```
Wall Network Port
        │
        ▼
     IP Phone
     ┌─────────────┐
     │             │
     │  Network    │
     │     │       │
     │     ▼       │
     │    Phone    │
     │             │
     │ PC Port ────┼────► Computer
     └─────────────┘
So instead of:
Wall ──► Phone
  
Wall ──► PC
You can have:
Wall
  │
  ▼
Phone
  │
  ▼
PC
```

The IP phone basically has a small Ethernet switch built into it.

# 20. How can the phone automatically get the correct VLAN?

There are several ways. Common methods include:

- Switch configuration
- LLDP
- DHCP
- Manual VLAN configuration

A common scenario:

```
Phone connects to switch
        │
        ▼
Switch tells phone: "Voice VLAN = 20"
Then: Phone traffic → VLAN 20
```

The phone may then get:

```
IP: 192.168.20.50
Gateway: 192.168.20.1
```

While the PC connected through the phone might get:

```
IP: 192.168.10.100
Gateway: 192.168.10.1
```

So physically:

```
             ONE CABLE
                 │
                 ▼
              IP PHONE
             /        \
            /          \
       VLAN 20        VLAN 10
        Voice           Data
          │               │
        Phone             PC
```

This is pretty common in enterprise environments.

# 21. Provisioning — how phones may be configured automatically

Now imagine your company has 100 phones. You don't want to manually configure:

```
Extension
Password
PBX IP
Codec
Timezone
Ring tone
Firmware
on every phone.
```

Instead, companies use **provisioning**. The architecture:

```
New Phone
    │
    ▼
DHCP / Provisioning URL
    │
    ▼
Provisioning Server
    │
    ▼
Download Config
    │
    ▼
Phone Configured
```

A configuration might conceptually contain:

```
Extension = 1002
Username = 1002
Password = xxxxxx
SIP Server = 192.168.20.10
```

So imagine you replace an employee's damaged phone. You plug in a new compatible phone.
The provisioning system identifies it by MAC address: AA:BB:CC:DD:EE:FF
The server says: This phone belongs to extension 1002.
The phone downloads its configuration and becomes: Extension 1002 without you manually entering everything.

# 22. MAC-based provisioning

This is an important thing to check in your company.
Every phone has a MAC address: 00:1A:2B:3C:4D:5E
The PBX/provisioning server may have a mapping:

```
MAC Address              Extension
------------------------------------------------
00:11:22:33:44:55         1001
00:11:22:33:44:56         1002
00:11:22:33:44:57         1003
```

Then when a phone boots: Phone MAC: 00:11:22:33:44:56
Provisioning server: Ah, this is extension 1002.
It sends the configuration.
This is why sometimes **swapping phones physically can cause confusion**.

Example:

You take Reception's phone and put it on your desk.
The phone may still show: Reception Extension 1000
because the phone's identity/configuration belongs to the device.

# 23. What is the PBX exactly?

PBX means: **Private Branch Exchange**
Think of it as: A private telephone exchange for your organization.
Old-style PBX:

```
Phones
   │
   │ Physical telephone cables
   ▼
PBX Hardware
   │
   ▼
Telephone Provider
```

Modern IP-PBX:

```
IP Phones
   │
   │ Ethernet
   ▼
IP-PBX Server
   │
   ▼
SIP Trunk
   │
   ▼
Telecom Provider
```

The PBX may be:
### A physical appliance

```
┌─────────────────┐
│ Yeastar PBX     │
│ Cisco System    │
│ Other Appliance │
└─────────────────┘
```
### A virtual machine

```
VMware / Hyper-V
       │
       ▼
PBX Virtual Machine
```
### A Linux server

```
Ubuntu Server
     │
     ▼
Asterisk / FreePBX
```
### Cloud PBX

```
Office Phones
      │
      │ Internet
      ▼
Cloud PBX
      │
      ▼
SIP Provider
```

# 24. Common PBX software/platforms you may encounter

You may discover that your Beetel phones connect to something completely different.
For example:

- Asterisk
- FreePBX
- 3CX
- Yeastar
- Cisco Unified Communications Manager
- Avaya IP Office
- Grandstream UCM
- Elastix
- Issabel
- Other telecom/vendor-specific PBX

So **Beetel phone ≠ Beetel PBX necessarily**.
Your Beetel phone may simply be a SIP-compatible endpoint.
For example:

```
Beetel Phone
      │ SIP
      ▼
Asterisk PBX
or:
Beetel Phone
      │ SIP
      ▼
3CX
or:
Beetel Phone
      │ SIP
      ▼
Cloud PBX
```

This is why finding **where your phone is registered** is the first investigation you should do.

# 25. The complete internal architecture

Let's build a realistic company setup.

```
                         ┌──────────────┐
                         │   INTERNET   │
                         └──────┬───────┘
                                │
                         Firewall/Router
                                │
                   ┌────────────┴────────────┐
                   │                         │
                   │                         │
               DATA VLAN                 VOICE VLAN
               VLAN 10                   VLAN 20
                   │                         │
                   └──────────┐  ┌───────────┘
                              │  │
                           Core Switch
                              │
                 ┌────────────┼────────────┐
                 │            │            │
              Access       Access       Access
              Switch       Switch       Switch
                 │            │            │
              Phone        Phone        Phone
              1001         1002         1003
  
  
                         PBX Server
                       192.168.20.10
                              │
                       SIP Registration
```

And the telecom side:

```
                       PBX
                        │
                        │ SIP
                        ▼
                   SIP Trunk
                        │
                        ▼
                  Telecom Provider
                        │
                  ┌─────┴─────┐
                  │           │
                PSTN       Mobile
```
# 26. Example: Let's trace an internal call packet by packet conceptually

You are: 
Extension 1001
IP: 192.168.20.51

Manager:
Extension 1002
IP: 192.168.20.52

PBX:
192.168.20.10
You dial: 1002

### Phase 1: SIP Invite

```
1001
 │
 │ SIP INVITE
 │ "I want to call 1002"
 ▼
PBX
PBX checks:
1002 → Registered
IP → 192.168.20.52
Then:
PBX
 │
 │ SIP INVITE
 ▼
1002
```

### Phase 2: Ringing

Phone 1002:
180 Ringing
PBX forwards that information.
Your phone starts playing:
Ring... Ring...

### Phase 3: Answer

Manager answers.
1002 → 200 OK

### Phase 4: Voice

Now RTP starts.
Possible:

```
1001 ◄──────── RTP ────────► 1002
```

### Phase 5: Hang up

You press the red button.
BYE
PBX terminates the session.
Call ends.

# 31. A phone configuration example

This is just an example:
Account Name: Koushik
SIP User ID: 1101
Authentication ID: 1101
Password: `*******`
Registrar Server: 10.20.30.10
SIP Port: 5060
Then:

```
Beetel Phone
     │
     │ REGISTER
     ▼
10.20.30.10
```

PBX response: 200 OK
Phone status: Registered
Now extension 1101 is active.

# 32. Where can you find the PBX IP?

## Method 1: Phone Network Information

On the phone, look for:

```
Status
Network
IP Address
```

You might find: IP Address: 192.168.20.51
Then check:

```
Gateway
DNS
```

## Method 2: Phone Web Interface

From a PC that can reach the phone: `http://PHONE_IP`
For example: `http://192.168.20.51`
Login credentials may be controlled by your organization/vendor.
Look under:

```
Account
SIP
Server
Registrar
```

You might see: `192.168.20.10`
That could be your PBX.

## Method 3: DHCP Server

Check the DHCP scope used by the phones.
You might discover:
Scope: Voice VLAN
192.168.20.0/24
And DHCP options related to provisioning/PBX.

## Method 4: Network Monitoring

From the network side, you may see SIP communication between: Phone → PBX IP

# 37. External SIP Trunk architecture

The PBX needs some connection to the real telephone network.
Usually something like:

```
Office PBX
     │
     │ SIP
     ▼
SBC / Router / Firewall
     │
     ▼
ISP / Telecom Provider
     │
     ▼
PSTN
```

Sometimes there is an additional device called an SBC.
**SBC = Session Border Controller**
Think of it roughly as a specialized border/security/control device for voice sessions.
Architecture:

```
Internal Phones
       │
       ▼
      PBX
       │
       ▼
      SBC
       │
       ▼
Telecom Provider
```

Not every company setup has a separate SBC.
Sometimes: PBX → Firewall → Provider

