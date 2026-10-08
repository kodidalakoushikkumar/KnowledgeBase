# openssl

## First: the big picture

Suppose you own: `example.com`
You want NGINX to serve: `https://example.com`
You need an **SSL/TLS certificate**.
The basic flow is:

```text
                 You
                  │
                  │ Generate private key
                  ▼
            Private Key
                  │
                  │ Generate CSR
                  ▼
                CSR
                  │
                  │ Send to Certificate Authority
                  ▼
              Sectigo
                  │
                  │ Issues certificate
                  ▼
            SSL Certificate
                  │
                  │ Install on NGINX
                  ▼
          https://example.com
```

Now let's decode every term.

### 1. ECC

**ECC = Elliptic Curve Cryptography**
It's a type of cryptography used for creating **public/private key pairs**.
There are two common types you'll encounter:

```text
RSA
ECC
```

For example:

```text
RSA private key
RSA public key
```

or:

```text
ECC private key
ECC public key
```

ECC provides strong security with relatively small keys.
For example:

```text
RSA: 2048-bit
ECC: 256-bit
```

You should **not** think: ECC = certificate
Instead: ECC = cryptographic technology used by the key/certificate.

### 2. CSR

**CSR = Certificate Signing Request**
This is a request you create when you want a Certificate Authority to issue you a certificate.
For example:

```text
You
 │
 │ "I need a certificate for example.com"
 │
 ▼
CSR
 │
 ▼
Sectigo
```

A CSR contains information such as:

```text
Domain name
Organization
Country
Public key
Signature
```

Importantly:
**The private key is NOT sent to Sectigo.**
You keep the private key.

```text
Your server

Private Key
     │
     └── KEEP SECRET


CSR
     │
     └── Send to Sectigo
```

### 3. Sectigo

**Sectigo is a Certificate Authority (CA).**
A Certificate Authority is an organization trusted to issue SSL/TLS certificates.
Other examples include:

- Let's Encrypt
- DigiCert
- GlobalSign
- Sectigo

Think of Sectigo like a trusted authority saying: "I have verified that this certificate belongs to example.com."
So:

```text
You
 │
 │ CSR
 ▼
Sectigo
 │
 │ Certificate
 ▼
You
```

### 4. `.crt`

`.crt` generally means: Certificate file
For example: `example.crt`
It usually contains a certificate such as:

```text
-----BEGIN CERTIFICATE-----
MIID...
...
-----END CERTIFICATE-----
```

The certificate contains things like:

```text
Domain
Issuer
Validity period
Public key
Signature
```

A `.crt` file **does not normally contain your private key**.

### 5. `.pem`

This one causes a lot of confusion.
**PEM is a format/encoding, not necessarily a specific type of certificate.**
A PEM file is basically text that looks like:

```text
-----BEGIN CERTIFICATE-----
MIID...
...
-----END CERTIFICATE-----
```

You can have:

```text
certificate.pem
private-key.pem
chain.pem
```

So `.pem` doesn't tell you exactly what's inside.
For example: `certificate.pem`
could contain a certificate.
And: `private-key.pem`
could contain a private key.
The important part is: `-----BEGIN CERTIFICATE-----` or: `-----BEGIN PRIVATE KEY-----`

### 6. `.pfx`

`.pfx` is a **container**.
This is very different from `.crt`.
A PFX file can contain multiple things:

```text
                    .pfx
                     │
          ┌──────────┼──────────┐
          │          │          │
     Certificate  Private Key  Chain
```

For example:

```text
certificate
private key
intermediate certificates
```

That's why `.pfx` is convenient for moving an entire certificate setup between systems.
PFX is also commonly called: `PKCS#12` So: `.pfx = PKCS#12 container`
It is often password protected.

### 7. X.509

This is **not a file extension**.
X.509 is a standard that defines how certificates are structured.
Think:

```text
X.509
  │
  └── Rules for what an SSL certificate looks like
```

An X.509 certificate contains information such as:

```text
Subject
Issuer
Serial Number
Validity
Public Key
Extensions
Signature
```

So when someone says: "This is an X.509 certificate"
they're talking about the **certificate standard**, not necessarily a `.x509` file.

### 8. DER

DER is an **encoding format**.
There are two common ways you might encounter an X.509 certificate:

#### PEM

Human-readable text:

```text
-----BEGIN CERTIFICATE-----
MIID...
...
-----END CERTIFICATE-----
```

#### DER

Binary: `(binary data)`
So:

```text
X.509 certificate
       │
       ├── PEM encoded
       │
       └── DER encoded
```

You can think of it like:

```text
Same certificate
       │
       ├── PEM → text representation
       │
       └── DER → binary representation
```

### Put everything together

Here's the important relationship:

```text
                    SSL/TLS
                       │
              Certificate
                       │
                    X.509
                       │
              ┌────────┴────────┐
              │                 │
             PEM               DER
              │                 │
        certificate.pem    certificate.der
```

And separately:

```text
Cryptography
     │
     ├── RSA
     │
     └── ECC
```

Then:

```text
Private Key
     │
     └── used to create CSR
                    │
                    ▼
                   CSR
                    │
                    ▼
                 Sectigo
              (Certificate
               Authority)
                    │
                    ▼
              X.509 Certificate
                    │
             ┌──────┴──────┐
             │             │
            PEM           DER
```

And finally:

```text
Certificate + Private Key + Chain
              │
              ▼
             PFX
            (.pfx)
```


## openssl

`openssl` is a **command-line tool for working with SSL/TLS certificates, private keys, CSRs, encryption, and cryptographic operations**.

For example, if you're working with certificates like the **Sectigo `.pfx` / `.pem` / `.crt`** files, OpenSSL is commonly used to inspect and convert them.

### Basic OpenSSL commands

**1. Check OpenSSL version** `openssl version`
**2. View a certificate** `openssl x509 -in certificate.crt -text -noout`
This shows:

- Subject / domain
- Issuer
- Validity dates
- SANs
- Public key
- Signature algorithm

**3. Check a private key** `openssl rsa -in private.key -check`
For an ECC key: `openssl ec -in private.key -check`
**4. Create a CSR** `openssl req -new -key private.key -out certificate.csr`
**5. Convert PEM certificate to DER** `openssl x509 -in certificate.pem -outform DER -out certificate.der`
**6. Extract certificates/private key from a PFX** `openssl pkcs12 -in certificate.pfx -info`
Or extract the private key: `openssl pkcs12 -in certificate.pfx -nocerts -out private.key`
Extract the certificate: `openssl pkcs12 -in certificate.pfx -clcerts -nokeys -out certificate.crt`

### **Checking the SSL/TLS configuration of a website**.

The most important command is: `openssl s_client -connect example.com:443`
For a real website, for example: `openssl s_client -connect google.com:443`
This connects to the website's **HTTPS port 443** and performs a TLS handshake.

### What are you actually checking?

Think of it like:

```text
Your Linux machine
       │
       │ HTTPS / TLS
       ▼
   example.com
       │
       ├── Certificate
       ├── Certificate chain
       ├── TLS version
       ├── Cipher
       ├── Public key
       └── Server response
```

## Check a website's certificate

```bash
openssl s_client -connect example.com:443
```

You'll see information including:

```text
Certificate chain
Server certificate
SSL handshake
TLS version
Cipher
```

The output can be huge.
A very useful version is: `openssl s_client -connect example.com:443 -servername example.com`

#### Why `-servername`?

Because many websites share the same IP address.
For example:

```text
                    1 IP address
                         │
             ┌───────────┼───────────┐
             │           │           │
          site1.com   site2.com   site3.com
```

NGINX can use **SNI (Server Name Indication)** to determine which certificate to return.
So: `-servername example.com`
tells the server: "I want the TLS certificate for example.com."

### Show only the certificate

You can pipe the output to `openssl x509`: `openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -text -noout`

Now you can inspect:

```text
Subject
Issuer
Validity
Public Key
SAN
Signature Algorithm
```

For example:

```text
Issuer: Sectigo
Subject: example.com
Not Before: ...
Not After : ...
DNS:example.com
DNS:www.example.com
```

### Check certificate expiry

A much simpler command: `openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -dates`
You'll get something like:

```text
notBefore=Jan 10 00:00:00 2026 GMT
notAfter=Jan 10 23:59:59 2027 GMT
```

So you can quickly check: When does this website's certificate expire?

### Check who issued the certificate

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -issuer
```

Example: `issuer=C=GB, O=Sectigo Limited, CN=Sectigo ...`
This tells you the **Certificate Authority**.

### Check the domain on the certificate

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -subject
```

And: `openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -ext subjectAltName`
The second command is particularly useful because modern certificates use **SAN (Subject Alternative Name)**.
You might see:

```text
DNS:example.com
DNS:www.example.com
DNS:api.example.com
```

### Check TLS version

You can explicitly test TLS versions.
TLS 1.2 `openssl s_client -connect example.com:443 -tls1_2`
TLS 1.3 `openssl s_client -connect example.com:443 -tls1_3`
If the connection succeeds, the server supports that protocol version.

### Check the certificate chain

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

You may see:

```text
Certificate chain
 0 s:CN=example.com
   i:CN=Intermediate CA

 1 s:CN=Intermediate CA
   i:CN=Root CA
```

This is particularly important when you're troubleshooting things like:

```text
Sectigo
    ↓
Intermediate certificate
    ↓
Root certificate
```

If the chain is incorrectly configured on NGINX, some clients may report certificate errors.

### Check a website and see the negotiated TLS details

A useful command: `openssl s_client -connect example.com:443 -servername example.com </dev/null`
Look near the bottom for something like:

```text
Protocol  : TLSv1.3
Cipher    : TLS_AES_256_GCM_SHA384
```

That tells you: **Protocol** `TLSv1.3`
**Cipher suite** `TLS_AES_256_GCM_SHA384`

