
# 1️⃣ Y2K
### 1. Y2K (The Computer Bug) 💻

This is the original meaning.
Back in the 1960s–1990s, computers had very little storage. To save memory, years were stored using only **2 digits**.
Instead of: 1998 → `1998`
they stored:

- 1998 → `98`
- 1999 → `99`

When the year **2000** arrived, computers would store it as: 2000 → `00`
Many programs couldn't tell if `00` meant:

- 1900 ❌
- 2000 ✅

So calculations could fail. Example:

```
Birth Year = 99
Current Year = 00

Age = 00 - 99 = -99
```

Obviously wrong. People feared things like:

- Banks losing account data
- Airplanes having navigation issues
- Power grids failing
- Hospital systems crashing
- Government databases breaking

Companies spent **billions of dollars** fixing software before January 1, 2000.
### What actually happened?

Almost nothing dramatic. Not because the fear was fake—but because programmers spent years fixing the problem before midnight on January 1, 2000.

### Why is it called "Y2K"?

It literally stands for:

- **Y** = Year
- **2** = Two
- **K** = Kilo = 1000

So: **Y2K = Year 2000**

# 2️⃣ SSL
### 1. What is SSL?

### Definition

SSL (Secure Sockets Layer) is a cryptographic protocol designed to secure communication over the internet. It ensures that any data exchanged between a user’s browser and a web server is encrypted and safe from eavesdroppers or hackers.

### Purpose of SSL

- **Encryption**: Converts readable data into a secure, scrambled format.
- **Authentication**: Confirms the identity of the server.
- **Data Integrity**: Ensures data is not altered during transmission.

### Evolution

SSL was developed by **Netscape** in the mid-1990s.

- **SSL 1.0 (1995)**: Never released due to security flaws.
- **SSL 2.0 (1995)**: Introduced publicly but deprecated in 2011 due to vulnerabilities.
- **SSL 3.0 (1996)**: More secure but replaced by TLS (Transport Layer Security) in 1999.

 _Today, SSL is commonly referred to as_ **_TLS_** _(the successor of SSL), but the term “SSL” is still widely used._

### Example

Imagine sending a letter. SSL ensures the letter is placed in a locked envelope, and only the recipient has the key.

### 2. What is HTTPS?

### Definition

**HTTPS** (Hypertext Transfer Protocol Secure) is the secure version of HTTP, which is the protocol your browser uses to communicate with websites. By adding SSL/TLS, HTTPS ensures data exchanged between your browser and the website is encrypted.

### Why HTTPS Matters

- **Protects Privacy**: Prevents hackers from snooping on sensitive information (e.g., passwords, payment details).
- **Builds Trust**: Users see a padlock icon in the address bar, signaling the website is secure.
- **Improves SEO**: Search engines like Google prioritize HTTPS websites in search rankings.

### Timeline

- **1994**: Introduction of HTTP by Tim Berners-Lee.
- **1995**: HTTPS introduced by Netscape.
- **2014**: Google announces HTTPS as a ranking signal, encouraging adoption.

### Fun Fact

Before HTTPS became the standard, most websites transmitted data in plain text, making it easy for hackers to intercept.

### 3. What are SSL Certificates?

### Definition

An **SSL certificate** is a digital document that verifies the identity of a website and enables encrypted communication. It’s issued by a trusted entity called a **Certificate Authority (CA)**.

### Components of an SSL Certificate

- **Domain Name**: Specifies the domain the certificate is issued for.
- **Issuer Information**: Details about the CA.
- **Public Key**: Used for encryption.
- **Validity Period**: Expiration date of the certificate.

### Types of Certificates

1. **Domain Validation (DV)**: Verifies domain ownership. Ideal for small websites.
2. **Organization Validation (OV)**: Verifies the organization’s identity.
3. **Extended Validation (EV)**: Displays the company name in the browser. Used by banks and e-commerce sites.

### Certificate Authorities (CAs)

CAs are trusted organizations responsible for issuing SSL certificates. Examples include:

- Let’s Encrypt (free certificates, launched in 2016)
- DigiCert
- GoDaddy

### Fun Fact

The first SSL certificate was issued by Netscape in the mid-1990s to secure its browser-server communication.

### 4. How SSL, HTTPS, and Certificates Work Together

Here’s a step-by-step breakdown of what happens when you visit an HTTPS website:

### Step 1: Browser Connects to the Server

Your browser sends a request to a website (e.g., `https://example.com`) and asks to establish a secure connection.

### Step 2: Server Sends Its SSL Certificate

The server responds with its SSL certificate to prove its identity.

### Step 3: Browser Verifies the Certificate

Your browser checks if:

- The certificate is valid (not expired).
- It matches the domain name.
- It’s issued by a trusted CA.

### Step 4: Handshake and Encryption

If the certificate is valid, the browser and server perform a **TLS handshake**:

- They exchange public and private keys to establish a secure connection.
- Once verified, they use a **shared secret key** for faster communication.

### Step 5: Secure Communication Begins

All data exchanged between the browser and server is now encrypted.

### Example

When you enter your credit card details on an e-commerce site, HTTPS ensures the information is encrypted and cannot be intercepted.

### 5. Why SSL/TLS is Secure

### Encryption Types

1. **Asymmetric Encryption**: Uses a public/private key pair. Slower but secure.
2. **Symmetric Encryption**: Uses a single shared key for both encryption and decryption. Faster for bulk data.

### Challenges and Solutions

- **Expired Certificates**: Renew them periodically.
- **Untrusted Certificates**: Use a certificate from a trusted CA.
- **Man-in-the-Middle Attacks**: HTTPS prevents attackers from intercepting or altering data.


When you **manually install a certificate**, you are telling the browser/OS: “Trust this certificate even if the internet doesn’t know it.”
### Why This Is Needed

Normally browsers trust certificates only from big public authorities like:

- Let's Encrypt
- DigiCert

These are already trusted by browsers like Google Chrome and Mozilla Firefox.
But **internal company servers** usually use **private certificates**.
Example internal site: `https://internal-app.company.local`
That domain is **not public**, so public authorities won’t issue a certificate for it.
So companies create their **own certificate**.
### What Happens Without Installing It

If the browser sees that certificate, it says: “I don’t know who issued this certificate.”
Then you get errors like: Your connection is not private or ERR_CERT_AUTHORITY_INVALID

### What Manual Installation Does

When you manually install the certificate into the **trusted store**, your computer now trusts it.
So the browser thinks: Server certificate → issued by Company CA → trusted
Then the website opens normally.

# 3️⃣ Registers

### What is a Register in Processors/CPUs?

Registers are a type of computer memory built directly into the processor or CPU (Central Processing Unit) that is used to store and manipulate data during the execution of instructions. A register may hold an instruction, a storage address, or any kind of data.
A register is composed of multiple flip-flops, which are electronic circuits capable of storing a single bit of information, which is represented through binary data – either a 0 or a 1. By combining multiple flip-flops, registers can store larger binary values, such as bytes or words.
Registers also contain control logic circuitry, which allows it to coordinate the flow of data and instructions within the CPU. This can include operations such as decoding control signals, performing data manipulation like data loading, storing, or arithmetic operations, and using mulitplexers to route data to a specific location within the register.
Together, flip-flops and control logic work in partnership within registers. Flip-flops provide the storage capacity, while control logic facilitates the coordination of data transfer, manipulation, and synchronization with other components of the CPU. This enables registers to store and process data efficiently during the execution of instructions.
#### Sizes of CPU Registers
The number and size of registers in a CPU are determined by the processor design and can have a significant impact on its performance and capabilities. Most modern computer processers include:
- 8-bit registers: These registers can store 8 bits of data (1 byte). They are often used for basic arithmetic operations and data manipulation.
- 16-bit registers: These registers can store 16 bits of data (2 bytes). They are commonly found in older processors or in specific architectures that require 16-bit operations.
- 32-bit registers: These registers can store 32 bits of data (4 bytes). They are widely used in many processors and are capable of handling larger data sizes and more complex calculations.
- 64-bit registers: These registers can store 64 bits of data (8 bytes). They are prevalent in modern processors and offer increased computational power and memory addressing capabilities.
Modern PCs today most often have 32-bit or 64-bit registers and are referred to as the 32-bit processors and 64-bit processors we often hear about. This indicates the size or width of the processor's registers and the amount of data the processor can handle in a single operation.
In some specialized processors or architectures, you may also find larger register sizes, such as 128-bit, 256-bit, or even larger registers. These larger registers are often used for specific purposes like vector processing or cryptographic operations, where parallelism and large data sets are involved.
#### Types of CPU Registers
Depending on the CPUs architecture and design, the type and number of registers can vary. Common types of registers found in a CPU may include:
- **Program Counter (PC)**: The Program Counter keeps track of the memory address of the next instruction to be fetched and executed.
- **Instruction Register (IR)**: The Instruction Register holds the currently fetched instruction being executed.
- **Accumulator (ACC)**: The Accumulator is a general-purpose register used for arithmetic and logical operations. It stores intermediate results during calculations.
- **General-Purpose Registers (R0, R1, R2…)**: These registers are used to store data during calculations and data manipulation. They can be accessed and utilized by the programmer for various purposes.
- **Address Registers (AR)**: Address Registers store memory addresses for data access or for transferring data between different memory locations.
- **Stack Pointer (SP)**: The Stack Pointer points to the top of the stack, which is a region of memory used for temporary storage during function calls and other operations.
- **Data Registers (DR)**: These registers store data fetched from memory or obtained from input/output operations.
- **Status Register/Flags Register (SR)**: The Status Register or Flags Register contains individual bits that indicate the outcome of operations, such as carry, overflow, zero result, and others. These flags help in making decisions and controlling program flow based on the results of previous operations.
- **Control Registers (CR)**: Control Registers manage various control settings and parameters related to the CPU's operation, such as interrupt handling, memory management, and system configuration.
### How Do Registers Operate with Other CPU Components?
CPUs are made up of various components that when used together, allow it to process data and perform calculations. The major components include the Control Unit (CU), Arithmetic Logic Unit (ALU), Registers, Clock, Cache, and Buses.

![CPU-registers](../../Images/DOC/CPU-registers.png)

The ALU is a fundamental component of the CPU responsible for performing arithmetic and logical operations. It can execute operations like addition, subtraction, AND, OR, and others. The ALU takes input from registers, performs the desired operation, and stores the result back into a register.
The CU directs and coordinates the operations of various components within the CPU. It interprets instructions and generates control signals to manage the flow of data between registers, the ALU, memory, and input/output devices.
The interaction between registers, ALU, and CU can be summarized in the following steps:
2. The CU fetches an instruction from memory and places it into the instruction register.
3. The CU decodes the instruction to determine the operation to be performed and identifies the registers involved.
4. The CU issues control signals to select the appropriate registers and routes the data to the ALU.
5. The ALU performs the arithmetic or logical operation on the data from the selected registers.
6. The result of the operation is stored back into a register, based on the CU's control signals.
### Purpose of Registers
Registers are used by computers for various purposes, including storing program instructions before they're executed or holding intermediate results from calculations so that their values can be retrieved later on if needed. They also help speed up processes by allowing processors to access frequently used values without having to retrieve them from main memory every time they need them.
#### Registers in Embedded Systems
CPUs or other processors are used extensively as the “brain”, or the main processing component, in embedded systems. Embedded systems are self-contained computer systems within larger devices like cars or household appliances. Registers provide a quick and easy way to store data within these small computing devices and their low power consumption ensures that they don’t strain the device's energy budget.
In some cases, registers can even be used to create special register files that allow the processor to access multiple register addresses in one instruction cycle. This type of register file is particularly useful for applications such as digital signal processing (DSP) where speed is essential for a successful outcome. By having all necessary data stored inside registers and accessed quickly through register files, embedded systems can operate faster and more efficiently compared with other approaches.
### Total Phase Emulation and Debugging Tools
I2C, SPI, USB, and CAN are common communication protocols used to connect CPUs to peripheral devices in embedded systems. Total Phase offers several tools to allow embedded systems engineers to develop and/or debug their devices.
Through our I2C/SPI host adapters, users can easily emulate I2C or SPI master or slave devices to test the validity of the system as a whole. For instance, by using a master device, users can inject messages onto the bus in order to review communication and behavior between devices. It can also be used to emulate a master device to evaluate peripherals such as sensors and memory chips, and can also be used to emulate a slave device to test commands sent from MCUs.

# 4️⃣ ARM vs aarch64 vs AMD64 vs x86_64 vs x86

Following is a table that will provide you with a good summary of what each string means:

|**CPU Architecture**|**Description**|
|---|---|
|**`x86_64`**/`x86`/`amd64`|Same name for 64-bit AMD/Intel CPUs|
|**`AArch64`**/`arm64`/`ARMv8`/`ARMv9`|Same name for 64-bit ARM CPUs|
|**`i386`**|32-bit AMD/Intel CPUs|
|**`AArch32`**/`arm`/`ARMv1` to `ARMv7`|Same name for 32-bit ARM CPUs|
|**`rv64gc`**/`rv64g`|Same name for 64-bit RISC-V CPUs|
|**`ppc64le`**|64-bit PowerPC CPUs  <br>with **little-endian memory ordering**|

Reading from left to right is the preference of using that term to describe the CPU architecture over the other, alternatively used terms on its right.

If you are nerdy like me and want a more in-depth explanation, do read on!

## General overview: CPU architectures

The terms that I listed above, generally speaking, are CPU architectures. Though, pedantically speaking, these are what a computer engineer calls a CPU ISA (Instruction Set Architecture).

A CPU ISA is what defines how the 1's and 0's of binary are interpreted by your CPU.

There are a few supersets of these CPU ISAs.

- x86 (AMD/Intel)
- ARM
- RISC-V
- PowerPC (still alive at IBM)

There are more CPU ISAs like MIPS, SPARC, DEC Alpha, etc. But the ones I listed above are the ones that are still widely used today (in some capacity).

The above listed ISAs have at least two subsets. This is mainly based on the _width of the memory bus_. The width of the memory bus denotes how many bits can be transferred between the CPU and the RAM in one go. There are several widths for the memory bus, but the two most important widths are 32-bit wide memory bus and a 64-bit wide memory bus.

The 32-bit counterparts of the CPU ISAs are either a relic of the past, kept alive for legacy support or are only used in micro-controllers. It is safe to assume that ****any new hardware is 64-bit**** (especially consumer facing hardware).

## x86 (AMD/Intel)

The x86 CPU ISA comes primarily from Intel as Intel was the one who created it in the first place with the 8085 micro-processor. The 8085 micro-processor had a 16-bit wide memory bus. Later, AMD came to the game and followed Intel's footsteps until AMD created their own superset 64-bit architecture, surpassing Intel.

The subsets of x86 architecture are as follows:

- `i386`: If you own a CPU from pre-2007, this is likely your CPU architecture. It is the 32-bit "variant" of the currently known x86 architecture from AMD/Intel.
- `x86_64`/`x86`/`amd64`: All three terms are used interchangibly depending on the project you look at. But they all refer to the 64-bit "variant" of the x86 AMD/Intel architecture. Regardless, the string `x86_64` is widely used (and preferred) over `x86` and `amd64`. An example of this is that the FreeBSD project refers to the 64-bit x86 architecture as `amd64` while Linux and macOS refer to this as `x86_64`.

Since AMD beat Intel at creating a 64-bit ISA, some projects like FreeBSD refer to the 64-bit variant of x86 as amd64. ****But the more widely accepted term is still**** ****x86_64********.****

The `x86` string for CPU ISA is a special one. You see, during the transition from 32-bit x86 (`i386`) to 64-bit x86 (`x86_64`), the CPU vendors made sure that the CPU can run both, 32-bit _and_ 64-bit instructions. Therefore, sometimes when you read `x86`, it can also mean "It will run only on a 64-bit computer, but if that computer can run 32-bit instructions, you can run 32-bit user software on it."

This ambuiguity of x86--meaning 64-bit processors that can also run 32-bit code--is mainly for/due-to Operating Systems that run on 64-bit processors, but allow the user of said OS to run 32-bit software. Windows makes use of this with a feature called "compatibility mode".

Let's recap, there are two CPU architectures for the CPUs designed by AMD and Intel. They are 32-bit (`i386`) and 64-bit (`x86_84`).

### Extra _intel_

_(Yeah! I am funny)_

The `x86_64` ISA also has sub-sets. All of these subsets are 64-bit but have various features added. Especially SIMD (Single Instruction Multiple Data) instructions.

- `x86_64-v1`: The base `x86_64` ISA that almost everyone is familar with. When someone says `x86_64`, they are most likely referring to the `x86_64-v1` ISA.
- `x86_64-v2`: This adds more instructions like SSE3 (Streaming SIMD Extensions 3) as extensions.
- `x86_64-v3`: Adds instructions like AVX (Advance Vector eXtensions) and AVX2 which can use **up-to 256-bit wide CPU registers**! This can massively parallelize your computations if you can take advantage.
- `x86_64-v4`: Iterates upon the `x86_64-v3` ISA by adding more SIMD instruction as extensions. Such as AVX256 and AVX512. The later can use **up-to 512-bit wide CPU registers**!

## ARM

ARM is a company that creates its own specification for a CPU ISA, designs and licenses their own CPU cores and also allows other companies to design their own CPU cores using the ARM CPU ISA. (The last part felt like an SQL query!)

You might have heard about ARM because of SBCs (Single Board Computer) like the Raspberry Pi line up of SBCs. But their CPUs are also widely used in mobile phones. Recently, Apple has switched from `x86_64` processors to using their own design of ARM processors in their laptop and desktop offerings.

Like any CPU architecture, there are two subsets based on the width of the memory bus.

The officially recognised names for the 32-bit and 64-bit ARM architectures are `AArch32` and `AArch64` respectively. The 'AArch' string stands for 'Arm Architecture'. These are **modes** a CPU can be in, for executing instructions.

The actual specification of an instruction that complies with ARM's CPU ISA are named `ARMv_X_` where _`X`_ refers to a generation number of a specification. To this date, there have been 9 major versions of this specification. Ranging from `ARMv1` to `ARMv7`, which is defines a CPU architecture specification for 32-bit CPUs. While `ARMv8` and `ARMv9` are specifications for the 64-bit ARM CPUs.

Each ARM CPU specification has further sub-specifications. Taking ARMv8 as an example, we have ARMv8-R, ARMv8-A, ARMv8.1-A, ARMv8.2-A, ARMv8.3-A, ARMv8.4-A, ARMv8.5-A, ARMv8.6-A, ARMv8.7-A, ARMv8.8-A and ARMv8.9-A. The -A stands for "Applications cores" and -R stands for "Real-time cores".

You might be wondering why some people call it `arm64` even when `AArch64` is the officially recognized name for 64-bit ARM architecture. The reason is two-folds:

1. The name `arm64` caught on before `AArch64` was decided upon by ARM. (ARM also refers to the 64-bit ARM architecture as `arm64` in some of it's official documentation... 😬)
2. Linus Torvalds dislikes the `AArch64` name. Therefore the Linux codebase largely refers to `AArch64` as `arm64`. But it will still report `aarch64` when you do a `uname -m`.

Therefore, for 32-bit ARM CPUs, you should look for the string `AArch32` but sometimes it might also be `arm` or `armv7`. Similarly, for 64-bit ARM CPUs, you should look for the string `AArch64` but sometimes it might also be `arm64` or `ARMv8` or `ARMv9`.

## RISC-V

RISC-V is an open source specification of a CPU ISA. **This doesn't mean that the CPUs themselves are open source!** It is a standard, kind of like Ethernet. The Ethernet specification is open source but the cables, routers and switches you purchase do cost money. Same deal with RISC-V CPUs. :)

Though, this has not prevented people from creating RISC-V cores that are freely available (**as designs**; not as physical cores/SoC) under an open source license.

TL;DR: You sould be looking for the string ****rv64gc**** if you are looking for software to run on RISC-V consumer CPUs. This is what a large number of Linux distributions have agreed upon.

Just like any CPU architecture, RISC-V has 32-bit and 64-bit CPU architectures. Since RISC-V is _very new_ (in the terms of a CPU ISA), all major CPU cores in consumer/client side are usually 64-bit CPUs. The 32-bit designs are mostly micro-controllers that have a very specific use-case.

What they do differ in, are CPU extensions. The absolute minimum extension one needs to implement to be called a RISC-V CPU is the 'Base Integer Instruction Set' (`rv64i`).

A table of a few extensions and the description is as below:

|**Extension name**|**Description**|
|---|---|
|`rv64i`|64-bit Base Integer Instruction Set (**mandatory**)|
|`m`|Multiplication and Division instructions|
|`a`|Atomic instructions|
|`f`|Single-precision floating point instructions|
|`d`|Double-precision floating point instructions|
|`g`|Alias; A collection of extensions necessary to run a **g**eneral-purpose OS (includes `imafd`)|
|`c`|Compressed instructions|

In the string `rv64i`, `rv` stands for RISC-V, `64` denotes that this is a 64-bit CPU architecture and `i` is the extension for the **mandatory** base integer instruction set. The reason why `rv64i` is written together is because, even though the `i` extension is an "extension", **it is mandatory**.

The convention is to have the extension name in the specific order listed as above. So `rv64g` expands to `rv64imafd`, not to `rv64adfim`.

There are other extensions like Zicsr and Zifencei that sit between the d and g extensions but I deliberately didn't include them as to not scare you off.  

# 5️⃣ How Docker use HOST Kernel
## When you install Docker on Ubuntu

You install:

```
Ubuntu Host
	├── Linux Kernel
	├── Docker Engine
	└── Docker CLI
```

Docker Engine **uses the host Linux kernel**.
There is **no separate kernel inside containers**.
## What happens when you run nginx?

```
docker run nginx
```

Docker downloads the nginx image.
The image contains a filesystem:

```
nginx image
├── /bin
├── /lib
├── /etc
├── /usr
└── nginx files
```

Then Docker starts a container using:

```
Container
├── Own filesystem
├── Own processes
├── Own network
└── Host Kernel
```

Notice: `Container Kernel = Host Kernel`
There is only one kernel.

## Visual

```
Host Ubuntu
│
├── Linux Kernel
	│
	├── Docker Engine
	│
	└── Container (nginx)
		    ├── /bin
			├── /lib
            ├── /etc
            └── nginx
```

The container gets its own `/bin`, `/lib`, `/etc`.
But when a process inside the container needs kernel functionality: `open() read() write() fork()`
those system calls go directly to the **host kernel**.

## Example

Host: `ls /`
Output: `binbootetchomelibusr...`
Now run: `docker run -it ubuntu bash`
Inside container: `ls /`
Output: `binetclibusr...`
Looks like another Linux machine.
But it isn't.
The container filesystem came from the Ubuntu image.
The kernel is still your host kernel.
## Compare with Virtual Machine

### Virtual Machine

```
Host Hardware
│
├── Hypervisor
	│
	├── VM 1
	│   ├── Kernel
	│   └── Filesystem
	│
	└── VM 2    
		├── Kernel    
		└── Filesystem
```

Each VM has its own kernel.
### Docker

```
Host Hardware│
├── Linux Kernel
	│
	├── Container 1
	│   └── Filesystem
	│
	├── Container 2
	│ 	└── Filesystem
	│
	└── Container 3    
		└── Filesystem
```

All containers share the same kernel.
That's why containers start in seconds and use much less memory.

## Then where do `/bin`, `/lib`, `/etc` come from?

From the image.
Example: `docker run ubuntu`
Docker pulls the Ubuntu image.
That image already contains:

```
Ubuntu Image
├── /bin
├── /lib
├── /etc
├── /usr
└── ...
```

When the container starts, Docker mounts this filesystem for the container.
So:

```
Host /bin        ≠ Container /bin
Host /etc        ≠ Container /etc
Host /usr        ≠ Container /usr
```

But: `Host Kernel == Container Kernel`

## Quick Proof

Host: `uname -r`
Suppose: `6.8.0-58-generic`
Now inside a container: `docker run -it ubuntu bashuname -r`
You'll see: `6.8.0-58-generic`
Same kernel version.
Why?
Because both are using the same kernel.
## One sentence summary

A Docker container gets **its own filesystem (/bin, /lib, /etc, /usr, etc.) from the image**, but **it does not get its own kernel**—all containers share and use the host Linux kernel through Docker Engine.