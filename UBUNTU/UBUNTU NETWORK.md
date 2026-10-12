

# 1️⃣4️⃣ UBUNTU NETWORK

**Phase 1: Physical, Driver & Hardware Verification (Bottom-Up Diagnostics)**
**Step 1: Verify Driver & Hardware Detection**

- **Action:** Check if the operating system recognizes the Network Interface Card (NIC). `lspci | grep -i net`
- **Action:** Check if the kernel driver module for your interface is loaded. `lsmod | grep <driver>`
- **Diagnostic:** If these commands return nothing, **the driver is missing** or the hardware is not properly seated.

**Step 2: Check Physical Link Status**

- **For Ethernet:** Verify the hardware connection and link speed. `ethtool <interface>`
- _Look for:_ `Link detected: yes` and `Speed: 1000Mb/s` (or `100/10` depending on your hardware).
- **For Wi-Fi:** Check the device management status. `nmcli device status`
- _Look for:_ The Wi-Fi interface should be listed as **connected** or **disconnected** (it should not read **unavailable**).
- **Diagnostic:** If `Link detected: no` or the status is unavailable, you have a **cable, port, or physical NIC issue**.

**Phase 2: Interface Control & Local IP Configuration**
**Step 3: Check IP Address Assignment**

- **Action:** Check the IP configuration of your active interfaces. `ip a`
- _Identify interfaces:_ Ethernet interfaces typically start with `enp...` or `eth0`; Wi-Fi interfaces typically start with `wlp...` or `wlan0`.
- **Diagnostic (IP States):**
    - `192.x.x.x`, `172.x.x.x`, or `10.x.x.x` $\rightarrow$ **DHCP is working properly**.
    - `169.254.x.x` $\rightarrow$ **DHCP failed** (Auto-IP / APIPA fallback).
    - _No IP address at all_ $\rightarrow$ **The interface is not up**.
- **Remediation:** If the interface is down, attempt to bring it up manually: `sudo ip link set <iface> up`

**Step 4: Clear & Renew DHCP**

- **Remediation:** If DHCP failed or you need a fresh lease, release the old lease and request a new IP:

```
sudo dhclient -r
sudo dhclient
```

**Phase 3: Network Services & Local Security**
**Step 5: Verify Network Manager Status**

- **Action:** Check if the system's network service is running. `systemctl status NetworkManager`
- **Remediation:** If the service is inactive or down, restart it: `sudo systemctl restart NetworkManager`

**Step 6: Check Firewall Restrictions**

- **Action:** Check if the Uncomplicated Firewall (UFW) is actively blocking connection attempts. `sudo ufw status`
- **Remediation:** If UFW is active and suspected of causing connectivity issues, temporarily disable it to isolate the issue: `sudo ufw disable`

**Phase 4: Routing & DNS Resolution**
**Step 7: Check Gateway Routing**

- **Action:** Verify that a path exists from your system to external networks. `ip route`
- **Diagnostic:** Ensure a default route exists in the output (e.g., `default via <gateway_ip>`). **If this default route is missing, you will not have internet access.**

**Step 8: Check DNS Server Settings**

- **Action:** Verify that your system knows where to send domain name lookup queries. `cat /etc/resolv.conf`
- **Diagnostic:** Look for a configured nameserver (e.g., `nameserver 8.8.8.8` or your local/company DNS addresses).

**Phase 5: Diagnostics & Global Connectivity**
**Step 9: Run Raw Connectivity Tests**

- **Action:** Execute sequential ping tests to isolate where traffic is dropping.

```
ping -c 3 8.8.8.8
ping -c 3 google.com
```

- **Diagnostic Matrix:** 

| Ping Test Result                        | Indicated Failure Point                        |     |
| :-------------------------------------- | :--------------------------------------------- | --- |
| `8.8.8.8` fails                         | **Gateway, routing, or ISP issue**             |     |
| `8.8.8.8` works, but `google.com` fails | **DNS configuration issue**                    |     |
| Both fail                               | **Network stack, hardware, or firewall block** |     |

**Step 10: Worst Case — Reset Network Stack**

- **Remediation:** If configuration files are correct but traffic is still stalled, toggle your networking services off and on to clear the stack:

```
sudo nmcli networking off
sudo nmcli networking on
# Alternate method: sudo systemctl restart NetworkManager
```

