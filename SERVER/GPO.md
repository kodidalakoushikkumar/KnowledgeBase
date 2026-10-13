


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