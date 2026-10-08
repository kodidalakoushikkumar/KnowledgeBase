---
Title:
Type:
Topic:
tags:
ee: "[[/IMAGES/BUS.png]]"
---
Yes. The missing piece is the **actual hands-on flow**. Think of it as:

```text
                 YOUR SERVER
                     │
                     │ 1. Generate private key
                     ▼
              server.key
                     │
                     │ 2. Generate CSR
                     ▼
              server.csr
                     │
                     │ 3. Send CSR to CA
                     ▼
              ┌──────────────┐
              │ Certificate  │
              │ Authority CA │
              └──────────────┘
                     │
                     │ 4. CA verifies domain
                     ▼
              server.crt
              intermediate.crt
                     │
                     │ 5. Install on Web Server
                     ▼
              HTTPS :443
```

The important thing is: **you generate and keep the private key. You send the CSR to the CA, not the private key.** A CSR contains your public key and identifying information; the CA signs it and gives you the certificate.

I'll show you the complete process using **Linux + OpenSSL + Nginx**, because this makes the concepts very clear.

### 1. First understand the files

Suppose your website is: `www.example.com`
You will eventually have something like:

```text
server.key
server.csr
server.crt
intermediate.crt
```

Their purposes:

|File|What it is|Who gets it?|
|---|---|---|
|`server.key`|Private key|**Only your server**|
|`server.csr`|Certificate Signing Request|Send to CA|
|`server.crt`|Your CA-signed certificate|Install on server|
|`intermediate.crt`|CA intermediate certificate|Install/configure on server|
|CA Root|Root CA certificate|Usually already trusted by clients|

**Never send `server.key` to the CA.**

### 2. Generate the private key

On your Linux web server: `openssl genrsa -out server.key 2048`
This creates: `server.key`
This is your **RSA private key**.
Check it: `openssl rsa -in server.key -check`
You should get something similar to: `RSA key ok`
Protect it: `chmod 600 server.key`
The private key should remain secret and have restricted permissions. NGINX also needs to be able to read it. ([Nginx Documentation](https://docs.nginx.com/nginx/admin-guide/security-controls/terminating-ssl-http/?utm_source=chatgpt.com "NGINX SSL Termination | NGINX Documentation"))

### 3. Where is the public key?

This is something that often confuses beginners.
You don't normally create a separate: `public.key`
file for this workflow.
The public key is mathematically derived from: `server.key`
You can extract it if you want: `openssl rsa -in server.key -pubout -out server-public.key`
Now:

```text
server.key
      │
      └──────► server-public.key
```

But for certificate creation, you normally don't need to manually send this public-key file.

### 4. Generate the CSR

Now create the Certificate Signing Request: `openssl req -new -key server.key -out server.csr`
OpenSSL will ask questions such as:

```text
Country Name:
State or Province Name:
Locality Name:
Organization Name:
Organizational Unit Name:
Common Name:
Email Address:
```

The important one historically was: `Common Name (CN):`
For example: `www.example.com`
But modern certificates primarily use **Subject Alternative Name (SAN)** for the hostname.
So I recommend generating the CSR with SAN explicitly.

### 5. Generate CSR with SAN

For example, suppose you want:

```text
example.com
www.example.com
```

Run:

```bash
openssl req -new \
  -key server.key \
  -out server.csr \
  -subj "/C=IN/ST=Telangana/L=Hyderabad/O=Example Corp/OU=IT/CN=example.com" \
  -addext "subjectAltName=DNS:example.com,DNS:www.example.com"
```

Now you have:

```text
server.key
server.csr
```

OpenSSL supports creating PKCS#10 CSRs with `openssl req`, including adding extensions such as SAN.

### 6. Look inside the CSR

Before sending it to the CA, check it: `openssl req -in server.csr -text -noout`
You'll see something like:

```text
Certificate Request:
    Subject:
        C=IN
        ST=Telangana
        L=Hyderabad
        O=Example Corp
        CN=example.com

    Subject Public Key Info:
        Public Key Algorithm: rsaEncryption
        RSA Public-Key: 2048 bit

    Requested Extensions:
        X509v3 Subject Alternative Name:
            DNS:example.com
            DNS:www.example.com
```

This is a very useful command for troubleshooting.

### 7. Now send the CSR to the CA

This is where the CA comes in.
For example, you might use a public CA such as:

- Let's Encrypt
- DigiCert
- GlobalSign
- Sectigo
- Entrust

Or your company may have an **internal CA**, such as Microsoft AD CS.
You give the CA: `server.csr`
You **do not give them**: `server.key`
The process is:

```text
                    YOUR SERVER

server.key
    │
    │
    ├───────────────┐
    │               │
    ▼               ▼
public key        CSR
                    │
                    │
                    ▼
              ┌───────────┐
              │    CA     │
              └───────────┘
                    │
                    │ verifies
                    │ ownership/identity
                    ▼
                server.crt
```

The CA verifies that you're authorized to obtain a certificate for the domain. For a public CA, this might involve DNS, HTTP, or organizational validation depending on the certificate.

### 8. What does the CA actually do?

This is the cryptographic heart of the process.
You generated:

```text
Private Key
     │
     ▼
Public Key
```

Your CSR contains: `Your identity information + Your public key + CSR signature`
The CA verifies the CSR and domain/organization validation.
Then the CA creates a certificate containing information such as:

```text
Subject:
    example.com

Public Key:
    <your public key>

Issuer:
    Example CA

Validity:
    2026 → 2027

SAN:
    example.com
    www.example.com

Signature:
    <CA's digital signature>
```

The CA signs that certificate using the CA's private key.
That's why browsers can trust it.

### 9. CA gives you the certificate

You might receive: `example.com.crt`
and perhaps: `intermediate.crt`
You might also receive: `fullchain.pem`
So your server may now have:

```text
server.key
server.csr
example.com.crt
intermediate.crt
```

Remember:

```text
server.key       ← YOU generated this
server.csr       ← YOU generated this
example.com.crt  ← CA generated/signed this
```

### 10. Why do we need the intermediate certificate?

This is another important concept.
The browser generally trusts a **Root CA**.
But your website certificate may have been signed by an **Intermediate CA**.
So you get a chain:

```text
Root CA
   │
   │ signs
   ▼
Intermediate CA
   │
   │ signs
   ▼
Your Server Certificate
```

For example:

```text
Root CA
   ↓
Intermediate CA
   ↓
example.com certificate
```

Your server normally sends the server certificate plus the intermediate certificate(s), allowing the client to build the chain to a trusted root.

NGINX supports putting the server certificate followed by intermediate certificates in the certificate file. ([Nginx](https://nginx.org/en/docs/http/ngx_http_ssl_module.html?utm_source=chatgpt.com "Module ngx_http_ssl_module"))

### 11. Create the certificate chain

Suppose you received:

```text
example.com.crt
intermediate.crt
```

You can create: `cat example.com.crt intermediate.crt > fullchain.crt`
Now: `fullchain.crt`
contains:

```text
-----BEGIN CERTIFICATE-----
Your server certificate
-----END CERTIFICATE-----

-----BEGIN CERTIFICATE-----
Intermediate CA certificate
-----END CERTIFICATE-----
```

The order matters:

```text
YOUR CERTIFICATE
       ↓
INTERMEDIATE
       ↓
ROOT
```

You normally don't need to include the root certificate in the server's served chain.

### 12. Install the files on NGINX

For example: `sudo mkdir -p /etc/nginx/ssl`
Copy:

```bash
sudo cp server.key /etc/nginx/ssl/
sudo cp fullchain.crt /etc/nginx/ssl/
```

Set permissions: `sudo chmod 600 /etc/nginx/ssl/server.key`
You now have:

```text
/etc/nginx/ssl/
├── server.key
└── fullchain.crt
```

### 13. Configure NGINX

Suppose your website configuration is: `/etc/nginx/sites-available/example`
You would configure something like:

```nginx
server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate /etc/nginx/ssl/fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    root /var/www/example;
    index index.html;
}
```

NGINX's HTTPS configuration uses `ssl_certificate` for the certificate and `ssl_certificate_key` for the private key.

### 14. Enable the website

For Ubuntu/Debian:

```bash
sudo ln -s /etc/nginx/sites-available/example \
           /etc/nginx/sites-enabled/example
```

Test configuration: `sudo nginx -t`
You want:

```text
syntax is ok
test is successful
```

Then reload: `sudo systemctl reload nginx`
Now: `https://example.com`
should use your CA-issued certificate.

### 15. The entire process as commands

If you're learning this for IT support/sysadmin work, memorize this workflow:

#### Step 1 — Generate private key

```bash
openssl genrsa -out server.key 2048
```

#### Step 2 — Generate CSR

```bash
openssl req -new \
  -key server.key \
  -out server.csr
```

#### Step 3 — Inspect CSR

```bash
openssl req -in server.csr -text -noout
```

#### Step 4 — Send CSR to CA

```text
server.csr
       ↓
      CA
       ↓
server.crt
```

#### Step 5 — Receive certificate

```text
server.crt
intermediate.crt
```

#### Step 6 — Create chain

```bash
cat server.crt intermediate.crt > fullchain.crt
```

#### Step 7 — Install

```bash
sudo cp server.key /etc/nginx/ssl/
sudo cp fullchain.crt /etc/nginx/ssl/
sudo chmod 600 /etc/nginx/ssl/server.key
```

#### Step 8 — Configure NGINX

```nginx
ssl_certificate /etc/nginx/ssl/fullchain.crt;
ssl_certificate_key /etc/nginx/ssl/server.key;
```

#### Step 9 — Test

```bash
sudo nginx -t
```

#### Step 10 — Reload

```bash
sudo systemctl reload nginx
```

### 16. RSA vs ECC in this process

You asked about RSA and ECC earlier.
The workflow is almost identical.

#### RSA

```bash
openssl genrsa -out server.key 2048
```

Then: `openssl req -new -key server.key -out server.csr`

#### ECC/ECDSA

Generate an EC key: `openssl ecparam -genkey -name prime256v1 -out server.key`
Then: `openssl req -new -key server.key -out server.csr`
So:

```text
                 RSA
                  │
             RSA private key
                  │
                  ▼
                 CSR
                  │
                  ▼
                  CA
```

or:

```text
                 ECC
                  │
            EC private key
                  │
                  ▼
                 CSR
                  │
                  ▼
                  CA
```

The resulting certificate can be used by the web server in the same general way. NGINX supports both RSA and ECDSA certificate/key configurations.

### 17. One very important distinction: certificate ≠ private key

Imagine this: `server.key`
contains your private key.
Someone steals it.
They may be able to impersonate your server, depending on the situation.
Therefore:

```text
NEVER:
email server.key
upload server.key to CA
put server.key in GitHub
send server.key over WhatsApp
```

But this is normally okay to distribute: ``
because the certificate is public information.
NGINX explicitly treats the certificate as public while the private key must be protected.

### 18. What happens when a user visits your website?

This is the part that ties everything together.
User: `https://example.com`
connects to: `NGINX :443`
NGINX sends:

```text
server certificate
+
intermediate certificate
```

The browser checks:

```text
Is certificate expired?
        ↓
Does hostname match?
        ↓
Is certificate chain valid?
        ↓
Do I trust the Root CA?
        ↓
Does the server prove it possesses
the corresponding private key?
        ↓
YES
        ↓
TLS connection established
        ↓
HTTPS
```

The certificate contains the server's public key and CA signature, while the private key remains on the server. ([Ubuntu](https://ubuntu.com/server/docs/explanation/security/certificates/?utm_source=chatgpt.com "About certificates - Ubuntu Server documentation"))

#### The one picture I'd remember

```text
                 YOU / SERVER
                      │
                      │
              Generate private key
                      │
                      ▼
                 server.key
                      │
                      │
              Generate CSR
                      │
                      ▼
                 server.csr
                      │
                      │
                      │ SEND ONLY CSR
                      ▼
              ┌───────────────┐
              │      CA       │
              │ DigiCert etc. │
              └───────┬───────┘
                      │
             CA verifies identity
                      │
                      ▼
                server.crt
                      │
                      │ + intermediate.crt
                      ▼
              ┌───────────────┐
              │    NGINX      │
              │     :443      │
              └───────┬───────┘
                      │
                      │ HTTPS
                      ▼
                   Browser


PRIVATE KEY:
server.key
     │
     └──── NEVER leaves your server


CERTIFICATE:
server.crt
     │
     └──── Public / sent to clients
```

