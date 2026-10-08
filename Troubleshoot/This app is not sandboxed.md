# 3️⃣8️⃣ This app is not sandboxed

In Ubuntu, when you hear **“this app is not sandboxed”**, it means the application is **not isolated from the rest of your system**.
Think of it like this:

### 1. Normal app — not sandboxed

Suppose you install a program normally:

```
Ubuntu
│
├── Your files
├── System files
├── Network
├── Hardware
└── Application
       └── can potentially access many of these
```

The application runs as your user account, so it generally has the same permissions that **you** have.
For example, if you are logged in as `koushik`, an application might be able to access:

```
/home/koushik/Documents
/home/koushik/Downloads
/home/koushik/.config
```

It doesn't automatically have permission to modify protected system files such as:

```
/etc
/usr
/boot
```

but it can access a lot of your user-level files.
### 2. Sandboxed app

A sandbox puts the application inside a restricted environment:

```
Ubuntu
│
├── Your files
│
├── System
│
└── Sandbox
     └── Application
          ├── Limited files
          ├── Limited hardware
          └── Limited system access
```

The application may be prevented from accessing things it doesn't need. For example:

```
Application
     │
     ├── ✓ its own files
     ├── ✓ allowed folders
     ├── ✓ allowed network access
     │
     ├── ✗ your private documents
     ├── ✗ arbitrary system files
     └── ✗ certain hardware
```

This is mainly a **security mechanism**.
### Where you'll see this in Ubuntu

You may encounter this when installing applications as **Snap** or **Flatpak**.
For example: `snap install firefox`
Snap applications normally run with confinement/sandboxing.
You might see terminology such as: `confinement: strict`
which means the Snap is restricted by its security policy.
Traditional packages installed with: `sudo apt install <package>` are generally **not sandboxed in the same application-isolation sense**.
### Important distinction

**Not sandboxed ≠ dangerous.**

It simply means: "This application isn't being isolated from the operating system by a sandbox."
For example, many ordinary Linux programs are not sandboxed and are perfectly normal and safe.
The bigger issue is **what permissions the application has**.
A program running as your normal user has your user's permissions.
A program running as `root` has extremely powerful permissions:

```
normal user
    ↓
limited access

root
    ↓
almost complete system access
```

So when Ubuntu says an application **isn't sandboxed**, think:

```
> Sandbox = a restricted room.  
> Not sandboxed = the application is running in the normal Ubuntu environment, subject to its user/system permissions.
```

Installed **Google Chrome from a `.deb` file**, it is a traditional system package rather than a Snap/Flatpak application.

That does **not** mean Chrome has no security isolation at all—Chrome itself uses its own sandbox internally. But if you mean **Ubuntu-level application sandboxing**, you can put the `.deb`-installed Chrome inside another sandbox.
### Your options

**1. Keep your current `.deb` Chrome**

- Chrome already has its **own browser sandbox**.
- This is the simplest option.
- You don't need to sandbox it again for normal use.

**2. Install Chrome as a Flatpak**

- Flatpak provides stronger OS-level application isolation.
- You control which files, devices, and services the application can access.
- This is usually the easiest way to get a Linux desktop-app sandbox.

**3. Use Firejail**

- You can keep your existing `.deb` Chrome and launch it through **Firejail**.
- Example: `sudo apt install firejail`

Then: `firejail google-chrome`
Firejail creates a restricted environment around the application.

### One important thing

Don't simply run: `sudo firejail google-chrome`
unless you specifically understand why you're doing it. Running Chrome as root is unnecessary and undesirable.

