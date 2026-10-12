# 1️⃣1️⃣ UFW

A firewall is your system's digital security guard. It inspects all network traffic trying to enter or leave your computer and decides whether to let it pass.

In Linux, the core firewall engine built into the kernel is called **iptables**. However, writing rules for `iptables` directly is notoriously difficult, verbose, and easy to mess up. To make things easier, Ubuntu uses **UFW (Uncomplicated Firewall)**, which is a user-friendly command-line interface that translates simple human commands into complex `iptables` rules behind the scenes.
#### The Core Risk: The Remote Server Trap

Before touching any firewall settings, you must understand the single most dangerous mistake a systems administrator can make: **locking yourself out of your own server**.
**The Scenario**
You are connected to a cloud server (like AWS, DigitalOcean, or a VPS) using **SSH (Secure Shell)** on its default port, **22**. If you enable UFW without setting up an explicit exception rule first, UFW will immediately block all incoming connections—including your active SSH session. Your terminal will freeze, you will be disconnected, and you will have **zero way to log back in** to turn it off.
The Golden Rule
Always configure your allowance rules (especially for SSH) **before** you enable the firewall. UFW lets you view, create, and modify rules while it is completely disabled, making it safe to set up your safety net first.
#### How UFW Works: Default Policies

At its heart, UFW operates on a set of **default policies** that govern how it behaves when no specific rule matches a connection. By default, UFW is configured as follows:

- **Default Incoming: DENY** Blocks all incoming connection attempts from the outside world. Nobody can connect to your machine unless you explicitly allow them to.
- **Default Outgoing: ALLOW** Allows your machine to freely initiate connections to the outside world. You can browse websites (HTTP/HTTPS) and download updates without restriction.

This configuration is **perfect for desktop users** because you want to fetch data from the internet, but you do not want random machines on the web trying to connect to your computer. However, for a **server**, you must customize this behavior so legitimate users can reach your hosted services.
#### Step-by-Step: Setting Up and Managing UFW

Here is the operational lifecycle of managing UFW on your system:

Step A: Installation and Status Check

On Ubuntu systems, UFW is installed by default but is **disabled** so it does not interfere with your initial network setup.

- **Install UFW** (if it is missing): `sudo apt install ufw`
- **Verify Installation:** `ufw --version`
- **Check Status and Active Rules:** `sudo ufw status`

If UFW is inactive, this command will simply return `Status: inactive`. If it is active, it will display a table of your current active firewall rules.

Step B: Setting Your Initial Rules (Safety First!)

Before turning the firewall on, you must allow your management ports. If you use SSH, you can allow it using either the port number or the service name:

- **Allow by Port:** `sudo ufw allow 22`
- **Allow by Service Name:** `sudo ufw allow ssh`

If your server runs a web server (like Apache or Nginx), you should also allow web traffic on port 80: `sudo ufw allow 80` Or at the service level: `sudo ufw allow apache`

Step C: Enabling and Disabling the Firewall

Once your rules are safely in place, you can turn the firewall on : Enable UFW:  `sudo ufw enable`
  *Note:* This starts the firewall immediately and configures it to **automatically start up every time your system boots** . You will see a warning confirming that the firewall is active and enabled on system startup .
If you need to troubleshoot network issues or turn the firewall off:  **Disable UFW:** `sudo ufw disable` 
  This stops the firewall immediately and prevents it from starting on boot.

#### Step D: Applying New Rules (Reloading)
If UFW is already running and you add or change a rule, **the changes do not take effect immediately** . You have two ways to apply them:

1. **The Manual Toggle:** Disable and re-enable UFW .
2. **The Clean Way (Reload):** Keep the firewall running and simply reload the configuration: `sudo ufw reload`
#### Step E: The Emergency Reset

If you make a mistake, create conflicting rules, or lock yourself into an odd configuration, you can wipe the slate clean: `sudo ufw reset`
This command **disables UFW and deletes all custom rules**, returning the firewall to its factory default state (no exceptions).
#### Advanced Rule Configuration: Ports, Protocols, and Applications

UFW gives you incredibly granular control over how you define what traffic is allowed or blocked.
#### 1. Managing IPv4 vs. IPv6
By default, whenever you add a rule (like allowing port 22), UFW adds two rules: one for **IPv4** and one for **IPv6** . 
If you want to disable IPv6 support entirely to simplify your rules:
1. Open the configuration file **/etc/default/ufw** in a text editor.
2. Find the line that says `IPV6=yes`.
3. Change it to **`IPV6=no`**.
#### 2. Filtering by Specific Protocols (TCP vs. UDP)
Some services use specific network transport protocols. By default, UFW allows both protocols. If you want to restrict a rule to just one:
* **Allow TCP only** (common for web traffic): `sudo ufw allow 80/tcp`
* **Allow UDP only** (common for DNS or streaming): `sudo ufw allow 53/udp`
#### 3. Allowing Port Ranges
If you run an application that requires multiple consecutive ports (for example, a game server or a voice-over-IP service), you can open a range. **When specifying a range, you must define the protocol (tcp or udp):** `sudo ufw allow 90:100/tcp`
This opens all ports from 90 through 100 for TCP connections.
#### 4. The Magic of `/etc/services`
When you run a command like `sudo ufw allow ssh` or `sudo ufw deny http`, how does UFW know what numbers those services use?
It reads a special system dictionary file located at **`/etc/services`**. This file maps thousands of human-readable service names to their official port numbers. You can inspect this file yourself using: `less /etc/services`
#### 5. Application Profiles
Many modern Linux applications register **profiles** with UFW during installation. These profiles package all the specific ports and protocols the application needs into one simple, named package.

* **List available application profiles:** `sudo ufw app list`
  *(On a standard machine, you will typically see things like `OpenSSH` or `CUPS` (printing service) listed .)*
* **Allow an application profile:**
  Instead of memorizing which ports a complex app needs, you can simply allow the app profile: `sudo ufw allow OpenSSH`
### Summary Checklist: Essential UFW Commands

| Action | Command | When to use it |
| :--- | :--- | :--- |
| **Check Active Rules** | `sudo ufw status` | To see what ports are currently open or closed [4]. |
| **Pre-Authorize SSH** | `sudo ufw allow 22` | Run this first on remote servers to prevent lockouts [2]. |
| **Turn On** | `sudo ufw enable` | Activates protection and enables boot-start [5]. |
| **Apply Changes** | `sudo ufw reload` | Run this after adding new rules to a running firewall [5]. |
| **Block a Port** | `sudo ufw deny 80` | Blocks incoming traffic on a specific port [6, 7]. |
| **Start Over** | `sudo ufw reset` | Erases all rules and turns off the firewall [5]. |
