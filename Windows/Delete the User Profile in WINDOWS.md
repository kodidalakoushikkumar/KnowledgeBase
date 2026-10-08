
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
