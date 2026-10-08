# 2️⃣4️⃣ Compiling Source code
## Why not just install a .deb file? Why compile from source?

Installing a .deb package is the easiest way to get software on a Debian/Ubuntu system. It’s fast, convenient, and designed for normal users. But .deb packages are limited to pre-built binaries that someone else compiled for you with their own choices — their own features, libraries, optimizations, and versions. When you install from a .deb, you get exactly what the distro maintainers decided, no more, no less.

Compiling from source code gives you much more control. Software can have optional features like GPU acceleration, codec support, hardware capabilities, debugging tools, and performance flags. Distributions often disable some of these features in binary packages because of licensing issues, dependencies, patents, or size. For example, Ubuntu’s FFmpeg often ships without some codecs because of legal reasons, while compiling from source allows you to enable them.

Another big reason is optimization. When building from source, you can compile code tuned specifically for your hardware (CPU architecture, instruction sets, optimization level). Distro maintainers build for the lowest common denominator, so their packages run on many machines, not just yours. If you compile yourself, you can enable flags like -march=native to use all your CPU features.

There are also cases where software evolves faster than your distro’s repositories. For example, Ubuntu LTS might have a version from 2 years ago, while the source project releases updates monthly. Developers, testers, and advanced users compile from source to get new versions, bug fixes, or experimental features long before they land in .deb form.

Finally, source compilation is essential for developers, because they need access to symbols, debugging info, internal structure, and build logs. A .deb package hides this. Source teaches you how software is structured, how dependencies interact, how libraries are linked, and how build systems work — skills valuable for engineering and debugging in the real world.

In summary: .deb is convenience; source is control, customization, and learning. Normal users install .deb files because they just need the software to work. Developers, power users, and system integrators compile from source because they need performance, features, modern versions, and understanding.

### 1. Install C Compiler & Build Tools on Ubuntu

Before compiling anything, you need the essential compiler tools:

#### Install GCC and Build Tools

```
sudo apt update
sudo apt install build-essential
```

This installs GCC (GNU C Compiler), make, and other tools required for building software from source.
#### Check the Compiler Version

`gcc --version` This confirms GCC is installed and working.

### 2. Compile a Simple C Program

#### Example hello.c

Create a file: `nano hello.c`
Put this inside:

```
#include <stdio.h>
int main() {
    printf("Hello, World!\n");
    return 0;
}
```

#### Compile & Run

```
gcc hello.c -o hello
./hello
```

This compiles the C source file hello.c into an executable named hello, then runs it.

### 3. Compile & Install Software from Source

When you download software source code (usually .tar.gz, .zip, or from GitHub), the typical steps are:

#### Step A — Prepare Environment

Install necessary tools: `sudo apt install build-essential checkinstall git`
Extract the source: `tar -xzvf [software.tar.gz](http://software.tar.gz)`
cd software Or clone from GitHub: `git clone https://github.com/author/project.git`
`cd project`

#### Step B — Build Process 

Most open-source C/C++ software uses a 3-step build pattern:

```
./configure
make
sudo make install
```

Let’s go through them one by one.

#### 1. ./configure — Prepare the Build

#### What it does

When you run: `./configure`
it checks:

✔ your system environment  
✔ compiler availability (gcc, clang, etc.)  
✔ required libraries & dependencies  
✔ CPU architecture  
✔ paths for installation (/usr/local/bin, /usr/lib, etc.)

Then it generates a file called Makefile which contains the exact instructions for compiling.
Example Output
You might see something like:

```
checking for gcc... yes
checking for OpenSSL... no
checking prefix... /usr/local
```

If dependencies are missing, it will fail with: `configure: error: OpenSSL headers not found`
Then you fix via apt: `sudo apt install libssl-dev`

Optional Arguments
You can customize install paths: `./configure --prefix=/opt/myapp`
Or enable/disable features: `./configure --enable-ssl --disable-tests`
Key idea: ./configure = create a build plan tailored to your system

#### 2. make — Compile Source Code

Once configure finishes, run: `make`
##### What it does

make reads the generated Makefile and compiles the .c or .cpp files into binaries.

Example:

```
gcc -c file1.c
gcc -c file2.c
gcc -o program file1.o file2.o
```

##### Why use make?

Because building manually would be crazy, imagine: `gcc -Wall -O2 -std=c11 src/*.c -o myprogram`
make automates and tracks dependencies.

##### Useful Variants

Multi-core compile (faster): `make -j$(nproc)`
nproc gives number of CPU cores, so build is parallel.

#### 3. sudo make install — Install to System

After binary is built, run: `sudo make install`

##### What it does

Copies built files into system directories like:

|           |                      |
| --------- | -------------------- |
| File Type | Directory            |
| binaries  | /usr/local/bin       |
| libraries | /usr/local/lib       |
| configs   | /usr/local/etc       |
| man pages | /usr/local/share/man |

##### Example install output:

```
install myapp /usr/local/bin
install libmyapp.so /usr/local/lib
```

Now you can run: `myapp`

##### Important Note

Uninstalling is difficult because system doesn’t track installed files.

#### Safer Alternative — checkinstall

Instead of: `sudo make install`
Use: `sudo checkinstall`
This creates a .deb package so you can later uninstall: `sudo apt remove myapp`

### Putting It All Together (Typical Workflow)

Example for a software package:

```
tar -xzf myapp-1.2.3.tar.gz
cd myapp-1.2.3
./configure --prefix=/usr
make -j$(nproc)
sudo checkinstall
```

### What is CMake?

CMake itself does not compile code. It generates build files for compilers like make, ninja, MSVC, etc.
Think of it like this: `CMake → generates → Makefiles → used by make → compiles code`

### Typical CMake Build Workflow

Almost all CMake projects follow:

```
mkdir build
cd build
cmake ..
make
sudo make install
```

Let's break it down:

### 1. mkdir build && cd build — Out-of-Source Build

Projects avoid building in the source directory.

✔ Keeps source clean  
✔ Easy to delete build artifacts  
✔ You can have multiple builds

### 2. cmake .. — Configure Build

This command reads the CMakeLists.txt from the parent directory.
Example output:

```
-- Checking for module 'zlib'
-- Found zlib: /usr/lib/x86_64-linux-gnu/libz.so
-- Configuring done
-- Generating done
```

This step:

✔ checks dependencies  
✔ detects compilers  
✔ sets install directories  
✔ generates Makefile or Ninja files

### 3. make — Compile the Source

Now compile with: `make -j$(nproc)`
This executes the generated Makefile.
Try : OpenCV, Git, HandBrake, Kernel

