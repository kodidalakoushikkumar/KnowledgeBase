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


