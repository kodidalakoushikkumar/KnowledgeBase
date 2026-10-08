# 2️⃣6️⃣ Prompt String

`PS1` is the **Primary Prompt String** in Bash.  
It controls **how your terminal prompt looks** (username, host, path, colors, etc.).
`${debian_chroot:+($debian_chroot)}`
This shows the **chroot name** if you are inside a chroot environment.
Syntax explanation : `${variable:+value}`
Meaning:

- If **variable exists and is not empty → print value**
- Otherwise → print **nothing**

So: `${debian_chroot:+($debian_chroot)}`
means:

|Condition|Output|
|---|---|
|debian_chroot exists|`(name)`|
|not in chroot|nothing|

Example:  `(mychroot) user@host:~$`
Normally you won't see this unless using **chroot containers**.
`\[\033[01;32m\]` This sets **text color**. `\033` means **escape character**.  `01;32m` means:

| Code | Meaning     |
| ---- | ----------- |
| 01   | Bold        |
| 32   | Green       |
| m    | apply color |

So this sets **bold green color**.
`\u` : Displays the **username**.
Example: `koushik`
`@` : Just prints the `@` symbol.
`\h` : Shows the **hostname** (computer name).
Example: `koushik@ubuntu`
`\[\033[00m\]` : This **resets the color** back to normal.
`00m` : means **reset formatting**.
`:` Just prints a colon.
Example: `koushik@ubuntu:`
`\[\033[01;34m\]` : Sets **bold blue color**.

|Code|Meaning|
|---|---|
|01|Bold|
|34|Blue|
`\w`
Displays the **current working directory**.
Example:  `~/projects/react`
So prompt becomes:  `koushik@ubuntu:~/projects/react`
`\[\033[00m\]` : Reset color again.
`\$` Shows:

|User type|Symbol|
|---|---|
|normal user|`$`|
|root user|`#`|

Example:  `koushik@ubuntu:~$` or `root@ubuntu:/#`
What `\[ \]` does
`\[` Start of **non-printing sequence** 
`\]`End of **non-printing sequence**
So Bash **ignores the length** of whatever is inside.