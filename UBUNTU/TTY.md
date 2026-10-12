---
Title: fstab
Type:
Topic:
tags:
cover: "[[/IMAGES/TTY.png]]"
---
# TTY

#### Demystifying the Linux TTY: A Historical and Architectural Masterclass
When you open a terminal in Linux, you are interacting with a software design that is over 70 years old.
The name **TTY** stands for **Teletypewriter**. It represents the evolutionary bridge from physical, paper-spitting typewriter machines to the modern windowed command lines we use today.
#### Phase 1: The Dawn of Computing (No Screens, No Keyboards)

In the **1950s and 1960s**, computers did not have monitors, keyboards, or mice. Instead, programmers used **punch cards**.

```
[Write Code on Paper] ➔ [Punch Holes in Card] ➔ [Feed Cards to Computer] ➔ [Wait Hours] ➔ [Get Printed Output]
```

This method was painfully slow, tedious, and completely lacked real-time interaction. If you made a single typo, your entire stack of cards was ruined, and you had to start over.

#### Phase 2: Humans Want to "Talk" (The Teletype Era)

To solve this, engineers looked for a way to have a live conversation with the computer. They took an existing technology—the electric typewriter—and wired it directly to the computer mainframe. This machine was called a **Teletype** (or **TTY** for short).

```
+--------------------------------------------------+
|                    TELETYPE                      |
|  [Keyboard] ➔ (User types "ls")                  |
|                                                  |
|  [Printer]  ➔ (Computer prints files on paper)   | [1, 2]
+--------------------------------------------------+
```

When you sat at a TTY:

1. You typed a command on the physical keyboard (e.g., `ls`).
2. The electrical signals traveled down a wire to the massive computer box down the hall.
3. The computer processed the command and sent back electrical signals.
4. The TTY's physical printer typed the output directly onto a roll of paper (e.g., printing `file1 file2`).

There was still **no monitor**. Your command history and the computer's output were physically printed on continuous rolls of paper.
#### Phase 3: From Paper to Glass (The Terminal is Born)

By the **1970s**, continuous paper printing was recognized as slow, noisy, and extremely expensive. Engineers decided to replace the heavy physical printer mechanism with a **glass television screen**. This new combination of a keyboard and a screen was called a **terminal**. Crucially, **the terminal was designed to behave exactly like the old paper-based TTY**. The computer sent the exact same characters, but instead of a physical typewriter hammer hitting paper, a beam of electrons painted characters on a screen. Because the underlying communication protocols didn't change, the term **TTY stuck around** as the standard name for these communication lines.

#### Phase 4: The Birth of UNIX and the "Everything is Text" Philosophy

During this exact transition from teletypes to video terminals, the **UNIX operating system** was born. UNIX was designed around a core philosophy: **"Everything is text"**. Because mainframes were incredibly expensive, companies couldn't buy a computer for every employee. Instead, they bought **one central UNIX mainframe** and connected **dozens of physical terminals** to it using long serial cables.

```
                  +-----------------------+
                  |  Central UNIX Server  |
                  +-----------+-----------+
            /-----------------+-----------------\
           /                  |                  \
  +--------+--------+  +--------+--------+  +--------+--------+
  |  User 1 Terminal|  |  User 2 Terminal|  |  User 3 Terminal|
  |     (TTY1)      |  |     (TTY2)      |  |     (TTY3)      | [2, 3]
  +-----------------+  +-----------------+  +-----------------+
```

Each user sat at their own terminal. To the central mainframe, **each connected wire was designated as a unique TTY**. The operating system read text input from that specific TTY line, processed it, and routed the text output back down the same TTY line to the user's screen.

#### Phase 5: The Architecture (TTY vs. Shell)

It is very common to confuse the terminal (TTY) with the shell, but they perform completely different jobs.

1. **The TTY (The Communication Line):** This is the physical or virtual connection wire. Its job is purely transmission: it takes the raw keystrokes you type and sends them to the computer, and takes the raw text bytes from the computer and displays them on your screen.
2. **The Shell (The Translator):** Computer programs and the operating system kernel cannot natively understand human language. The shell is a special software translator that sits on the receiving end of the TTY.
    - When you type `ls` and hit enter, the **TTY** sends those raw letters to the system.
    - The **Shell** reads the letters `l` and `s`, translates them to mean _"run the list-directory program,"_ executes that program, and feeds the resulting directory list back to the TTY to display on your screen.

#### Phase 6: Personal Computers and Virtual TTYs

By the **1980s and 1990s**, Personal Computers (PCs) arrived. Instead of sharing one giant mainframe down the hall, every user had their own individual computer with a monitor and keyboard physically built right into the box. Even though the physical serial wires were gone, **Linux kept this exact same multi-terminal UNIX architecture** in its core code. When your Linux computer boots up:

1. It initializes several **virtual TTYs** in the background.
2. It displays a text-based login prompt on one of them.
3. Once you log in, it starts a **shell interpreter** inside that TTY.

This design means your single, personal computer still behaves internally like a powerful multi-user mainframe.
#### Phase 7: Graphical User Interfaces (GUIs) and Terminal Emulators

When you use a modern desktop environment like GNOME, KDE, or Ubuntu's default desktop, you are using a **Graphical User Interface (GUI)** with windows, icons, and a mouse. Because modern users expect graphical windows but still need access to the command line, engineers created **Terminal Emulators** (such as GNOME Terminal, Alacritty, or `xterm`). A terminal emulator is simply a **graphical window that pretends to be a physical hardware TTY**.

```
[You Type in GNOME Terminal] 
           │
           ▼
[Linux Creates a Fake TTY: Pseudo-Terminal (PTY)] 
           │
           ▼
[Bash Shell runs inside PTY, thinks it's talking to a hardware teletype] [4]
```

Under the hood, when you open GNOME Terminal, Linux allocates a **Pseudo-Terminal (PTY)**. To the Bash shell running inside it, this PTY behaves exactly like a physical, hardware TTY from the 1970s. The shell has no idea it is running inside a graphical window; it simply reads and writes text streams exactly as it did decades ago.
#### Phase 8: The Ultimate Troubleshooting Lifesaver (Ctrl+Alt+F Keys)

Because Linux maintains these virtual hardware TTYs in the background, you have a powerful safety net built directly into your system keyboard. Linux systems run several real, text-only TTYs in the background (usually designated as `tty1`, `tty2`, `tty3`, up to `tty6`). Your graphical desktop (GUI) is typically launched on just **one** of these virtual screens (frequently `tty1` or `tty2`). The Emergency Rescue Scenario, Imagine you are updating your graphics drivers or running a heavy application, and your graphical desktop suddenly **freezes completely**. Your mouse cursor won't move, your windows are locked, and clicking does nothing.
On Windows, you are forced to hold down the physical power button to hard-reboot, risking data corruption.
On Linux, you can bypass the frozen GUI entirely using the keyboard shortcut: **Ctrl + Alt + F3** (or `F4`, `F5`, `F6`)
This key combination instantly switches your monitor away from the frozen graphical interface (`tty1`) and drops you directly into a raw, black-and-white, text-only virtual console (tty3).
Once there, you can:

1. Type your username and password to log in.
2. Run troubleshooting commands (like `top` or `ps aux` to find the frozen application).
3. Kill the frozen desktop process safely (e.g., `sudo systemctl restart gdm`).
4. Switch back to your fixed graphical desktop using **Ctrl + Alt + F1** or **Ctrl + Alt + F2**.

This keeps your running work safe and prevents you from having to hard-reset your hardware.

Summary Checklist: TTY Concepts at a Glance

|Term|What it is in Simple Terms|Modern Equivalent|
|---|---|---|
|**TTY**|A communication line mapping keystrokes to text output.|`/dev/ttyX` virtual console.|
|**Terminal**|A physical screen and keyboard combination.|The physical monitor and keyboard plugged into your computer.|
|**Shell**|The command interpreter/translator that executes programs.|`bash`, `zsh`, or `sh`.|
|**Terminal Emulator**|A software window drawing a fake terminal.|GNOME Terminal, Termius, or `xterm`.|
|**PTY**|A "Pseudo-Terminal" created inside a GUI.|`/dev/pts/0`, `/dev/pts/1`, etc.|

# Other Version TTY
### STEP 1: Before screens existed (1950s–60s)

Early computers had:

- No monitor
- No keyboard
- No mouse

People used punch cards
You punched holes → fed cards → waited → got printout.
Painful. Slow.

### STEP 2: Humans want to talk to computers

Engineers said: “We need a typewriter connected to the computer”
So they made a machine:

- Keyboard
- Printer (not screen)

You type → computer replies → it prints on paper
This machine was called a: Teletype (short: TTY)
TTY = a typing machine connected to a computer

### STEP 3: TTY becomes standard

Every computer had:

- One computer (big box)
- Many teletypes connected by wires

People sat at TTY machines and typed commands.
Example: ls
Computer prints: file1 file2
No screen. Just paper.

### STEP 4: Screens replace printers (1970s)

Printers were slow and expensive.
So engineers replaced: Printer with Screen
Keyboard + screen = terminal
Important: The terminal behaved EXACTLY like a TTY
So the name TTY stayed even though paper was gone.

### STEP 5: UNIX is born (this matters)UNIX was created in this era.

UNIX philosophy: “Everything is text”
So UNIX:

- Talks through TTY
- Reads input from TTY
- Writes output to TTY
- This is why Linux (UNIX-like) still uses TTY today.

### STEP 6: One big computer, many users

One UNIX computer.
Many people connect using terminals.
Each person:

- gets one TTY
- logs in
- runs commands

TTY = your communication line

### STEP 7: What is a shell? (finally)

Computer programs are dumb.
They don’t understand human words.
So UNIX added a translator.
This translator:

- reads text
- understands commands
- runs programs

This program is the: Shell
Example: You → "ls" Shell → "run the ls program"
So:

- TTY = communication line
- Shell = translator

### STEP 8: PCs arrive (1980s–90s)

Now:

- Each person has their own computer
- Screen + keyboard built in

But UNIX/Linux kept the same design.
So when Linux boots:

1. It creates TTY
2. Shows login3. Starts a shell

Nothing changed conceptually.

### STEP 9: GUI appears (Windows, X11, GNOME)

People want windows and mouse.
So engineers said: “Let’s draw fake terminals inside windows”
This is: Terminal Emulator
Examples:

- GNOME Terminal
- xterm

They pretend to be a real TTY
Internally:

- Linux creates a fake TTY (PTY)
- Shell runs inside it

### STEP 10: Why Ctrl+Alt+F keys exist

Those keys switch between real TTYs.
Linux always keeps:

- TTY1, TTY2, TTY3…

GUI usually runs on one of them.
If GUI dies:

- TTY still works
- You can log in and fix stuff
