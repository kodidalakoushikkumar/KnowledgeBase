### WHOLE TREE

```
WINDOWS LICENSES
│
├── 1. By acquisition / sales channel
│   ├── Retail / FPP
│   ├── OEM
│   ├── Volume Licensing
│   ├── Subscription / Cloud licensing
│   └── Special / Academic / Developer / Evaluation channels
│
├── 2. By activation method
│   ├── Digital License
│   ├── Product Key
│   ├── MAK
│   ├── KMS
│   └── Active Directory-Based Activation
│
├── 3. By license rights / usage
│   ├── Perpetual
│   ├── Subscription
│   ├── Upgrade
│   └── Full license
│
└── 4. By edition
    ├── Windows Home
    ├── Pro
    ├── Pro for Workstations
    ├── Enterprise
    ├── Education
    └── Server editions
```


#### 1. FPP — Full Packaged Product

Historically, this was the boxed Windows product you bought from a shop.
People may casually call this a **"paper license"** because it could include physical documentation or a printed key, but **paper is not an official top-level Windows license category**.

#### 2. Digital Retail

A retail license generally has more transfer flexibility than an OEM license, subject to Microsoft's license terms.

### B. OEM License

**OEM = Original Equipment Manufacturer**
This is the license that comes with a new laptop or PC.
Microsoft describes modern OEM activation as associating Windows with the computer's firmware/BIOS.

#### C. Volume Licensing

#### Volume Licensing is the **license acquisition category**

But MAK, KMS, and AD-Based Activation are primarily **activation methods under volume activation**.
For supported Windows Pro and Enterprise volume deployments, Microsoft documents MAK, KMS, and Active Directory-based activation as the main volume activation models.

##### C1. MAK — Multiple Activation Key

Example: Company buys licenses for 100 PCs
A MAK has a predetermined number of permitted activations. Microsoft describes it as essentially similar to a retail key, except that it can activate multiple computers.

##### Best for

- Small organizations
- Remote PCs
- Isolated PCs
- Organizations that don't want to maintain KMS infrastructure

##### C2. KMS — Key Management Service

Instead of every PC activating directly with Microsoft:
The KMS host manages activation for computers inside the organization.
Microsoft specifies activation thresholds, including **25 Windows client computers** for KMS activation of Windows clients.
##### Best for

- Medium/large companies
- Domain environments
- Large numbers of devices
- Centralized activation

#### C3. Active Directory-Based Activation

A domain-joined PC can activate through the organization's Active Directory activation configuration.
Microsoft documents this as another volume activation model alongside MAK and KMS.
##### Best for

- Organizations already using Active Directory
- Domain-joined Windows PCs
- Environments where you don't want a separate KMS activation process for clients

#### C4. GVLK — Generic Volume License Key

This is another thing people confuse with MAK/KMS.
A GVLK does **not** activate directly against Microsoft's activation servers by itself.

### 1 By activation method

```
ACTIVATION METHODS
│
├── Product Key Activation
│   ├── Retail key
│   ├── OEM key
│   └── MAK
│
├── Firmware/Digital OEM Activation
│   └── Key embedded in UEFI
│
├── KMS Activation
│   └── Company KMS host
│
├── Active Directory-Based Activation
│   └── AD domain
│
└── Subscription Activation
    └── Microsoft Entra / assigned subscription
```

### By license rights / usage

```
LICENSE DURATION
│
├── Perpetual
│   └── License does not require recurring subscription payments
│
└── Subscription
    ├── Monthly
    └── Annual / term-based
```


Modern Microsoft licensing can also work through subscriptions.
Microsoft calls this **subscription activation**. A licensed Windows Pro device can step up to Enterprise based on the user's assigned E3/E5 license and identity configuration.
When the subscription is removed or expires, the device can revert to its underlying Pro/Pro Education edition, subject to the applicable subscription activation conditions.

License Duration
Examples:

| License                               | General model                                          |
| ------------------------------------- | ------------------------------------------------------ |
| Retail                                | Usually perpetual                                      |
| OEM                                   | Usually perpetual for that licensed device             |
| Traditional Volume                    | Can include perpetual or agreement/subscription models |
| Windows Enterprise E3/E5 subscription | Subscription                                           |

License Scope
For example, Windows subscription activation for Enterprise is based on **per-user licensing**, not per-device licensing.