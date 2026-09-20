# Lesson 16: Shell Scripting Basics & Syntax

**Module:** Scripting (Bash & Python)
**Duration:** 120-150 min
**Prerequisites:** Linux Basics module (permissions, piping/redirection), Git Version Control module

## Learning Objectives

By end of lesson student can:
- Write a script with a correct shebang, make it executable, and explain the difference between running it as `./script.sh` and `bash script.sh`
- Declare and use variables correctly, including quoting rules
- Read user input and access positional parameters
- Write conditionals using `if`/`elif`/`else`, `test`, `[ ]`, and `[[ ]]`
- Explain and use exit codes and logical operators (`&&`, `||`, `!`)

## Topics

- Script structure: shebang (`#!/bin/bash`), `chmod +x`, `./script` vs `bash script`
- Variables: declaration, quoting (`""` vs `''`), `$VAR`, `${VAR}`, `readonly`, `unset`
- User input: `read`, `read -p`, `read -s`; positional params: `$1` `$2` `$@` `$#` `$0`
- Conditionals: `if`/`elif`/`else`/`fi`, `test`/`[ ]`/`[[ ]]`, comparison operators (`-eq`, `-lt`, `-z`, `-f`, `-d`)
- Exit codes: `$?`, `exit N`; logical operators: `&&`, `||`, `!`

## Concepts

### The shebang and running a script

The **shebang** — the first line of a script, `#!/bin/bash` — tells the kernel which interpreter should execute the rest of the file when it's run directly as a program. Without it, the shell has no way to know whether the file's contents are Bash, Python, or something else.

Two distinct ways to run a script:

- `bash script.sh` — explicitly tells Bash to interpret the file's contents. The shebang line is irrelevant here (it's just treated as a comment), and the file doesn't need execute permission.
- `./script.sh` — asks the kernel to execute the file *directly*, which requires (1) execute permission (`chmod +x script.sh`, Lesson 3) and (2) a valid shebang line so the kernel knows what to hand the file's contents to.

`#!/bin/bash` vs `#!/usr/bin/env bash` — both work; `env bash` looks up `bash` in `$PATH` rather than assuming it's at `/bin/bash`, which is slightly more portable across systems where bash might live elsewhere, though on Ubuntu `/bin/bash` is a safe, standard assumption.

### Variables: declaration and quoting

A shell variable is assigned with `name=value` — **no spaces** around the `=` (a very common syntax error for beginners coming from other languages). Reading it back uses `$name` or `${name}` (the braces form is required when the variable name would otherwise blend into surrounding text, e.g. `${name}_suffix`).

Quoting matters a great deal in Bash:

| Quoting | Behavior |
|---|---|
| `"double quotes"` | Variables (`$VAR`) and command substitutions are expanded inside; spaces are preserved as literal text |
| `'single quotes'` | Nothing is expanded — everything inside is treated as literal text, including `$VAR` |
| no quotes | Variables are expanded, AND the result is subject to word-splitting (broken into separate words on whitespace) and glob expansion — usually not what you want for variables that might contain spaces |

`readonly name=value` makes a variable's value permanent for the rest of the script — any later attempt to reassign it fails. `unset name` removes a variable entirely, as distinct from setting it to an empty string (`name=""`), which keeps the variable defined but empty.

### Reading user input

`read variable_name` pauses the script and waits for the user to type a line, storing it in the named variable. `read -p "Prompt: " variable_name` combines this with printing a prompt first, so you don't need a separate `echo` before it. `read -s variable_name` reads without echoing the typed characters back to the terminal — used for passwords or other sensitive input.

### Positional parameters

When a script is invoked with arguments (`./deploy.sh production v2.1`), Bash automatically makes them available as **positional parameters**:

| Parameter | Value |
|---|---|
| `$0` | The script's own name/path as invoked |
| `$1`, `$2`, ... | The first, second, etc. argument |
| `$#` | The total number of arguments passed |
| `$@` | All arguments, each as a separate word (the standard choice for "give me everything passed in") |

`"$@"` (quoted) correctly preserves each argument as a separate item even if an individual argument contains spaces — this distinction becomes important once loops over arguments are introduced in the next lesson.

### Conditionals

Bash's `if` statement structure:

```bash
if condition; then
    # commands
elif other_condition; then
    # commands
else
    # commands
fi
```

The `condition` is actually just a command being run — `if` succeeds down the `then` branch if that command's exit code is 0 (success), and falls through to `elif`/`else` otherwise. `test`, `[ ]`, and `[[ ]]` are the commands almost always used as that condition:

- `test expression` and `[ expression ]` are equivalent (`[` is literally a command, and `]` is its required closing argument) — this is the older, POSIX-portable form.
- `[[ expression ]]` is Bash's own extended, more forgiving syntax — safer with unquoted variables containing spaces, and supports additional operators (like pattern matching and `&&`/`||` directly inside the brackets). Prefer `[[ ]]` in Bash-specific scripts; use `[ ]`/`test` only when POSIX portability to non-Bash shells genuinely matters.

Common comparison operators:

| Operator | Tests |
|---|---|
| `-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge` | Numeric equal/not-equal/less-than/etc. |
| `==`, `!=` | String equal/not-equal (inside `[[ ]]`; use `=` for POSIX `[ ]`) |
| `-z` | String is empty |
| `-n` | String is non-empty |
| `-f` | Path exists and is a regular file |
| `-d` | Path exists and is a directory |

Note: `-eq`/`-lt`/etc. are for **numbers**; `==`/`!=` are for **strings**. Using `==` to compare `5` and `10` numerically works by accident for equal values but is semantically wrong and can misbehave — always match the operator to the data type being compared.

### Exit codes

Every command, including a whole script, finishes with an **exit code**: an integer from 0 to 255, where `0` conventionally means success and any non-zero value means some kind of failure (the specific non-zero meaning is up to whatever produced it). `$?` immediately after a command holds that command's exit code — but only if checked *immediately*, since the next command run overwrites it. `exit N` inside a script ends it immediately with exit code `N` (defaulting to the exit code of the last command run, if `exit` is called with no argument).

### Logical operators

`&&` and `||` chain commands based on the previous command's exit code, short-circuit style: `cmd1 && cmd2` runs `cmd2` only if `cmd1` succeeded (exit 0); `cmd1 || cmd2` runs `cmd2` only if `cmd1` failed (non-zero exit). `!` negates a condition's truth value, most often seen inside `if !` or `[[ ! -f file ]]` ("if this file does NOT exist").

## Commands / Syntax Reference

| Syntax | Purpose | Example |
|---|---|---|
| `#!/bin/bash` | Shebang line | first line of script |
| `chmod +x` | Make script executable | `chmod +x deploy.sh` |
| `VAR=value` | Assign a variable | `NAME="Alex"` |
| `$VAR` / `${VAR}` | Read a variable | `echo "Hello, $NAME"` |
| `readonly` | Make a variable immutable | `readonly MAX_RETRIES=3` |
| `unset` | Remove a variable | `unset TEMP_VAR` |
| `read -p` | Prompt and read input | `read -p "Name: " name` |
| `read -s` | Read without echoing input | `read -s password` |
| `$1`, `$2`, ... | Positional arguments | `echo "First arg: $1"` |
| `$#` | Argument count | `if [[ $# -eq 0 ]]; then` |
| `$@` | All arguments | `echo "$@"` |
| `if`/`elif`/`else`/`fi` | Conditional block | see walkthrough |
| `[[ ]]` | Bash conditional test | `[[ -f "$1" ]]` |
| `$?` | Last exit code | `echo $?` |
| `exit N` | Exit with code N | `exit 1` |

## Examples / Walkthrough

```bash
#!/bin/bash
# --- script structure ---
# save as greet.sh
echo "Script name: $0"

# make it executable, then run both ways:
# chmod +x greet.sh
# ./greet.sh        <- needs execute permission + shebang
# bash greet.sh     <- works regardless of permission or shebang
```

```bash
#!/bin/bash
# --- variables and quoting ---
NAME="Alex Student"
GREETING="Hello, $NAME!"                # double quotes: $NAME expands
LITERAL='Hello, $NAME!'                 # single quotes: $NAME stays literal text

echo "$GREETING"                         # -> Hello, Alex Student!
echo "$LITERAL"                          # -> Hello, $NAME!

# unquoted variable danger: word-splitting
PATH_WITH_SPACE="my file.txt"
echo $PATH_WITH_SPACE                     # unquoted: may be treated as TWO words
echo "$PATH_WITH_SPACE"                   # quoted: correctly treated as one string

readonly MAX_RETRIES=3
# MAX_RETRIES=5                            # this line would fail: readonly, cannot reassign

TEMP="scratch value"
unset TEMP
echo "TEMP is now: '$TEMP'"                # -> TEMP is now: '' (empty, variable is gone)
```

```bash
#!/bin/bash
# --- reading user input ---
read -p "Enter your name: " user_name
echo "Hello, $user_name!"

read -p "Enter your favorite color: " color
echo "You chose: $color"

read -s -p "Enter a password (hidden): " pw
echo                                        # newline, since -s suppresses the input's own newline
echo "Password length: ${#pw}"               # ${#var} = length of the string in var
```

```bash
#!/bin/bash
# --- positional parameters ---
# run as: ./deploy.sh production v2.1

echo "Script: $0"
echo "First argument (environment): $1"
echo "Second argument (version): $2"
echo "Total arguments passed: $#"
echo "All arguments: $@"

if [[ $# -lt 2 ]]; then
    echo "Usage: $0 <environment> <version>"
    exit 1
fi
```

```bash
#!/bin/bash
# --- conditionals ---
read -p "Enter a number: " num

if [[ $num -gt 0 ]]; then
    echo "$num is positive"
elif [[ $num -lt 0 ]]; then
    echo "$num is negative"
else
    echo "$num is zero"
fi

# string comparison
read -p "Enter a color: " color
if [[ "$color" == "red" ]]; then
    echo "Stop!"
elif [[ "$color" == "green" ]]; then
    echo "Go!"
else
    echo "Unknown color: $color"
fi

# file/directory tests — common in DevOps scripts
CONFIG_FILE="/etc/myapp/config.yaml"
if [[ -f "$CONFIG_FILE" ]]; then
    echo "Config file found."
elif [[ -d "/etc/myapp" ]]; then
    echo "Directory exists but config file is missing."
else
    echo "Neither the directory nor the config file exist."
fi

# -z / -n: empty string checks
read -p "Enter your email (optional): " email
if [[ -z "$email" ]]; then
    echo "No email provided."
else
    echo "Email: $email"
fi
```

```bash
#!/bin/bash
# --- exit codes and logical operators ---
ls /tmp > /dev/null
echo "Exit code of ls: $?"                   # 0 on success

ls /nonexistent-directory > /dev/null 2>&1
echo "Exit code of failed ls: $?"             # non-zero on failure

# && and || chaining
mkdir -p /tmp/backup-test && echo "Directory ready" || echo "Failed to create directory"

[[ -f /etc/hostname ]] && echo "/etc/hostname exists"
[[ -f /etc/does-not-exist ]] || echo "/etc/does-not-exist is missing, as expected"

# ! negation
if [[ ! -f "/tmp/lockfile" ]]; then
    echo "No lockfile present, safe to proceed"
fi

# exit with a specific code, useful for scripts other tools/CI check the result of
if [[ $# -eq 0 ]]; then
    echo "Error: no arguments provided" >&2    # >&2: send error message to stderr (Lesson 4)
    exit 1
fi
exit 0
```

## Common Pitfalls

- **Spaces around `=` in variable assignment** — `NAME = "Alex"` is a syntax error (Bash tries to run `NAME` as a command with arguments `=` and `"Alex"`). Must be `NAME="Alex"`, no spaces.
- **Forgetting to quote variables in conditions** — `[[ -f $FILE ]]` breaks if `$FILE` is empty or contains spaces; `[[ -f "$FILE" ]]` is the safe habit to build from day one.
- **Using `==` for numeric comparison, or `-eq` for string comparison** — they're not interchangeable; using the wrong one either fails outright or silently produces wrong results for certain values.
- **Confusing `./script.sh` permission errors with syntax errors** — "Permission denied" means the file isn't executable (`chmod +x` needed); it says nothing about whether the script's contents are actually valid.
- **Checking `$?` too late** — running even one more command (like `echo` to print a message) between the command you care about and checking `$?` overwrites it with that command's own exit code instead.
- **Assuming `read` without `-r` preserves backslashes literally** — by default, `read` interprets backslash escapes in the input; `read -r` disables that, which is usually what you actually want when reading arbitrary user text (worth knowing now, becomes more relevant reading from files in the next lesson).

## FAQ

**Q: What's the actual difference between `[ ]` and `[[ ]]`?**
A: `[ ]` (and `test`) is a POSIX-standard external-ish command, portable to any POSIX shell, but stricter about quoting and lacks some conveniences. `[[ ]]` is a Bash keyword with more forgiving behavior around unquoted variables and extra operators (like `&&`/`||` directly inside, and pattern matching with `==`). In Bash-only scripts, `[[ ]]` is generally preferred.

**Q: Why does my script say "command not found" right after the shebang line?**
A: Usually a Windows-style line ending (`\r\n` instead of `\n`) sneaking into the file — the `\r` gets appended to the shebang path, making it invalid. Save/edit scripts with a Linux-native editor (vim/nano, Lesson 3) or explicitly convert line endings if the file came from a Windows source.

**Q: Do I always need `#!/bin/bash` at the top?**
A: You need it if the script will be run directly (`./script.sh`) so the kernel knows how to interpret it. If you always invoke it explicitly with `bash script.sh`, the shebang is optional (though still good practice for clarity and portability).

**Q: What's the difference between `exit` and just letting the script reach its last line?**
A: If a script has no explicit `exit`, its overall exit code is simply whatever the last command in it returned. `exit N` lets you deliberately signal success/failure with a specific meaning at any point, which matters a lot once other scripts, CI pipelines, or `&&`/`||` chains depend on checking that exit code.

**Q: Why does `read -s` need an extra `echo` afterward?**
A: `-s` suppresses echoing the *input* back to the terminal (so a password isn't visible) — but it also means the terminal cursor doesn't automatically move to a new line the way it normally would after pressing Enter. A plain `echo` after `read -s` just prints a newline so subsequent output doesn't run into the same line.

## Practice / Exercise

**Core:**
1. Write a script with a proper shebang that prints "Hello, World!", make it executable, and run it both as `./script.sh` and `bash script.sh`.
2. Write a script that declares a variable holding your name and one holding a sentence with a space in it; demonstrate the difference between double-quote and single-quote expansion by echoing both quoted and unquoted forms.
3. Write a script using `read -p` to ask for a number, then use an `if`/`elif`/`else` chain to print whether it's positive, negative, or zero.
4. Write a script that checks `$#` and prints a usage message and `exit 1` if fewer than 2 arguments were given; otherwise print both arguments.
5. Write a script that checks (with `-f` and `-d`) whether a given path argument is a file, a directory, or doesn't exist, printing a different message for each case.
6. Write a one-liner using `&&` and `||` that attempts to create a directory and reports success or failure accordingly.

**Stretch:**
1. Write a script using `read -s` to collect a "password," and print its length using `${#variable}` without ever printing the password itself.
2. Deliberately write a script comparing two numbers with `==` instead of `-eq` and find/explain a case where it produces the wrong result.
3. Write a script that intentionally exits with a specific non-zero code (e.g. `exit 42`) under one condition, and confirm with `echo $?` immediately after running it that the code was captured correctly.

## Further Reading

- `man bash` (search for "CONDITIONAL EXPRESSIONS" and "PARAMETERS")
- [Bash Guide (Greg's Wiki)](https://mywiki.wooledge.org/BashGuide)
- [ShellCheck](https://www.shellcheck.net/) — a static analysis tool for catching common shell scripting mistakes (worth introducing early, even before automating its use)
