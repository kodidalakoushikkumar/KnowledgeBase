

# 2️⃣ File Share on UBUNTU using samba

```
sudo apt update
sudo apt install samba 
mkdir -p ~/SharedFolder
chmod 777 ~/SharedFolder

sudo nano /etc/samba/smb.conf

[Public]
path = /srv/public
browseable = yes
read only = no
writable = yes
guest ok = yes
guest only = yes
create mask = 0777
directory mask = 0777
force user = nobody

sudo systemctl restart smbd
sudo ufw allow samba
sudo smbpasswd -a username(prefer local user)
sudo smbpasswd -e kodidala.kumar
sudo systemctl restart smbd
```
