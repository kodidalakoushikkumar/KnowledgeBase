# 1️⃣2️⃣ KDC TGT RADIUS

- **Kerberos** is the **authentication protocol/system** that defines how users prove their identity securely on a network.
- **KDC (Key Distribution Center)** is a **component inside Active Directory** that acts like the “ticket office.” It **issues tickets** needed for authentication.
- **TGT (Ticket-Granting Ticket)** is the **first ticket** the KDC gives you when you log in successfully. It proves to the network that you are a valid user.
- After getting a TGT, the **Kerberos protocol** uses it to get **service tickets** for internal resources without asking for your password again.

**Flow in simple terms (your network):**

```
User enters AD credentials 
	↓ 
Kerberos protocol triggers
	↓   
KDC (AD server) verifies credentials 
	↓
KDC issues TGT 
	↓   
Device uses TGT to request service tickets 
	↓
Access internal resources
```

So yes, **KDC provides TGT**, and **Kerberos is the system/protocol that uses it** for authentication.

## Is there any protocol other than Kerboros For auth?

Yes! Kerberos is the default authentication protocol in **Windows/Active Directory environments**, but there are several **other authentication protocols** used in networks depending on the system, use case, or vendor. Here’s a detailed breakdown:

### **1. RADIUS (Remote Authentication Dial-In User Service)**

- **Purpose:** Centralized authentication for network devices (Wi-Fi, VPN, switches).
- Often used with **802.1X / PEAP** for Wi-Fi authentication.
- Works with AD to verify credentials.
- **Flow:** Device → Network Device (AP/Firewall) → RADIUS server → AD

### **2. LDAP / LDAPS (Lightweight Directory Access Protocol / Secure)**

- **Purpose:** Access directory services for authentication and user info.
- AD supports LDAP for applications that need to verify credentials.
- **LDAPS** adds encryption for security.

### **3. NTLM (NT LAN Manager)**

- **Purpose:** Legacy Windows authentication protocol.
- Used when Kerberos is unavailable (like across non-AD domains or certain legacy applications).
- Less secure than Kerberos because it doesn’t use tickets; instead, it relies on challenge/response hashing.

### **4. SAML (Security Assertion Markup Language)**

- **Purpose:** Web-based Single Sign-On (SSO) for cloud or internal applications.
- Often used with **Okta, Azure AD, or Google Workspace**.
- Allows users to authenticate once and access multiple web apps.

### **5. OAuth / OpenID Connect**

- **Purpose:** Modern token-based authentication for web and mobile apps.
- Common in cloud services (Google, Microsoft 365, GitLab SaaS).

### **6. EAP Variants (for Wi-Fi / VPN)**

- **PEAP (Protected EAP)** – encapsulates authentication in a TLS tunnel; works with AD + Kerberos or LDAP.
- **EAP-TLS** – certificate-based authentication without passwords.
- **EAP-TTLS** – similar to PEAP, supports various inner authentication methods.

### **Summary Table**

| Protocol               | Use Case          | Notes                               |
| ---------------------- | ----------------- | ----------------------------------- |
| Kerberos               | AD, LAN, SSO      | Ticket-based, secure, default in AD |
| NTLM                   | Legacy Windows    | Challenge-response, less secure     |
| RADIUS                 | Wi-Fi, VPN        | Centralized auth, works with AD     |
| LDAP / LDAPS           | Directory lookups | Can authenticate apps via AD        |
| SAML                   | Web apps / SSO    | Token-based, cloud-focused          |
| OAuth / OpenID Connect | Web / mobile apps | Modern token-based auth             |
| EAP variants           | Wi-Fi/VPN         | PEAP, EAP-TLS, EAP-TTLS             |


RADIUS is middle man btw AD and user for passing cred and used by aruba(which is RADIUS CLIENT or Server)  if user connected to AP.