# 3️⃣6️⃣ Permanent DNS set

How to **`nameserver 10.60.10.3` to remain permanently in `/etc/resolv.conf`** on native Ubuntu.
Your `/etc/resolv.conf` is managed by `systemd-resolved`, so **don't edit that file directly**. Configure the DNS server underneath it.
### Permanent solution

Run: `sudo resolvectl dns wlp3s0 10.60.10.3`
Then verify: `resolvectl status`
You should see:

```
Link 3 (wlp3s0)
    Current DNS Server: 10.60.10.3
    DNS Servers: 10.60.10.3
```

However, `resolvectl dns` by itself can be lost when the network connection changes/reconnects.

### Make it permanent with NetworkManager

First get your Wi-Fi connection name: `nmcli connection show`
You'll see something like:

```
NAME              UUID                                  TYPE      DEVICE
Office WiFi       xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  wifi      wlp3s0
```

Then run: `sudo nmcli connection modify "Office WiFi" ipv4.dns "10.60.10.3"`
Also tell NetworkManager to use the DNS you specified: `sudo nmcli connection modify "Office WiFi" ipv4.ignore-auto-dns yes`
Reconnect:

```
sudo nmcli connection down "Office WiFi"
sudo nmcli connection up "Office WiFi"
```

Now check: `resolvectl status`
and: `cat /etc/resolv.conf`
You will still see: `nameserver 127.0.0.53`
**That is normal.** `127.0.0.53` is Ubuntu's local `systemd-resolved` DNS stub. The actual upstream DNS will be: `10.60.10.3`
So you don't actually need the literal line `nameserver 10.60.10.3` in `/etc/resolv.conf`; `systemd-resolved` forwards the requests to `10.60.10.3`.