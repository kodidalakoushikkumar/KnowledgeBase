

# 2️⃣8️⃣ Add user to docker group
### Step-by-step fix

1. **Create the docker group (if it doesn’t exist):** `sudo groupadd docker`
2. **Add your user to the docker group:** `sudo usermod -aG docker $USER`
3. **Apply the changes (choose one):** Log out and log back in  **OR** Run: `newgrp docker`

### Verify it worked

Run: `docker run hello-world`, If it runs without `sudo`, you're good 👍

### What’s happening behind the scenes?

When you run a command like: `docker ps`
You’re not directly running Docker itself—you’re talking to the **Docker daemon** (`dockerd`), which runs as **root**.

### Why does `sudo` work?

When you do: `sudo docker ps`
You’re basically saying: “Run this command as root”
Since the Docker daemon only accepts requests from **root or authorized users**, it works.

### So what does the `docker` group do?

Linux uses **groups** to control access to things like files and sockets.
Docker exposes a special file (a Unix socket): `/var/run/docker.sock`
Check its permissions: `ls -l /var/run/docker.sock`
You’ll see something like: `srw-rw---- 1 root docker ...`
Meaning:

- Owner: `root`
- Group: `docker`
- Permissions: read/write for owner and group

### Key idea

If your user is in the **docker group**, you can access that socket **without sudo**.
So when you run: `docker ps`
It works because:

- You → belong to `docker` group
- Socket → allows `docker` group access
- Docker daemon → accepts your request

### Why do you need to log out?

When you add yourself to a group: `sudo usermod -aG docker $USER`
Your current session doesn’t immediately “know” about it.
Logging out and back in refreshes your group memberships.


	




