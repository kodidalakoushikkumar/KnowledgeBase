---
Title:
Type:
Topic:
tags:
ee: "[[/IMAGES/BUS.png]]"
---
# 1. ICS (Industrial Control Systems)

Imagine a normal office.
Your laptop opens Outlook, Chrome, Teams.  That's **IT**.
Now imagine a factory. There are computers whose only job is to control machines.
Example:

- Conveyor belts
- Robot arms
- Power plant turbines
- Water pumps
- Oil refinery valves
- Traffic signals

Those computers form an **Industrial Control System (ICS).**

```
Office
Laptop ---- Printer
      \
       ---- Server
= IT


Factory
PLC ---- Conveyor
PLC ---- Robot
PLC ---- Motor
PLC ---- Valve
= ICS
```

Instead of opening Excel... they send commands like

```
Motor ON
Motor OFF
Valve Open
Temperature = 85°C
Pressure = 110 PSI
```

## Why hackers care

If someone hacks your office PC, Worst case steals files or encrypts data
If someone hacks an ICS, Worst case

```
Explosion
Power outage
Factory shutdown
Chemical leak
Water contamination
```

People can actually die. That's why ICS security is a separate field.
# 2. OT (Operational Technology)

People confuse ICS and OT. Think of it like this.

```
OT
│
├── ICS
├── PLC
├── SCADA
├── DCS
├── Sensors
├── Actuators
├── Industrial switches
└── RTU
```

OT means **Technology that interacts with the physical world.**
IT manages information. OT manages machines.
Example 
IT `Email server` stores emails.
OT `PLC` opens a water valve.

## IT vs OT

|IT|OT|
|---|---|
|Laptop|PLC|
|Email|Conveyor belt|
|Printer|Robot arm|
|Database|Water pump|
|File server|Turbine|

# 3. NIST

National Institute of Standards and Technology
Think of NIST as "The people who publish the rule books."
They don't manufacture products. They don't sell firewalls.
They publish documents saying  "Here's the secure way to build systems."
Example

```
Password policies
Incident response
Encryption
Cloud security
Zero Trust
Industrial security
```

Companies worldwide follow NIST guidance.

# 4. NIST SP 800-82

This is one of NIST's publications.

```
SP
Special Publication

800
Security Series

82
Document Number
```

This publication focuses on Security of Industrial Control Systems.
It explains

- PLC security
- SCADA security
- OT networks
- firewalls
- segmentation
- remote access
- malware
- ransomware
- USB attacks

If you work in ICS security, this is basically one of the main playbooks.

# 5. CAINE

CAINE stands for **Computer Aided Investigative Environment**
It is a Linux distribution built specifically for **digital forensics**.
Think `Ubuntu` for everyday work.
`Kali Linux for penetration testing.
`CAINE` for investigating computers after an incident.
It contains tools like

```
Autopsy
Volatility
Wireshark
Hashing
Disk imaging
Memory analysis
```

Suppose an employee deletes company data. Instead of booting Windows, you boot CAINE. Then

```
Read hard disk
Recover deleted files
Calculate hashes
Extract browser history
Analyse RAM
Create forensic image
```

without modifying evidence.

# 6. GHOST Protocol

This one depends on the context.
**There isn't one universally recognised cybersecurity protocol simply called "GHOST Protocol".**
It can refer to different things depending on the vendor, product, or document you're reading.
Common possibilities include:

- a vendor-specific industrial communication protocol,
- a proprietary remote-management protocol,
- or an internal project codename.

If you saw "GHOST Protocol" in a vulnerability report or scanner output, share the report or product name. The exact meaning depends on that context.

# 7. CVE

Common Vulnerabilities and Exposures
Think of CVE as A serial number for security bugs.
Example `CVE-2026-12345`
Every serious vulnerability gets a unique ID.
Example

```
Windows Bug
↓
CVE-2026-12345

Linux bug
↓
CVE-2026-56789

Cisco bug
↓
CVE-2026-90001
```

Just like

```
Car
Registration Number
MH-12-AB-1234
```

Every vulnerability gets its own identifier.

## Why useful

Instead of saying `That Windows bug` , everyone can say `CVE-2026-12345` , Everyone knows exactly which vulnerability is being discussed.
# 8. CWSS

CWSS stands for **Common Weakness Scoring System**
People confuse

```
CVE
CVSS
CWE
CWSS
```

Let's separate them.

## CVE

Specific vulnerability.
Example `Windows Print Bug` gets `CVE-2026-12345`

## CWE

Common Weakness Enumeration. Think `Programming mistakes.`
Example

```
SQL Injection
Buffer Overflow
Hardcoded Password
Improper Authentication
```

These are weakness _types_, not individual bugs.

## CVSS

Common Vulnerability Scoring System. Scores how severe a specific CVE is.
Example

```
9.8 Critical
7.4 High
5.2 Medium
2.1 Low
```

## CWSS

Scores the **underlying weakness (CWE)** rather than a single vulnerability.
Example

```
Hardcoded password
↓
How dangerous is this weakness in general?
↓
CWSS Score
```

It considers factors such as exploitability, impact, and environmental context to prioritise which classes of weaknesses deserve attention.

# Relationship between them

```
Bad Programming Practice
        │
        ▼
      CWE
        │
        ▼
Developer writes vulnerable code
        │
        ▼
Specific vulnerability discovered
        │
        ▼
Gets a CVE ID
        │
        ▼
Severity measured by CVSS
```

CWSS fits alongside this by providing a way to score the _weakness category_ (the CWE), rather than the individual vulnerability.

```
Vulnerability Management
│
├── CWE  → Weakness type
├── CWSS → Weakness score
├── CVE  → Specific vulnerability ID
└── CVSS → Severity score
```
