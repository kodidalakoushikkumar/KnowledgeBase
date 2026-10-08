
# 3️⃣ KEYRING ISSUE

**Ubuntu Keyring is like a secure password vault for your applications**.
When you log in to websites or apps such as **Chrome, Slack, Wi-Fi, VPN, etc.**, Ubuntu can store passwords, tokens, and other secrets inside the **GNOME Keyring** instead of leaving them as normal files.
Example, Suppose you log in to Slack:

1. You enter your Slack username/password.
2. Slack receives an authentication token.
3. That token may be stored securely in the **GNOME Keyring**.
4. Next time you open Slack, it tries to read that token automatically.
5. If the keyring is **locked**, corrupted, or its password doesn't match your Ubuntu login password, you may get a popup asking: **Enter password to unlock your login keyring**

#### Option 1: Remove the keyring password (Recommended if you don't need encryption)

This keeps the keyring but disables its password.

1. Install Seahorse if you don't have it: `sudo apt install seahorse`
2. Open it: `seahorse`
3. Under **Passwords → Login**:
    - Right-click **Login**
    - Select **Change Password**
    - Enter your current keyring password.
    - Leave the **New Password** and **Confirm Password** fields **blank**.
    - Accept the warning.

The keyring will still exist, but it won't prompt you anymore.
#### Option 2: Delete the keyring completely

This removes all stored passwords (Wi-Fi, VPN, browser secrets, Git credentials, etc.).

```
rm -rf ~/.local/share/keyrings
```

Log out and log back in. If an application needs a keyring later, Ubuntu may recreate it.

#### Option 3: Disable the GNOME Keyring daemon (Most aggressive)

This prevents the keyring service from starting.

```
systemctl --user mask gnome-keyring-daemon.service
systemctl --user mask gnome-keyring-daemon.socket
```

You can also mask related sockets:

```
systemctl --user mask gnome-keyring-daemon-secrets.socket
systemctl --user mask gnome-keyring-daemon-pkcs11.socket
systemctl --user mask gnome-keyring-daemon-ssh.socket
```

Then log out and log back in.
To undo:

```
systemctl --user unmask gnome-keyring-daemon.service
systemctl --user unmask gnome-keyring-daemon.socket
systemctl --user unmask gnome-keyring-daemon-secrets.socket
systemctl --user unmask gnome-keyring-daemon-pkcs11.socket
systemctl --user unmask gnome-keyring-daemon-ssh.socket
```

#### Option 4: Uninstall it

```
sudo apt purge gnome-keyring seahorse
sudo apt autoremove
```

**Warning:** Some desktop environments and applications expect GNOME Keyring to be present. Removing it may cause:

- Saved Wi-Fi passwords to stop working.
- Browsers to stop securely storing passwords.
- VPN or SSH credentials to require manual entry.


