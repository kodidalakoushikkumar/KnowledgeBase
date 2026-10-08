# 1️⃣ Do nothing when closing the lid on UBUNTU


➖ sudo edit /etc/systemd/logind.conf
➖ If HandleLidSwitch is not set to ignore then change it:
➖ HandleLidSwitch=ignore
➖ sudo systemctl restart systemd-logind