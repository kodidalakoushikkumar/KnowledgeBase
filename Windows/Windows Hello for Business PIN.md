
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