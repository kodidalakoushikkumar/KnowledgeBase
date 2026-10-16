# AD from Ubuntu Perspective

### The Core Network Infrastructure: Active Directory and the Domain Controller

Managing individual user accounts across hundreds of separate physical computers in a corporate environment is highly inefficient. To streamline this, organizations centralize all user, group, computer, and network identities inside a unified directory service called **Active Directory (AD)**. The physical server hosting this central database and managing the domain's security policies is known as the **Domain Controller (DC)**. In your specific setup, your Ubuntu laptop (identified as `IBS-LAP-074`) is integrated into an Active Directory domain named `IMPRESSICO.LOCAL`, which relies on a Domain Controller situated at the IP address `10.60.10.3`. Operating under your corporate identity (`kodidala.kumar@IMPRESSICO.LOCAL`), your laptop queries this Domain Controller to verify your identity, inspect group memberships, or request directory details.

### Authentication and Single Sign-On: How Kerberos Secures the Network

To access network-wide resources securely without repeatedly typing your password, the network employs **Kerberos**, an authentication protocol designed to enable **Single Sign-On (SSO)**. When you log in initially, your laptop authenticates with the Domain Controller and receives a cryptographic credential called a **Ticket Granting Ticket (TGT)**. You can think of this process like entering a secure office building: you present your primary ID (your password) at the security desk once, and security hands you a company badge (the TGT). As you access various Kerberos-aware services across the domain—such as a Windows File Server, an internal web application, a Microsoft SQL Server, or a Linux SSH server—you simply present this badge to obtain service-specific tickets without having to re-enter your password.

You can view your active tickets at any time using the `klist` command, which confirms whether you are authenticated and displays your ticket expiration times. Crucially, possessing a Kerberos ticket does not automatically grant you access to every system; it simply provides trusted proof of identity that Kerberos-aware services can verify. When you successfully query an LDAP service using your ticket, your Kerberos list updates to show that in addition to the original Ticket Granting Ticket (`krbtgt`), you have acquired a specific service ticket for the LDAP directory.

### Querying the Directory: The Language of LDAP

While Kerberos handles the _authentication_ (proving who you are), **LDAP (Lightweight Directory Access Protocol)** is the standardized language and protocol used to _search_ and query the Active Directory database. If Active Directory is a massive directory database, LDAP is conceptually like a Google search engine used to ask questions, fetch user lists, or locate groups. To run these queries from the Linux terminal, admins use a utility called `ldapsearch`.

Active Directory organizes objects in a hierarchical **tree structure**, much like folders and subfolders. Because of this, LDAP paths are written using "Domain Components" or **DC** notation. For your domain, the top of the tree is written as `DC=IMPRESSICO,DC=LOCAL`. When you run a query, specifying this as the search base (using the `-b` option) tells the system to start at the root of the tree and search downwards. To keep search results focused, you use logical **filters** like `(&(objectCategory=person)(objectClass=user))`, which tell AD to return only objects that are both people and user accounts. Finally, you can specify individual **fields/columns** to retrieve, such as `sAMAccountName` (the login username) and `displayName` (the full display name), which is equivalent to displaying only those specific columns in an Excel sheet. Combining these elements with your existing Kerberos ticket (`-Y GSSAPI` utilizing SASL/GSS-SPNEGO) allows you to search the entire directory and retrieve a clean list of domain usernames and display names without ever typing your password—a prime example of Single Sign-On in action.

### Linux-to-Windows Integration: SSSD vs. Winbind

To make Ubuntu understand Windows Active Directory identities, the system must utilize a bridge daemon. The two most common integration mechanisms are **SSSD** (System Security Services Daemon) and **Winbind**.

- **SSSD** is the modern integration standard. It translates Active Directory identities so that native Linux commands can understand them. When SSSD is running, a command like `getent passwd` (which lists all users known to the OS) can query Active Directory and display domain users mixed right in with your local accounts. You can also query specific AD users on demand using `getent passwd <username>` to verify that Ubuntu is successfully communicating with Active Directory. However, depending on SSSD configuration settings, a broad command like `getent passwd` may not always list every domain user automatically.
- **Winbind** is an alternative integration daemon that utilizes Samba. When using Winbind, you can list all domain users with `wbinfo -u` and all domain groups with `wbinfo -g`, which is the closest terminal equivalent to listing all AD accounts.

Since SSSD and Winbind are completely different, mutually exclusive integration layers, running Winbind commands like `wbinfo -u` on an SSSD-configured machine will fail with errors. This does not mean Active Directory is offline; it simply means your system relies on SSSD instead of Winbind, making the `wbinfo` command irrelevant to your setup.

### Diagnostics and Graphical Administration Tools

On Windows, administrators manage directories using graphical consoles like **Active Directory Users and Computers**. While the default Ubuntu settings panel is restricted to managing local user accounts, Linux administrators can achieve a similar directory-browsing experience. For a graphical interface, you can use specialized tools like **Apache Directory Studio** or **LXAD**, which provide an LDAP GUI to browse organizational containers (or **OUs**) and directory objects.

In the terminal, the single most informative command for diagnosing your integration is `realm list`. This command instantly tells you how your Ubuntu machine is joined to the domain, whether SSSD or Winbind is driving the integration, and what configuration options are active. To troubleshoot further, administrators can inspect configuration files like `/etc/sssd/sssd.conf` (making sure to avoid leaking passwords) or `/etc/krb5.conf`, and use network utilities like `nslookup` to ensure your machine can resolve the Domain Controller over your network's DNS.

### The Linux-to-Windows Translation Layer: SSSD vs. Winbind

To understand how a Linux computer operates within a Windows-centric Active Directory environment, it is helpful to look at how operating systems manage user identities1. By default, Linux systems identify users locally through standard files like `/etc/passwd`12. Active Directory, on the other hand, relies on centralized Windows accounts1. For Ubuntu to understand and work with these domain-level accounts, it requires a specialized translation layer or bridge12. The two primary technologies that handle this translation are **SSSD (System Security Services Daemon)** and **Winbind**13.

Both SSSD and Winbind are designed to connect Linux to Active Directory, handle domain user lookups, facilitate domain logins, and operate in tandem with Kerberos3. However, they do so in different ways. **Winbind** is a service closely tied to the Samba suite, offering deep Samba integration and providing a CLI command-line tool called `wbinfo`2more_horiz. When using Winbind, commands like `wbinfo -u` and `wbinfo -g` query the daemon to list known domain users and groups respectively35.

In contrast, **SSSD** is the modern Linux standard for directory service integration3. SSSD does not use Samba's Winbind under the hood34. If your Ubuntu laptop is configured to use SSSD, running `wbinfo` commands will result in errors35. This failure does not indicate a broken connection to Active Directory; it simply means that `wbinfo` is irrelevant to your system because SSSD is managing your identity integration instead of Winbind3more_horiz. Both systems are capable of managing Kerberos, LDAP, and DNS underneath, but they use different internal routines and client tools47.

### Group Policy Objects (GPOs) and Their Limitations on Linux

In a standard Windows environment, administrators use **Group Policy Objects (GPOs)** to push registry edits, security baselines, and configuration changes to computers across the network89. When you look at how a GPO is built, it actually exists in two completely separate parts on the Domain Controller:

1. **The Active Directory Component:** Inside Active Directory, a GPO is represented as an LDAP object under a container like `CN=Policies,CN=System,DC=IMPRESSICO,DC=LOCAL`810. This object contains metadata, including the policy's name and the file path where its configurations reside11. You can search and view these AD GPO objects and their metadata using LDAP queries811.
2. **The SYSVOL Component:** The actual, physical policy files, administrative templates, and configuration scripts are stored on the Domain Controller in a shared folder called **SYSVOL** (typically accessed as a network share path like `\\IMPRESSICO.LOCAL\sysvol\IMPRESSICO.LOCAL\Policies\...`)12. The LDAP object in Active Directory references these physical files by mapping a human-readable display name to a unique identifier that matches a directory inside SYSVOL12.

A major technical distinction arises when discussing whether Ubuntu can apply these GPOs12. Windows GPOs are heavily designed around Windows-specific mechanisms like the Windows Registry, software installation paths, and desktop settings9. Because Linux does not have a Windows Registry or identical operating system directories, **Ubuntu cannot automatically parse and apply normal Windows GPO settings**912.

However, there is an exception: **SSSD can interact with AD GPO access control configurations**913. When configured to do so, SSSD can read specific security-related GPOs from Active Directory to determine host-access control—meaning it can evaluate whether a particular Active Directory user is permitted to log in or SSH into that specific Linux laptop913. SSSD uses this policy logic strictly for login authorization, without acting as a full-fledged client for typical Windows registry policies13.

### Solving Kerberos Naming Issues: IP Address vs. Hostname

When trying to execute LDAP queries against a Domain Controller using Kerberos authentication, you might encounter a GSSAPI failure such as: `GSSAPI Error: Unspecified GSS failure. Minor code may provide more information (Server not found in Kerberos database)`1014. This error does not mean your credentials are bad, nor does it represent an LDAP permissions issue1516. Instead, it highlights a fundamental characteristic of how **Kerberos** handles network security15.

Kerberos operates strictly on **Service Principal Names (SPNs)**, which are registered hostnames rather than raw network addresses1517. If you run an LDAP search command and direct it to an IP address (e.g., `ldap://10.60.10.3`), the system tries to request a Kerberos service ticket for a name associated with that IP address1017. Because the Domain Controller's Kerberos database registers its LDAP service using fully qualified domain names (FQDNs)—like `DC01.IMPRESSICO.LOCAL`—it does not recognize the raw IP address as a valid principal name1517. Consequently, the Kerberos negotiation fails before the LDAP search can even begin16.

To resolve this issue, you must first discover the Domain Controller’s true hostname by using local lookup commands like `nslookup 10.60.10.3` or `dig -x 10.60.10.3`1518. Once you have determined the FQDN returned by your network's DNS, you must replace the IP address in your LDAP host option (such as changing `-H ldap://10.60.10.3` to `-H ldap://your-dc-hostname.IMPRESSICO.LOCAL`)1519. This ensures that your system requests the correct ticket from the Kerberos Key Distribution Center (KDC), allowing the client to establish a secure, authenticated session without requiring manual password prompts1719.

### Constructing an Active Directory Query Lab on Ubuntu

Once your Kerberos tickets are active (which you can verify using `klist`) and your Domain Controller's hostname is resolved, you can construct `ldapsearch` commands to retrieve a wealth of information from Active Directory17more_horiz. Every command follows a uniform pattern2223: $$\text{ldapsearch} \rightarrow \text{Server Flag (-H)} \rightarrow \text{Search Base (-b)} \rightarrow \text{Authentication (-Y GSSAPI)} \rightarrow \text{Logical Filter} \rightarrow \text{Target Attributes}$$
Using this syntax, you can target specific administrative details across your domain:

- **Inspecting Your Own Account Details:** Running a targeted query for your username (e.g., using the filter `(sAMAccountName=kodidala.kumar)`) retrieves your complete Active Directory record2425. Rather than dumping every single attribute, you can append specific attributes to your command to display only your **Distinguished Name (DN)**, display name, and login credentials25. The DN represents your unique, complete path within the hierarchical directory tree25.
- **Searching for Disabled User Accounts:** Active Directory tracks account state flags using a multi-bit integer attribute called **userAccountControl**26. One specific bit in this integer represents the `ACCOUNTDISABLE` flag (which has a hexadecimal value of `0x0002` or decimal `2`)26. To ask Active Directory to find users with this flag enabled, you utilize a bitwise matching filter: `(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))`26.
- **Locating Locked Accounts:** Locked-out accounts (usually resulting from too many failed login attempts) are flagged via the `lockoutTime` attribute or checked via computed lockout states27. You can query active lockouts by using the bitwise filter `(userAccountControl:1.2.840.113556.1.4.803:=16)`, which isolates the lockout flag (decimal `16`)27. If the search returns no objects, it simply means there are currently no locked-out user accounts in that part of the directory27.
- **Targeting specific OUs:** By default, directory searches start at the root of the domain (`DC=IMPRESSICO,DC=LOCAL`) and search downward28. If you only want to see users in a specific department, such as IT Support, you must modify your search base option (`-b`) to target that specific **Organizational Unit (OU)**29. To find your available OU paths, you can first run a broad query filtering for `(objectClass=organizationalUnit)`29.
- **Analyzing Security Groups and Membership:** Active Directory defines groups as distinct objects29. You can list all domain groups using the filter `(objectClass=group)`29. When investigating who belongs to a group, LDAP does not return basic text names; it returns the complete Distinguished Name of every member inside the `member` attribute30. To perform a reverse lookup and see every group a specific user belongs to, you can query using the filter `(member=CN=Kumar...,OU=...,DC=...)`3031.
- **Auditing Domain Computers:** Computers are also represented as individual security objects in Active Directory31. By running a query with the filter `(objectCategory=computer)` or `(objectClass=computer)`, you can retrieve information on every machine joined to the domain2231.

You can extract specific columns of data by specifying attributes at the end of your query2231. Active Directory maps system properties to specific LDAP attributes:

| Information Target                 | Active Directory LDAP Attribute |
| ---------------------------------- | ------------------------------- |
| **Computer Name**                  | `name` or `cn`31                |
| **Operating System**               | `operatingSystem`31             |
| **Distinguished Name**             | `distinguishedName`31           |
| **Object Creation Date**           | `whenCreated`31                 |
| **Last Modified Date**             | `whenChanged`31                 |
| **OS Version**                     | `operatingSystemVersion`31      |
| **DNS Hostname**                   | `dNSHostName`31                 |
| **Last Logon Timestamp**           | `lastLogonTimestamp`31          |
| **Account Control Flags**          | `userAccountControl`31          |
| **When password was last changed** | `pwdLastSet`                    |
| **Account lockout information**    | `lockoutTime`                   |
| **Bad password attempts**          | `badPwdCount`                   |
| **Account expiration**             | `accountExpires`                |
| **Groups the user belongs to**     | `memberOf`                      |
| **User's manager**                 | `manager`                       |
| **Unique AD object ID**            | `objectGUID`                    |
| **Department**                     | `department`                    |
| **Job title**                      | `title`                         |
| **Username**                       | `sAMAccountName`                |
| **Full name**                      | `displayName`                   |
| **Email**                          | `mail`                          |

By executing a structured search using these attributes (e.g., appending `name operatingSystem distinguishedName` to your computer query), you can format clear, readable reports on your network's physical assets directly from your Ubuntu terminal22.

```
KRB5_CONFIG=<(printf '[libdefaults]\ndefault_realm = IMPRESSICO.LOCAL\nrdns = false\ndns_canonicalize_hostname = false\n') ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(objectClass=user)" sAMAccountName
```

#### How Linux Talks to Active Directory

To interact with AD from a Linux terminal, you use two primary protocols:

1. **LDAP (Lightweight Directory Access Protocol)**: The language used to search and modify objects in the AD database. The tool `ldapsearch` is used for **reading** data, while `ldapmodify` is used for **writing** or editing data.
2. **Kerberos (GSSAPI)**: The authentication protocol that lets you log in securely. By using the `-Y GSSAPI` flag in `ldapsearch`, you are telling Linux to use your active Kerberos login ticket (which you get from running `kinit`) to authenticate to AD automatically without typing a password in plain text

#### Demystifying the Confusing LDAP Filters

In your log, you encountered this filter: `"(&(objectClass=computer)(userAccountControl:1.2.840.113556.1.4.803:=2))"`
Here is exactly what that filter means, broken down step-by-step:

- **&** **(AND Operator)**: This specifies that **both** conditions inside the parentheses must be true.
- **objectClass=computer**: This tells AD to only look for computer accounts, ignoring users, groups, or organizational units.
- **userAccountControl**: This is an integer attribute in AD that acts as a collection of binary "on/off" switches (known as bit flags) representing the status of an account (such as whether it is disabled, locked, or requires a password change).
- **The Number** **2**: This is the specific binary bit that represents `ACCOUNTDISABLE` (Account is Disabled).
- **The Mysterious OID** **1.2.840.113556.1.4.803**: This is an LDAP matching rule called **LDAP_MATCHING_RULE_BIT_AND**.
    - _Why can't we just write_ _userAccountControl=2__?_ Because an account can have multiple flags turned on at the same time. For example, if an account is disabled (`2`) _and_ its password never expires (`65536`), AD stores the sum `65538`.
    - If you searched for `userAccountControl=2`, you would completely miss this account. The matching rule performs a **bitwise AND**, which extracts only the specific "disabled" switch (bit `2`) to see if it is turned on, regardless of what other switches are active.

| Filter                           | Finds            |
| -------------------------------- | ---------------- |
| `objectClass=user`               | User objects     |
| `objectClass=computer`           | Computer objects |
| `objectClass=group`              | Groups           |
| `objectClass=organizationalUnit` | OUs              |
| `objectClass=domain`             | Domain object    |

#### Understanding `lockoutTime` & Windows FILETIME

You saw a value like `lockoutTime: 134256308026930830` and wondered how to read it1112.

- Active Directory does not store dates as simple Unix timestamps. Instead, it uses **Windows FILETIME**, which is the number of **100-nanosecond intervals that have elapsed since January 1, 1601 (UTC)**1112.
- To convert this to a normal, human-readable date, you can use Python in your Linux terminal12. The formula divides the FILETIME value by `10` because Python's `timedelta` library measures time in microseconds ($10^{-6}$ seconds), and there are ten 100-nanosecond intervals in one microsecond ($100\text{ ns} \times 10 = 1000\text{ ns} = 1\text{ }\mu\text{s}$)12.

**The Python Conversion Code**

```
python3 -c "from datetime import datetime,timezone; print(datetime(1601,1,1,tzinfo=timezone.utc)+__import__('datetime').timedelta(microseconds=134256308026930830/10))"
```

This converts the huge number into the actual date: **August 28, 2026, at 12:06:42 PM UTC**1213. Note that a `lockoutTime` of `0` means the account is not locked
#### The Kerberos / SASL Troubleshooting Journey

Your terminal session showed a common, frustrating issue: your queries worked fine, and then suddenly failed with `Server not found in Kerberos database`.

Here is what went wrong and why:

1. **The Hostname Canonicalization Loop**: When you type `SHAIL.impressico.local`, Kerberos and OpenLDAP try to be helpful. They query DNS, get the server's IP address, and then perform a reverse lookup to find the "canonical" hostname.
2. **The Error**: If DNS reverses this IP to a generic address (like an IP string `10.60.10.3`), Kerberos requests a ticket for `ldap/10.60.10.3` instead of `ldap/SHAIL.impressico.local`. Because Active Directory does not have an entry for that IP name in its Kerberos database, it throws the `Server not found` error.

The Fixes:

- **The** **/etc/krb5.conf** **Fix**: Adding these two lines under the `[libdefaults]` section of `/etc/krb5.conf` tells the Kerberos library to stop doing reverse DNS lookups and stop canonicalizing hostnames:

```
rdns = false
dns_canonicalize_hostname = false
```

- **The** **-N** **Flag**: Sometimes the OpenLDAP client library itself tries to canonicalize the hostname during GSSAPI bindings. Adding the **-N** **flag** to your `ldapsearch` command tells it to bypass this step entirely and use the exact host you provided:

```
ldapsearch -N -H ldap://SHAIL.impressico.local -Y GSSAPI ...
```

#### Writing and Resetting (The Secure LDAP Rule)

Your notes point out that while `ldapsearch` works over plain LDAP, Active Directory **will reject password changes or resets over unencrypted connections (****ldap://****)**.

- AD stores passwords in a special attribute called `unicodePwd`.
- To change or reset passwords, AD enforces a security rule requiring **LDAPS (Secure LDAP over SSL/TLS, typically port 636)** or a StartTLS connection. You cannot use a plain connection for password modifications

#### Search for specific User

```
KRB5_CONFIG=<(printf '[libdefaults]\ndefault_realm = IMPRESSICO.LOCAL\nrdns = false\ndns_canonicalize_hostname = false\n') ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -N -b "DC=IMPRESSICO,DC=LOCAL" "(sAMAccountName=kodidala.kumar)" whenCreated whenChanged lastLogonTimestamp lastLogon pwdLastSet badPwdCount lockoutTime badPasswordTime logonCount memberOf
```

### Modify AD Account with ldapmodify

`ldapmodify` is basically the **write counterpart to `ldapsearch`**.

- `ldapsearch` → read AD
- `ldapmodify` → modify AD attributes
- `ldapadd` → create an object
- `ldapdelete` → delete an object

```
printf 'dn: CN=Konduru Manasa Veena,OU=Team-Trainee,DC=impressico,DC=local\nchangetype: modify\nreplace: unicodePwd\nunicodePwd:: '$(printf '"Support@2026@OCT"' | iconv -f UTF-8 -t UTF-16LE | base64 -w0)'\n' | KRB5_CONFIG=<(printf '[libdefaults]\ndefault_realm = IMPRESSICO.LOCAL\nrdns = false\ndns_canonicalize_hostname = false\n') ldapmodify -H ldaps://SHAIL.impressico.local -Y GSSAPI -N
```

With your Kerberos setup, you can use `ldapmodify -Y GSSAPI`.
#### 1. Basic `ldapmodify` structure

The general format is: `ldapmodify -H ldap://SHAIL.impressico.local -Y GSSAPI`
Then provide an LDIF operation:

```
dn: <distinguished name>
changetype: modify
replace: <attribute>
<attribute>: <new value>
```

For example:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
replace: department
department: IT Support
```

For example:

```
dn: uid=username,ou=users,dc=example,dc=com
changetype: modify
replace: userPassword
userPassword: NewPassword123
```

This means: Find John Doe and replace his `department` with `IT Support`.
#### 2. `add`

Use `add` when the attribute doesn't currently have the value you want.

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
add: telephoneNumber
telephoneNumber: +91-1234567890
```

Command: `ldapmodify -H ldap://SHAIL.impressico.local -Y GSSAPI`
Paste the LDIF and press: `Ctrl+D`
#### 3. `replace`

This replaces the existing value.

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
replace: department
department: Infrastructure
```

This is useful for things like:

```
department
title
description
telephoneNumber
mobile
physicalDeliveryOfficeName
manager
```

assuming you have permission to modify them.

#### 4. `delete`

Remove an attribute entirely:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
delete: telephoneNumber
```

Or remove a **specific value**:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
delete: memberOf
memberOf: CN=SomeGroup,OU=Groups,DC=IMPRESSICO,DC=LOCAL
```

However, **don't manually modify `memberOf`** in normal AD administration. Group membership is normally changed through the group's `member` attribute.
#### 5. Multiple modifications in one operation

You can modify several attributes at once:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify

replace: department
department: IT Support
-
replace: title
title: Desktop Support Engineer
-
replace: telephoneNumber
telephoneNumber: +91-1234567890
```

The `-` separates individual modifications.
#### 6. Unlock an account

This is one of the useful AD-specific modifications.

```
dn: <USER-DN>
changetype: modify
replace: lockoutTime
lockoutTime: 0
```

For example:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modify
replace: lockoutTime
lockoutTime: 0
```

```
ldapmodify -H ldap://SHAIL.impressico.local -Y GSSAPI -N <<'EOF'
dn: CN=Kodidala Koushik Kumar,OU=Laptop User's,OU=Team-IT,DC=impressico,DC=local
changetype: modify
replace: lockoutTime
lockoutTime: 0

EOF
```

```
ldapmodify -H ldap://SHAIL.impressico.local -Y GSSAPI -N <<'EOF'
dn: CN=Naroju Supriya,OU=Team-Trainee,DC=impressico,DC=local
changetype: modify
replace: lockoutTime
lockoutTime: 0

EOF
```


CN=Madhava Reddy,OU=Laptop User's,OU=Yapstone-Project,DC=impressico,DC=loc

That clears the lockout timestamp. **Your account needs appropriate AD permissions.**
#### 7. Enable a disabled account

This is more interesting because `userAccountControl` is a **bit field**.
Don't blindly do:

```
replace: userAccountControl
userAccountControl: 512
```

because that can accidentally remove other account flags.  Instead, the safest administrative approach is usually to use an AD-aware tool such as PowerShell/AD tools, or calculate the existing flags correctly before modifying them.
For example, normal enabled user accounts commonly have: `512 = NORMAL_ACCOUNT`
while: `514 = NORMAL_ACCOUNT + ACCOUNTDISABLE` , So:

```
512 → enabled
514 → disabled
```

But real accounts can have additional flags.
#### 8. Disable an account

Same issue in reverse: you shouldn't blindly overwrite `userAccountControl`.
You need to preserve the existing flags and set the `ACCOUNTDISABLE` bit.
The relevant bit is: 2
Conceptually: `new UAC = old UAC OR 2`
To enable: `new UAC = old UAC AND NOT 2`
This is why `userAccountControl` is more complicated than ordinary LDAP attributes.
#### 9. Change group membership

Suppose you have: `CN=IT Support,OU=Groups,DC=IMPRESSICO,DC=LOCAL`
and want to add a user.
Modify the **group**, not `memberOf` on the user:

```
dn: CN=IT Support,OU=Groups,DC=IMPRESSICO,DC=LOCAL
changetype: modify
add: member
member: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
```

Remove them:

```
dn: CN=IT Support,OU=Groups,DC=IMPRESSICO,DC=LOCAL
changetype: modify
delete: member
member: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
```

This is a very important AD concept:

```
User
  │
  │ memberOf
  ▼
Group

But when modifying membership:

Group
  │
  │ member
  ▼
User
```

#### 10. Resetting a password

Password modification is a special case. AD's password attribute is: `unicodePwd`
You generally need **LDAPS (`ldaps://`) or another protected channel**.
You cannot safely do: `ldapmodify -H ldap://...` and send a password over ordinary LDAP.
An administrative password reset also requires appropriate privileges.
So don't treat `unicodePwd` like:

```
department
telephoneNumber
description
```

#### 11. Moving an object

`ldapmodify` can also perform a **Modify DN** operation.
For example:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
changetype: modrdn
newrdn: CN=John Smith
deleteoldrdn: 1
```

That changes the object's RDN/name. You can also move objects between OUs using `newsuperior`, subject to permissions.

#### 12. What can you modify?

Potentially **many AD attributes**, but there are three important categories.
##### Normal editable attributes

Examples:

```
displayName
description
department
title
telephoneNumber
mobile
mail
physicalDeliveryOfficeName
manager
streetAddress
l
st
postalCode
```

Whether you can modify them depends on your permissions.
##### Administrative attributes

Examples:

```
lockoutTime
userAccountControl
member
group membership
accountExpires
pwdLastSet
```

These have more consequences and often require elevated permissions or special handling.
##### Sensitive/protected attributes

Examples:

```
unicodePwd
objectSid
objectGUID
sAMAccountName
servicePrincipalName
```

These shouldn't be modified casually. Some are system-controlled, protected, or require special protocols/permissions.

#### 13. First thing you should always do

Before modifying an object, get its DN and current attributes.

For example:

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI \
-b "DC=IMPRESSICO,DC=LOCAL" \
"(sAMAccountName=USERNAME)" \
distinguishedName userAccountControl lockoutTime department memberOf
```

You'll get something like:

```
dn: CN=John Doe,OU=Users,DC=IMPRESSICO,DC=LOCAL
userAccountControl: 512
lockoutTime: 0
department: IT Support
```

Then you know **exactly which object you're modifying and what its current state is**.

## AD accounts/computers configured for Kerberos delegation

LDAP can find them by querying `userAccountControl` and `msDS-Allowed*` attributes.
#### 1. Unconstrained delegation

Find user accounts with the `TRUSTED_FOR_DELEGATION` bit (`524288`):

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=524288))" sAMAccountName distinguishedName userAccountControl
```

For **computers**:

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(&(objectClass=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))" name dNSHostName userAccountControl
```
#### 2. Constrained delegation

Look for: `msDS-AllowedToDelegateTo`

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(msDS-AllowedToDelegateTo=*)" sAMAccountName name msDS-AllowedToDelegateTo
```

This tells you **which service principals the account is allowed to delegate to**.
#### 3. Resource-based constrained delegation (RBCD)

Look for: `msDS-AllowedToActOnBehalfOfOtherIdentity`

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(msDS-AllowedToActOnBehalfOfOtherIdentity=*)" name dNSHostName
```

This is particularly important because the attribute is attached to the **target computer/service**, and its value is an access-control descriptor rather than a simple username.
#### 4. Find privileged/admin groups

If you mean **who are the actual domain administrators**, start with: `ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(cn=Domain Admins)" member`

Other important built-in groups include:

```
Domain Admins
Enterprise Admins
Administrators
Schema Admins
Account Operators
Server Operators
Backup Operators
Print Operators
DNS Admins
Group Policy Creator Owners
```

For example:

```
ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -b "DC=IMPRESSICO,DC=LOCAL" "(|(cn=Domain Admins)(cn=Enterprise Admins)(cn=Administrators)(cn=Schema Admins)(cn=Account Operators)(cn=Server Operators)(cn=Backup Operators)(cn=DNS Admins))" cn member
```

#### One important distinction

**Delegation ≠ administrator.**
For example:

```
Domain Admin
    ↓
Administrative privilege

Kerberos delegation
    ↓
Ability for a service/account to act on behalf of users
```

An account can have delegation configured without being a Domain Admin, and a Domain Admin doesn't automatically mean the account is configured for delegation.

# RootDSE, DNS, and SMB Discovery

The three groups of commands you are using answer three different questions:

```
1. LDAP RootDSE  →  "What AD environment am I connected to?"
2. DNS SRV       →  "Where are the AD services?"
3. SMB           →  "What shares does this server expose?"
```

The overall picture is:

```
                    Active Directory
                           │
              ┌────────────┼────────────┐
              │            │            │
             LDAP          DNS          SMB
              │            │            │
          AD structure   Services      Shares
              │            │            │
          RootDSE       SRV records   NETLOGON
          Users         DCs           SYSVOL
          Computers     LDAP          IPC$
          Groups        Kerberos
          OUs           Global Catalog
```

## 1. LDAP RootDSE discovery

Your command:

```
KRB5_CONFIG=<(printf '[libdefaults]\ndefault_realm = IMPRESSICO.LOCAL\nrdns = false\ndns_canonicalize_hostname = false\n') ldapsearch -H ldap://SHAIL.impressico.local -Y GSSAPI -N -b "" -s base "(objectClass=*)" defaultNamingContext configurationNamingContext rootDomainNamingContext dnsHostName serverName namingContexts
```

This looks complicated because several commands/options are combined together. Let's break it apart.

### 1.1 `KRB5_CONFIG=...`

```
KRB5_CONFIG=<(printf '[libdefaults]
default_realm = IMPRESSICO.LOCAL
rdns = false
dns_canonicalize_hostname = false
')
```

`KRB5_CONFIG` tells Kerberos: "For this command, use this Kerberos configuration."
Normally Kerberos configuration comes from: `/etc/krb5.conf`
But you're temporarily providing a small configuration instead.

#### `default_realm`

```
default_realm = IMPRESSICO.LOCAL
```

Your AD/Kerberos realm is: `IMPRESSICO.LOCAL`
Kerberos realms are conventionally written in uppercase. Your AD DNS domain is: `impressico.local` So:

```
DNS domain:       impressico.local
Kerberos realm:   IMPRESSICO.LOCAL
```

They normally correspond to the same AD environment.
#### `rdns = false`

`rdns = false` This tells Kerberos: Don't perform reverse DNS lookups when determining the identity of a host.
This is useful in your environment because you don't want Kerberos unnecessarily doing:

```
hostname
   ↓
IP
   ↓
reverse DNS
   ↓
hostname
```

#### `dns_canonicalize_hostname = false`

This tells Kerberos: Don't automatically change/canonicalize the hostname you gave it.
You're explicitly connecting to: `SHAIL.impressico.local`
so Kerberos should use that hostname rather than trying to transform it.

### 1.2 `ldapsearch`

`ldapsearch` This is an LDAP client. LDAP is the protocol you use from Ubuntu to communicate with Active Directory's directory service.
Think:

```
Ubuntu
  │
  │ LDAP
  ▼
Domain Controller
  │
  ▼
Active Directory
```

### 1.3 `-H ldap://SHAIL.impressico.local`

```
-H ldap://SHAIL.impressico.local
```

`-H` specifies the LDAP server.
You're saying: Connect to the LDAP service on SHAIL.
The protocol is: `ldap://`
Normally LDAP uses: `TCP 389`
So conceptually:

```
Ubuntu
  │
  │ TCP 389
  ▼
SHAIL.impressico.local
  │
  ▼
Active Directory LDAP
```

### 1.4 `-Y GSSAPI`

`-Y GSSAPI` This specifies the SASL authentication mechanism.
GSSAPI allows LDAP to use Kerberos authentication.
So instead of: `username + password`
you're using your existing Kerberos credentials.
Your previous output:

```
SASL/GSSAPI authentication started
SASL username: kodidala.kumar@IMPRESSICO.LOCAL
SASL SSF: 256
SASL data security layer installed.
```

means Kerberos/GSSAPI authentication succeeded.
### 1.5 `-N`

`-N` This tells `ldapsearch` not to use reverse-DNS canonicalization for the SASL hostname.
It's particularly useful in your setup because you're explicitly controlling the hostname.
### 1.6 `-b ""`

This is very important.
`-b ""` means: Use an empty LDAP search base.
Normally you might search: `DC=impressico,DC=local` for users or computers.
But RootDSE is special.
RootDSE is located at the **top/root of the LDAP server**, so you use: `-b ""`
Think of it as:

```
LDAP server
└── RootDSE
```

rather than:

```
DC=impressico,DC=local
└── Users
└── Computers
└── ...
```

### 1.7 `-s base`

`-s base` means: Search only the exact object specified by the search base.
Since: `-b ""` and: `-s base` together mean: "Give me information about the LDAP server's RootDSE itself."
You're **not searching all users/computers**.

### 1.8 `(objectClass=*)`

`"(objectClass=*)"` This is the LDAP search filter.
`*` means essentially:  Match an object with an objectClass.
Because you're querying RootDSE with `-s base`, you're asking for the RootDSE object.

### 1.9 The attributes you're requesting

At the end you have:

```
defaultNamingContext
configurationNamingContext
rootDomainNamingContext
dnsHostName
serverName
namingContexts
```

These are the important pieces.

#### `defaultNamingContext`

Your output was: `defaultNamingContext: DC=impressico,DC=local`
This tells you: What is the default AD domain partition?
Your domain is: `DC=impressico,DC=local`
In human terms: `impressico.local`
 So when you want to search normal AD objects, you can use: `-b "DC=impressico,DC=local"`
For example:

```
DC=impressico,DC=local
├── CN=Users
├── CN=Computers
├── OU=...
└── ...
```

#### `configurationNamingContext`

You got:

```
configurationNamingContext:
CN=Configuration,DC=impressico,DC=local
```

This is the **Configuration partition**.
It contains forest-wide AD configuration information. Conceptually:

```
Forest
│
├── Domain partition
│    └── impressico.local
│
├── Configuration partition
│    └── CN=Configuration,...
│
└── Schema partition
```

The Configuration partition contains information such as:

- Sites
- Domain Controllers
- Replication configuration
- Services
- Some infrastructure configuration
- DFS-related configuration in environments where it is stored there

That's why you previously used: `-b "CN=Configuration,DC=impressico,DC=local"` for DFS discovery.

#### `rootDomainNamingContext`

You got:

```
rootDomainNamingContext:
DC=impressico,DC=local
```

This tells you the **root domain of the AD forest**.
In your environment the forest root is: `impressico.local`
If an organization had something like:

```
corp.example.com
child.corp.example.com
```

the root domain might be: `corp.example.com`
while the current domain could be: `child.corp.example.com`
In your environment they're currently the same.

#### `dnsHostName`

You got: `dnsHostName: SHAIL.impressico.local`
This is the DNS hostname of the server you're connected to.
So:

```
LDAP server
    ↓
SHAIL.impressico.local
```

And you already know from DNS that:

```
SHAIL.impressico.local
        ↓
    10.60.10.3
```

#### `serverName`

You got something similar to:

```
serverName:
CN=SHAIL,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=impressico,DC=local
```

This is an **AD Distinguished Name (DN)** identifying SHAIL's server object inside the Configuration partition. Break it down:

```
CN=SHAIL
   ↓
CN=Servers
   ↓
CN=Default-First-Site-Name
   ↓
CN=Sites
   ↓
CN=Configuration
   ↓
DC=impressico
DC=local
```

The interesting part for now is:

```
SHAIL
   │
   └── Server object
        │
        └── AD Site
```

This is more infrastructure information than you need for basic resource discovery, but it becomes useful when learning AD topology.

### `namingContexts`

This is one of the most useful RootDSE attributes. You saw:

```
namingContexts:
DC=impressico,DC=local

CN=Configuration,DC=impressico,DC=local

CN=Schema,CN=Configuration,DC=impressico,DC=local

DC=ForestDnsZones,DC=impressico,DC=local

DC=DomainDnsZones,DC=impressico,DC=local
```

This tells you: What LDAP directory partitions are available from this server?
Think of them as different databases/sections of the AD directory.

### Your environment

```
AD LDAP
│
├── DC=impressico,DC=local
│     → Normal domain objects
│
├── CN=Configuration,...
│     → Forest configuration
│
├── CN=Schema,...
│     → AD object/schema definitions
│
├── DC=ForestDnsZones,...
│     → Forest-wide DNS data
│
└── DC=DomainDnsZones,...
      → Domain DNS data
```

This is why RootDSE discovery is so useful. You don't have to hardcode these values.
Your script can ask: "What naming contexts does this AD server provide?"
and then use the returned values.

## 2. DNS SRV discovery

Now your three commands:

```
dig @SHAIL.impressico.local _gc._tcp.impressico.local SRV
dig @SHAIL.impressico.local _ldap._tcp.impressico.local SRV
dig @SHAIL.impressico.local _ldap._tcp.dc._msdcs.impressico.local SRV
```

These are **service discovery queries**.
Instead of asking: "What IP belongs to this hostname?"
you're asking: "Where is this particular AD service?"
### What is an SRV record?

SRV = **Service Record**.
It tells a client:

```
Service
   ↓
Protocol
   ↓
Domain
   ↓
Which server?
   ↓
Which port?
```

For example: `_ldap._tcp.impressico.local`
means approximately:

```
LDAP
 ↓
TCP
 ↓
impressico.local
```

### 2.1 `_ldap._tcp.impressico.local`

Your earlier result contained:

```
_ldap._tcp.impressico.local. 600 IN SRV 0 100 389 nandi.impressico.local.
_ldap._tcp.impressico.local. 600 IN SRV 0 100 389 gana.impressico.local.
_ldap._tcp.impressico.local. 600 IN SRV 0 100 389 shail.impressico.local.
```

Let's understand one line: `0 100 389 shail.impressico.local.`
The important fields are:

```
priority   weight   port   target
   0        100      389   shail.impressico.local
```

So:

```
LDAP service
     ↓
TCP
     ↓
Port 389
     ↓
SHAIL
```

Your DNS server told you that these hosts advertise LDAP:

```
NANDI → 389
GANA  → 389
SHAIL → 389
```

This is **service discovery**. You didn't hardcode those servers.

### 2.2 `_ldap._tcp.dc._msdcs.impressico.local`

This is more specifically related to **Domain Controllers**.

```
_ldap
  ↓
LDAP

_tcp
  ↓
TCP

dc
  ↓
Domain Controller

_msdcs
  ↓
Microsoft Domain Controller DNS discovery namespace
```

So this query is useful when you specifically want to discover **domain controllers advertising LDAP**.
This is more precise for your AD discovery script than simply asking: `_ldap._tcp.impressico.local`

### 2.3 `_gc._tcp.impressico.local`

GC means: **Global Catalog**
A Global Catalog is an AD service that provides a searchable representation of objects across the forest.
The normal Global Catalog port is: `3268`
and secure GC commonly uses: `3269`
So: `dig @SHAIL.impressico.local _gc._tcp.impressico.local SRV`
asks: Which servers provide the Global Catalog service?
Conceptually:

```
AD forest
   │
   └── Global Catalog
          │
          ├── Server A : 3268
          └── Server B : 3268
```

## 3. SMB discovery

Your command: `smbclient -L //SHAIL.impressico.local -U 'impressico\kodidala.kumar'`
is different from LDAP and DNS.
Here you're talking to **SMB**, Microsoft's network file/remote-resource protocol.

```
Ubuntu
   │
   │ SMB
   ▼
SHAIL
```

### `smbclient`

`smbclient` is an Ubuntu/Linux command-line SMB client.
It lets you interact with Windows SMB servers.

#### `-L`

`-L` means: List the shares available on the SMB server.
So: `smbclient -L //SHAIL.impressico.local ...`
means: Ask SHAIL to list its SMB shares.

#### `//SHAIL.impressico.local`

This is the SMB server.
The syntax is: `//SERVER/SHARE`
But because you used: `//SHAIL.impressico.local`
without a share name, you're asking for the **server's share list**.

#### `-U`

`-U 'impressico\kodidala.kumar'` means: Authenticate as this user.
Your username is: `kodidala.kumar`
and the domain is: `IMPRESSICO`
So you're effectively saying:

```
Domain:   IMPRESSICO
User:     kodidala.kumar
```

Then SMB asks for your password.

### Your result

You got:

```
Sharename       Type      Comment
---------       ----      -------
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
SYSVOL          Disk      Logon server share
```

Let's decode that.

#### `IPC$`

IPC = **Inter-Process Communication**.
It's a special SMB share used for communication with Windows services.
It isn't a normal file-storage share.

```
IPC$
 ↓
SMB communication/infrastructure
```

#### `NETLOGON`

`NETLOGON` is a special domain-controller share used for domain-related operations, including logon-related scripts/files.

```
NETLOGON
   ↓
Domain infrastructure
```

#### `SYSVOL`

`SYSVOL` is another special domain-controller share.
It contains replicated domain policy/configuration files, including Group Policy-related files.

```
SYSVOL
   ↓
Domain policy/configuration files
```

So your SHAIL result:

```
SHAIL
 │
 └── SMB
      │
      ├── IPC$
      ├── NETLOGON
      └── SYSVOL
```

is exactly what you'd expect to see when querying a domain controller.
## Putting all three technologies together

This is the key section for your **"AD from Ubuntu Perspective"** notes.

### LDAP

LDAP answers: **What objects and AD structure exist?**

```
LDAP
 │
 ├── Users
 ├── Groups
 ├── Computers
 ├── OUs
 ├── Printers
 └── AD configuration
```

Example: `ldapsearch ...`

### DNS

DNS answers: **Where are the AD services and machines?**

```
DNS
 │
 ├── A records
 │      hostname → IP
 │
 ├── PTR records
 │      IP → hostname
 │
 └── SRV records
        service → server + port
```

Example: `dig @SHAIL.impressico.local _ldap._tcp.dc._msdcs.impressico.local SRV`
### SMB

SMB answers: **What network shares/resources does this SMB server expose to me?**

```
SMB
 │
 └── Server
       │
       ├── IPC$
       ├── NETLOGON
       ├── SYSVOL
       └── Other shares
```

Example: `smbclient -L //SHAIL.impressico.local -U 'impressico\kodidala.kumar'`
## The complete AD discovery workflow

This is the mental model I recommend putting at the top of your notes:

```
                    Ubuntu
                       │
                       │
               ┌───────┴────────┐
               │                │
             Kerberos          DNS
               │                │
          Authentication    Service discovery
               │                │
               └───────┬────────┘
                       │
                       ▼
                      LDAP
                       │
             "What is in AD?"
                       │
          ┌────────────┼────────────┐
          │            │            │
       Users       Computers     Printers
                       │
                       ▼
                  DNS hostname
                       │
                       ▼
                     IP
                       │
                       ▼
                      SMB
                       │
              "What shares exist?"
                       │
          ┌────────────┼────────────┐
          │            │            │
        Shares       SYSVOL       NETLOGON
                       │
                       ▼
                      DFS
```


