## Scripts Run in a Subshell

When you run a script (e.g. `./myscript.sh`), it runs in a **subshell**. This is a new, separate shell process. When the script ends, the subshell closes and **any changes made inside it are lost.**

**Example:**

```bash
#!/bin/bash
cd /tmp
pwd        # prints /tmp
```

```bash
cd ~
./myscript.sh   # prints /tmp
pwd             # still prints your home directory
```

The `cd` only happened inside the subshell. Your original shell never moved.

**This also applies to variables:** a variable set inside a script is not available in your shell after the script ends.

**To keep the changes:** run the script with `source` (or `.`). It then runs in the **current shell** instead of a subshell:

```bash
source ./myscript.sh
```

```bash
. ./myscript.sh
```

**Summary:**

| How you run it | Where it runs | Changes kept? |
|---|---|---|
| `./script.sh` | Subshell (new shell) | No |
| `source script.sh` or `. script.sh` | Current shell | Yes |


----

## Shebang چیست؟

**Shebang** همان خط اول اسکریپت است که با `#!` شروع می‌شود:

```bash
#!/bin/bash
```

این خط به سیستم‌عامل می‌گوید **این فایل را با کدام برنامه اجرا کند.**

## اسمش از کجا آمده؟

- `#` را در انگلیسی **sharp** یا **hash** می‌گویند.
- `!` را **bang** می‌گویند.

این دو با هم شده‌اند **sha-bang** و بعدها **shebang**.

## چطور کار می‌کند؟

وقتی می‌زنی `./myscript.sh`، سیستم خط اول را می‌خواند، می‌بیند `#!/bin/bash` نوشته شده و در واقع این دستور را اجرا می‌کند:

```bash
/bin/bash ./myscript.sh
```

یعنی بعد از `#!` **مسیر کامل برنامه‌ای** می‌آید که باید اسکریپت را اجرا کند. مدرست هم با جمله‌ی «Shell to use» به همین خط اشاره می‌کرد.

## چند نمونه‌ی رایج

| Shebang | معنی |
|---|---|
| `#!/bin/bash` | با Bash اجرا شود |
| `#!/bin/sh` | با شل استاندارد sh اجرا شود |
| `#!/usr/bin/env bash` | Bash را هر جای سیستم که نصب باشد پیدا کند و با آن اجرا کند |
| `#!/usr/bin/env python3` | با پایتون اجرا شود |

نسخه‌ی `#!/usr/bin/env bash` قابل‌حمل‌تر است، چون Bash در همه‌ی سیستم‌ها دقیقاً در `/bin/bash` نیست.

## چند نکته‌ی مهم

1. Shebang **باید دقیقاً در خط اول** باشد، حتی بدون فاصله یا خط خالی قبلش.
2. اگر اسکریپت را این‌طور اجرا کنی: `bash myscript.sh`، خودت برنامه را مشخص کرده‌ای و shebang نادیده گرفته می‌شود.
3. اگر shebang نداشته باشی، سیستم معمولاً اسکریپت را با شل پیش‌فرض اجرا می‌کند. اگر کدت مخصوص Bash باشد، ممکن است درست کار نکند.
4. برای اجرا با `./`، فایل باید اجازه‌ی اجرا داشته باشد:
   ```bash
   chmod +x myscript.sh
   ```

---


**Shebang (`#!`)**: the first line of a script. It tells the system which interpreter to run the script with.

```bash
#!/bin/bash
```

| Shebang | Meaning |
|---|---|
| `#!/bin/bash` | Run with Bash |
| `#!/bin/sh` | Run with the standard sh shell |
| `#!/usr/bin/env bash` | Find Bash wherever it is installed and run with it |
| `#!/usr/bin/env python3` | Run with Python 3 |

- It must be on the **very first line**.
- `#!/usr/bin/env bash` is more portable, because it finds bash wherever it is installed.
- If you run the script as `bash script.sh`, the shebang is ignored.
- The script needs execute permission (`chmod +x script.sh`) to run as `./script.sh`.

---


**Shell**: a program that reads your commands and tells the operating system what to do. It runs inside a terminal.

| Shell | Notes |
|---|---|
| `sh` | Original Bourne shell. Basic and available almost everywhere. |
| `bash` | "Bourne Again Shell". Compatible with sh but has more features. Default on most Linux distros. |
| `zsh` | Similar to bash, with better autocompletion and customization. Default on macOS. |

- Some features exist in only some shells. For example, `[[ ]]` and arrays work in bash and zsh but not in sh, and array indexes start at 0 in bash but at 1 in zsh.
- That is why a script should start with a shebang (`#!/bin/bash`): it makes the script run in the intended shell.
- Check your current shell with `echo $SHELL`.

---

In Linux, sourcing a script means running it in the current shell, rather than starting a separate shell process to execute it.

There are two equivalent ways to source a script.

## 1. How to source a script

Suppose you have a script named `config.sh`.

Option 1 — Using `source`

```
source config.sh
```

Option 2 — Using a dot (`.`)

```
. config.sh
```

Both commands do the same thing in Bash. If the script is in another directory, provide its path:

```
source /path/to/config.sh
```

If it's in the current directory, you can explicitly write:

```
source ./config.sh
```

## 2. What is the difference between sourcing and executing?

Suppose `config.sh` contains:

```
#!/bin/bash

export APP_ENV="development"
cd /tmp
```

Compare these three commands:

| Command            | Behavior                                                               |
| ------------------ | ---------------------------------------------------------------------- |
| `./config.sh`      | Executes the script in a separate process (assuming it is executable). |
| `bash config.sh`   | Starts a new Bash process to execute the script.                       |
| `source config.sh` | Runs the script in your current shell.                                 |

Let's see why this matters.

If you run:

```
source config.sh
echo "$APP_ENV"
pwd
```

You will see:

```
development
/tmp
```

The environment variable and directory change remain in your current shell.

However, if you execute the script using:

```
bash config.sh
```

the variable and directory changes won't persist in your parent shell.

Important: A script sourced into your shell can also change shell options, define functions, and even terminate your current shell if it executes `exit`.

## 3. When should you source a script?

Common use cases include:

- Loading environment variables.
- Loading shell configuration.
- Defining reusable shell functions.
- Activating a Python virtual environment.
- Loading helper functions into another Bash script.

For example, to reload your Bash configuration after editing it:

```
source ~/.bashrc
```

Or to load environment variables from a file:

```
source .env.sh
```

One important distinction: sourcing a script does not require it to have executable permission, because the current shell reads and executes its contents directly.

In short: Use `source script.sh` when you want the script's changes to affect your current shell session.

---

### Bash: Removing a Prefix

- `${VAR#PATTERN}`: Removes the **shortest matching prefix** from a variable's value.
- `${VAR##PATTERN}`: Removes the **longest matching prefix** from a variable's value.
- The original variable remains unchanged.

**Example:**
```bash
COUNTER="-5"
echo "${COUNTER#-}"  # Output: 5
```

---

### Bash: Parameter Expansion

- `${VAR#PATTERN}`: Removes the **shortest matching prefix** from a variable's value.
- `${VAR##PATTERN}`: Removes the **longest matching prefix** from a variable's value.
- `${VAR%PATTERN}`: Removes the **shortest matching suffix** from a variable's value.
- `${VAR%%PATTERN}`: Removes the **longest matching suffix** from a variable's value.
- These operations do not modify the original variable unless the result is assigned back to it.

**Examples:**

```bash
# Remove a prefix
VAR="uid=zahra,ou=users"
echo "${VAR#*=}"     # zahra,ou=users
echo "${VAR##*=}"    # users

# Remove a suffix
VAR="zahra,ou=users"
echo "${VAR%,*}"     # zahra,ou=users? 
echo "${VAR%%,*}"    # zahra
```


---

