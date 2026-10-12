# AD Password Change

In Ubuntu 20, `passwd` often worked more transparently with domain users. In Ubuntu 24, stricter policies and updated PAM/SSSD configs can prevent local password changes for domain accounts.

### 🔍 What’s happening

When you run: `passwd` on a **domain user**, Ubuntu 24 may:

- Reject the change silently or fail
- Require proper Kerberos tickets
- Or defer password changes to the domain controller (Windows side)

Meanwhile, on Windows, the password change works because it’s directly talking to Active Directory.
### ✅ Ways to fix / work around it

#### 1. Try using `kpasswd` (Kerberos-native way)

If your system uses Kerberos (very likely), use: `kpasswd`
This talks directly to the domain controller and usually works even when `passwd` doesn’t.
If this works, it’s the _correct_ method for domain users in Ubuntu 24.
#### 2. Make sure you have a valid Kerberos ticket

Before changing password:

```
kinit your-username  
klist
```

Then: `kpasswd`, If `kinit` fails, your system isn’t properly authenticated to the domain.
#### 3. Check SSSD configuration

Ubuntu 24 relies more strictly on SSSD.
Check: `sudo nano /etc/sssd/sssd.conf`
Look for: `chpass_provider = ad`
If missing, add it under your domain section:

```
[domain/your-domain]  
chpass_provider = ad
```

Then restart: `sudo systemctl restart sssd`
#### 4. Verify PAM is allowing password changes

Check: `cat /etc/pam.d/common-password`
You should see something involving: `pam_sss.so`
If not, domain password changes won’t work.

### 🔑 `kinit your-username`

This command **logs you into the domain from the terminal**.
When you run: `kinit your-username`, It will:

- Ask for your **domain password**
- Contact the domain controller (AD)
- If successful, it gives you a **Kerberos ticket**

Think of it like: “Hey domain, verify me and give me temporary access.”
That ticket is what allows you to:

- Access domain resources
- Change password using `kpasswd`
- Authenticate without re-entering password repeatedly

### 📜 `klist`

This command shows your **current Kerberos tickets**.
Run: `klist`, You’ll see something like:

```
Ticket cache: FILE:/tmp/krb5cc_1000  
Default principal: user@DOMAIN.COM  
  
Valid starting     Expires            Service principal  
04/10/26 10:00    04/10/26 20:00     krbtgt/DOMAIN.COM@DOMAIN.COM
```

#### What this means:

- ✅ You are authenticated
- ⏳ Ticket has an expiry time
- 🎫 You can now use Kerberos-based services

## with admin permission

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -N -b "DC=impressico,DC=local" "(cn=*Koushik*)" distinguishedName
```

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -N -b "DC=impressico,DC=local" "(sAMAccountName=kodidala.kumar)" distinguishedName
```

```
ldappasswd -H ldap://SHAIL.impressico.local -Y GSSAPI -N -S "CN=Kodidala Koushik Kumar,OU=Laptop User's,OU=Team-IT,DC=impressico,DC=local"
```