# AD Realm Config and Kerberos Details

#### The connection between your Ubuntu laptops and Active Directory is handled by three main components:

- **SSSD (System Security Services Daemon)**: This is the **client software** managing the identity and authentication against your Active Directory domain. It acts as the local broker, communicating directly with AD Domain Controllers using LDAP (to look up user/group information) and Kerberos (to authenticate credentials).
- **PAM (Pluggable Authentication Modules)**: Specifically integrated via **libpam-sss**. PAM is the framework in Ubuntu that handles authentication tasks (like logging in, checking passwords, or changing passwords). The SSSD PAM module routes local login or password-change requests to SSSD, which in turn updates Active Directory.
- **NSS (Name Service Switch)**: Specifically integrated via **libnss-sss**. This component tells Ubuntu how to resolve user and group names. It ensures that AD users (such as members of the **Domain Users** group) are recognized as valid system users by local Linux commands and permissions.

#### How Your Joining Script Works

```
#!/bin/bash
set -e  # Exit immediately if a command exits with a non-zero status
sudo apt update
sudo apt install -y realmd sssd sssd-tools adcli oddjob oddjob-mkhomedir
domain_name="impressico.local"
realm discover "$domain_name"
echo -n "Enter username for domain join: "
read domain_user
sudo realm join --user="$domain_user" "$domain_name"
realm list
CONFIG_FILE="/etc/sssd/sssd.conf"
touch "$CONFIG_FILE"
cp "$CONFIG_FILE" "${CONFIG_FILE}.bak"
if grep -q "^use_fully_qualified_names" "$CONFIG_FILE"; then
    sed -i 's/^use_fully_qualified_names.*/use_fully_qualified_names = False/' "$CONFIG_FILE"
else
    echo "use_fully_qualified_names = False" >> "$CONFIG_FILE"
fi

if grep -q "^fallback_homedir" "$CONFIG_FILE"; then
    sed -i 's|^fallback_homedir.*|fallback_homedir = /home/%u|' "$CONFIG_FILE"
else
    echo "fallback_homedir = /home/%u" >> "$CONFIG_FILE"
fi
chmod 600 "$CONFIG_FILE"
chown root:root "$CONFIG_FILE"
sudo systemctl restart sssd
sudo systemctl enable sssd
sudo pam-auth-update --enable mkhomedir
sudo realm permit --groups "Domain Users"
echo "Domain join and configuration completed successfully."
```

Based on your system configuration, the domain-joining script you ran automated several key steps:

1. **Installed Required Dependencies**: It installed packages like `sssd`, `sssd-tools`, `libpam-sss`, `libnss-sss`, `adcli`, and `samba-common-bin`.
2. **Discovered and Joined the Realm**: It performed a DNS lookup to find the Domain Controllers for `impressico.local` and ran `realm join` or `adcli join`. This created a computer account for your laptop in Active Directory, establishing a secure trust relationship and generating a local keytab file (usually `/etc/krb5.keytab`) for machine-level authentication.
3. **Configured SSSD**: It created SSSD's configuration (usually in `/etc/sssd/sssd.conf`) specifying `active-directory` as the backend, defining the login format (`%U`), and enforcing access control such as restricting login to the **Domain Users** group.
4. **Enabled SSSD in PAM and NSS**: It updated `/etc/nsswitch.conf` and updated the PAM configuration (usually via `pam-auth-update`) so that Ubuntu knows to consult SSSD for user details and authentication.

#### What `kinit` and `kpasswd` Do

- **kinit** **(Kerberos Initialize)**: This tool is used to manually authenticate against the Kerberos Key Distribution Center (KDC, which is hosted on your AD Domain Controller). When you run `kinit`, it prompts for your AD password, contacts AD, and obtains a **Ticket-Granting Ticket (TGT)**. This ticket is stored in a local cache (usually in `/tmp`) and proves your identity to Kerberos-aware services without requiring you to re-enter your password.
- **kpasswd** **(Kerberos Password)**: This is a utility specifically designed to change a user's Kerberos password directly on the KDC. Because you have a valid TGT (obtained via `kinit`), `kpasswd` uses that ticket to securely talk to the Kerberos Password Change Service (port 464) on your AD Domain Controller and update your password.

#### Why `passwd` Previously Worked vs. Why It Stopped Working

How it worked before:

When SSSD is functioning correctly, running the standard **passwd** command invokes the PAM stack. Because SSSD is integrated via `libpam-sss`, PAM automatically routes the password-change request to SSSD (`pam_sss.so`). SSSD securely contacts the Domain Controller and handles the password update in Active Directory behind the scenes. **Users do not need to run** **kinit** **manually** because SSSD handles the secure connection on behalf of the system using its machine account trust or by leveraging your credentials during the interactive `passwd` session.

#### Why it stopped working:

Since `kinit` followed by `kpasswd` successfully changes the password, your network connection to the domain controller, Kerberos DNS resolution, and AD password policies are working perfectly. The failure of `passwd` means the **SSSD-to-PAM integration is broken**. This typically happens due to:

1. **SSSD in Offline Mode**: SSSD caches credentials so users can log in when disconnected from the office. If SSSD thinks it is offline (due to a temporary network blip or startup issue), it will refuse to contact AD for password changes. When SSSD is offline, the PAM-based `passwd` command will fail, whereas direct Kerberos tools like `kinit` bypass SSSD's offline state entirely to talk directly to AD.
2. **PAM Configuration Desynchronization**: If an OS update or manual configuration change altered `/etc/pam.d/common-password` or disabled SSSD, the `passwd` command will no longer hand off the request to SSSD.
3. **SSSD Cache Issues**: The local cache database (`/var/lib/sss/db/`) can sometimes become corrupted, causing SSSD to fail to update credentials even if the network is online.
4. **Machine Account Trust Out-of-Sync**: Your Ubuntu machine's computer account password in Active Directory may have expired or been disabled in AD, breaking SSSD's secure system-level connection to the Domain Controller.

#### How to Troubleshoot and Restore `passwd`

To fix the standard `passwd` integration so users do not have to use `kinit` and `kpasswd`, try these steps on an affected machine:

1. **Clear the SSSD Cache**: Corrupted caches are a frequent culprit. You can safely clear the cache and restart SSSD by running:

```
sudo sss_cache -E
sudo systemctl restart sssd
```

1. **Verify SSSD is Enabled in PAM**: Ensure `pam_sss.so` is active in your PAM configuration. You can re-enable and configure it automatically by running: `sudo pam-auth-update`

Ensure that **SSS authentication** is checked in the menu and select **OK**.

1. **Check if SSSD is Online**: Run `sssctl domain-status impressico.local` to verify if SSSD is in "online" or "offline" mode. If it is offline, check your SSSD logs (`/var/log/sssd/sssd_impressico.local.log`) for LDAP connection errors.
2. **Test Machine Account Trust**: Check if the laptop's computer account is still trusted by AD: `sudo adcli testjoin`

If this fails, you may need to rejoin the domain using your script or by running `sudo realm join impressico.local` to renew the machine secrets.