# Lesson 17: Shell Scripting – Loops & Functions

**Module:** Scripting (Bash & Python)
**Duration:** 120-150 min
**Prerequisites:** Lesson 16 (Shell Scripting Basics & Syntax)

## Learning Objectives

By end of lesson student can:
- Write `for` loops over lists, ranges, and C-style counters
- Write `while`/`until` loops, including reading a file line by line
- Use `break` and `continue` to control loop flow
- Define and call functions, pass arguments, use local variables, and return values
- Declare, index, and iterate over Bash arrays
- Write more robust scripts using `set -e`/`-u`/`-o pipefail` and `trap`

## Topics

- `for` loops: list iteration, C-style `for ((i=0;i<n;i++))`, range with `{1..10}`/`seq`
- `while`/`until` loops: condition-based; read line from file; `break` and `continue`
- Functions: declaration, calling, arguments (`$1` `$2`), local variables, return value
- Arrays: declaration, indexing (`${arr[0]}`), `${arr[@]}`, `${#arr[@]}`, append
- Robust scripting: `set -e`, `set -u`, `set -o pipefail`; `trap` for cleanup

## Concepts

### `for` loops

Bash's `for` loop has two distinct forms. The **list form** iterates over a sequence of words:

```bash
for item in one two three; do
    echo "$item"
done
```

The list can come from a literal word list, a brace range `{1..10}`, the `seq` command (`seq 1 2 10` for stepped ranges), command output, or a glob pattern (`for f in *.txt`).

The **C-style form** mirrors C/Java's classic counting loop, useful when you need explicit control over the counter's start, condition, and increment:

```bash
for ((i=0; i<10; i++)); do
    echo "$i"
done
```

Prefer the list form for iterating over known items (files, arguments, array elements); reach for the C-style form when you specifically need arithmetic control over a counter.

### `while` and `until` loops

`while condition; do ... done` repeats as long as `condition`'s exit code is 0 (success) — same "command as condition" model as `if` from the previous lesson. `until condition; do ... done` is the mirror image: repeats as long as `condition` is *false* (non-zero), stopping once it becomes true.

A very common DevOps pattern is reading a file line by line:

```bash
while IFS= read -r line; do
    echo "Line: $line"
done < input.txt
```

`IFS=` (empty) prevents `read` from trimming leading/trailing whitespace from each line; `-r` disables backslash escape interpretation (from Lesson 16's FAQ) — both matter for reading arbitrary file content faithfully rather than a simple word.

### `break` and `continue`

`break` exits the innermost loop immediately, skipping any remaining iterations. `continue` skips the rest of the *current* iteration only, and moves on to the next one — the loop keeps running, just skips ahead. Both work identically across `for`, `while`, and `until`.

### Functions

A function groups reusable logic under a name. Two equivalent declaration syntaxes exist (`name() { ... }` is more common/portable, `function name { ... }` is a Bash-specific alternative); calling a function is just writing its name, like any other command.

```bash
greet() {
    echo "Hello, $1!"
}
greet "Alex"       # -> Hello, Alex!
```

Inside a function, `$1`, `$2`, `$#`, `$@` refer to the arguments passed to *that function call* — not the script's own original arguments (Lesson 16). By default, variables assigned inside a function are **global** — they leak out and affect the rest of the script, which is usually not what you want. The `local` keyword scopes a variable to just that function call: `local result="value"`.

Bash functions don't have a real "return a value" mechanism like most programming languages — `return N` only returns a *numeric exit code* (0-255, same range as a script's own exit code), used to signal success/failure, not to hand back computed data. To actually get a computed *value* out of a function, the idiomatic approach is to `echo` it and capture that output with command substitution: `result=$(my_function arg1 arg2)`.

### Arrays

Bash supports indexed arrays (numeric keys, starting at 0):

```bash
fruits=("apple" "banana" "cherry")
echo "${fruits[0]}"        # apple — braces are required for array indexing
echo "${fruits[@]}"        # all elements, each as a separate word
echo "${#fruits[@]}"       # 3 — the number of elements
fruits+=("date")            # append a new element
```

`"${fruits[@]}"` (quoted, with `@`) is the correct way to iterate elements safely even if one contains spaces — same principle as `"$@"` for script arguments in the previous lesson.

### Robust scripting: `set -e`, `set -u`, `set -o pipefail`

By default, Bash scripts are surprisingly forgiving of errors — a failing command doesn't stop the script, and a typo'd variable name just silently expands to an empty string. Three options tighten this considerably, and are considered standard practice for any script beyond a quick one-off:

| Option | Effect |
|---|---|
| `set -e` | Exit immediately if any command returns a non-zero exit code (with some nuanced exceptions, e.g. inside an `if` condition, where failure is expected and handled) |
| `set -u` | Treat referencing an undefined variable as an error and exit, instead of silently substituting an empty string |
| `set -o pipefail` | Without this, a pipeline's exit code is only the *last* command's — a failure earlier in the pipe is invisible; this option makes the whole pipeline fail if *any* command in it fails |

These are almost always combined at the top of a script: `set -euo pipefail`.

### `trap`: cleanup on exit or interruption

`trap 'commands' SIGNAL` registers a command (or function call) to run automatically when the script receives a given signal (Lesson 8's territory) or exits — `EXIT` is a special pseudo-signal name meaning "run this no matter how the script ends, success or failure." This is the standard way to guarantee cleanup (removing a temp file, releasing a lock) happens even if the script fails partway through or is interrupted with `Ctrl+C` (`SIGINT`).

## Commands / Syntax Reference

| Syntax | Purpose | Example |
|---|---|---|
| `for x in list; do ... done` | List-form for loop | `for f in *.txt; do echo "$f"; done` |
| `for ((...)); do ... done` | C-style for loop | `for ((i=0;i<5;i++)); do echo "$i"; done` |
| `{1..10}` | Brace range | `for n in {1..10}; do echo "$n"; done` |
| `seq` | Generate a numeric range | `seq 1 2 10` |
| `while cond; do ... done` | While loop | `while [[ $i -lt 5 ]]; do ...; done` |
| `until cond; do ... done` | Until loop | `until [[ $i -ge 5 ]]; do ...; done` |
| `break` | Exit the loop | inside a loop body |
| `continue` | Skip to next iteration | inside a loop body |
| `name() { ... }` | Define a function | `greet() { echo "Hi $1"; }` |
| `local` | Scope a variable to a function | `local x=1` |
| `return N` | Return an exit code from a function | `return 0` |
| `arr=(...)` | Declare an array | `arr=("a" "b" "c")` |
| `${arr[i]}` | Access one element | `${arr[0]}` |
| `${arr[@]}` | All elements | `"${arr[@]}"` |
| `${#arr[@]}` | Array length | `${#arr[@]}` |
| `set -euo pipefail` | Strict mode | top of script |
| `trap` | Register a cleanup handler | `trap cleanup EXIT` |

## Examples / Walkthrough

```bash
#!/bin/bash
# --- for loops ---

# list form: literal words
for env in dev staging production; do
    echo "Deploying to: $env"
done

# list form: brace range
for n in {1..5}; do
    echo "Count: $n"
done

# list form: seq, with a step
for n in $(seq 0 10 50); do
    echo "Step: $n"
done

# list form: iterating over files
for file in /var/log/*.log; do
    echo "Found log file: $file"
done

# C-style form: explicit counter control
for ((i=0; i<5; i++)); do
    echo "i is $i"
done
```

```bash
#!/bin/bash
# --- while / until, break / continue ---

count=0
while [[ $count -lt 5 ]]; do
    echo "count is $count"
    ((count++))
done

count=0
until [[ $count -ge 5 ]]; do
    echo "count is $count"
    ((count++))
done

# reading a file line by line — the standard, correct pattern
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/hostname

# break and continue
for n in {1..10}; do
    if [[ $n -eq 7 ]]; then
        break                          # stop the loop entirely once we hit 7
    fi
    if [[ $((n % 2)) -eq 0 ]]; then
        continue                       # skip even numbers, keep looping
    fi
    echo "Odd number: $n"
done
```

```bash
#!/bin/bash
# --- functions ---

greet() {
    echo "Hello, $1! You are argument number 1 in this function call."
}
greet "Alex"

# local vs global variables
counter=0
increment() {
    local counter=100                  # this "counter" is LOCAL, only exists inside this function
    ((counter++))
    echo "Inside function, counter is: $counter"
}
increment
echo "Outside function, counter is still: $counter"    # unchanged — the outer variable was untouched

# returning a computed VALUE (not just an exit code) via echo + command substitution
add_numbers() {
    local sum=$(( $1 + $2 ))
    echo "$sum"                          # "return" the value by printing it
}
result=$(add_numbers 3 4)                 # capture the printed output
echo "3 + 4 = $result"

# return CODE (0-255) — for signaling success/failure, not data
is_even() {
    if (( $1 % 2 == 0 )); then
        return 0                          # success = "yes, it is even"
    else
        return 1                          # failure = "no, it is not even"
    fi
}
if is_even 4; then
    echo "4 is even"
fi
```

```bash
#!/bin/bash
# --- arrays ---

fruits=("apple" "banana" "cherry")

echo "First fruit: ${fruits[0]}"
echo "All fruits: ${fruits[@]}"
echo "Number of fruits: ${#fruits[@]}"

fruits+=("date")                          # append
echo "After append: ${fruits[@]}"

for fruit in "${fruits[@]}"; do            # always quote "${arr[@]}" when iterating
    echo "Fruit: $fruit"
done

unset 'fruits[1]'                           # remove one element (banana) by index
echo "After removing index 1: ${fruits[@]}"
```

```bash
#!/bin/bash
# --- robust scripting: set -euo pipefail and trap ---
set -euo pipefail

TEMP_FILE=$(mktemp)                          # create a temp file to work with

cleanup() {
    echo "Cleaning up temp file: $TEMP_FILE"
    rm -f "$TEMP_FILE"
}
trap cleanup EXIT                             # runs automatically on ANY script exit, success or failure

echo "Writing to $TEMP_FILE"
echo "some data" > "$TEMP_FILE"

# with "set -e", this failing command would immediately stop the script here:
# ls /this/path/does/not/exist

# with "set -u", referencing an undefined variable would error out immediately:
# echo "$UNDEFINED_VARIABLE"

# with "set -o pipefail", a failure anywhere in a pipeline is caught, not just the last command:
# cat /nonexistent/file | grep "pattern"     # fails loudly instead of grep silently seeing empty input

echo "Script completed successfully"
# "cleanup" runs automatically here via the EXIT trap, even without an explicit call
```

## Common Pitfalls

- **Forgetting `local` inside functions** — variables assigned in a function are global by default; a function that reuses a common name (`i`, `result`, `count`) can silently clobber a variable the calling code was relying on. Default to `local` for anything not deliberately meant to be shared.
- **Trying to `return` a computed value directly** — `return` only accepts 0-255 as an exit code, not arbitrary data. Attempting `return "some string"` or `return 300` either errors or truncates unexpectedly (values wrap modulo 256). Use `echo` + command substitution to actually get data out of a function.
- **Iterating `for x in $(cat file)` instead of a `while read` loop** — command substitution word-splits on whitespace, so this silently breaks on lines containing spaces, treating each word as a separate iteration instead of each line. The `while IFS= read -r line; do ... done < file` pattern is the correct one for line-by-line file processing.
- **Not quoting `"${arr[@]}"` when iterating an array** — without quotes, an element containing spaces gets word-split into multiple loop iterations instead of staying as one item, same class of bug as unquoted `$@`.
- **`set -e` not catching a failure inside a pipeline** — without `pipefail`, `false | true` "succeeds" (exit code reflects only `true`, the last command), masking an earlier failure. Always pair `set -e` with `set -o pipefail` for pipelines.
- **`set -u` breaking on an intentionally-optional variable** — a variable that might legitimately be unset (e.g. an optional environment variable) needs a default: `${VAR:-default_value}` provides a fallback without triggering `set -u`'s error, whereas a bare `$VAR` reference would.

## FAQ

**Q: When should I use `for` vs `while`?**
A: `for` is the natural choice when iterating over a known, finite list (files, array elements, a fixed range). `while` fits better when the number of iterations isn't known ahead of time and depends on a condition changing during the loop (waiting for a service to become ready, reading an unknown number of lines from a file).

**Q: Why doesn't my function's `return` value show up in a variable the way I expected?**
A: `return` sets the function's exit code (checked via `$?` or directly in an `if`), not a data value. If you want to capture computed data, the function must `echo` it, and the caller captures that output with `$(function_name args)`.

**Q: Is `set -euo pipefail` always a good idea?**
A: For almost any script beyond a 2-line throwaway, yes — it catches real bugs (a failed command silently ignored, a typo'd variable name) that would otherwise cause confusing downstream failures. The main adjustment needed is being deliberate about genuinely-optional variables (`${VAR:-default}`) and commands that are *expected* to sometimes fail (wrapped in an `if` or followed by `|| true` when that specific failure is fine).

**Q: What's the difference between `break` and `exit` inside a loop?**
A: `break` only stops the current loop and lets the script continue with whatever comes after it. `exit` stops the entire script immediately, regardless of what loop or function it's called from.

**Q: Why does `trap cleanup EXIT` matter more than just calling `cleanup` at the end of the script?**
A: A plain call at the end only runs if the script reaches that line normally. If `set -e` causes an early exit, or the user hits `Ctrl+C`, or any other unexpected termination happens, a `trap ... EXIT` still fires — a normal function call at the bottom of the script would simply never be reached.

## Practice / Exercise

**Core:**
1. Write a `for` loop (list form) that prints "Processing: <name>" for a list of at least 4 server names.
2. Write a C-style `for` loop that prints only the even numbers from 0 to 20.
3. Write a `while` loop that reads a file (e.g. `/etc/hostname` or any text file) line by line and prints each line with a line number prefix.
4. Write a loop over 1-20 that uses `continue` to skip multiples of 3 and `break` once it reaches 15.
5. Write a function `is_valid_port` that takes one argument and returns success (0) if it's a number between 1 and 65535, failure (1) otherwise; call it from an `if` statement.
6. Write a function that computes and `echo`s the square of a number, and capture its result into a variable using command substitution.
7. Declare an array of at least 4 items, print the whole array, print its length, append one more item, then iterate over it with a `for` loop.
8. Write a script with `set -euo pipefail` at the top and a `trap` that removes a temp file on exit; deliberately make one command inside fail and confirm the cleanup still runs.

**Stretch:**
1. Write a script that reads a list of usernames from a file (one per line) and, for each, prints whether a local system account with that name exists (hint: check `/etc/passwd`, or use `id username` and check its exit code — Lesson 6's material).
2. Write a function using `local` correctly, and then a second, buggy version that omits `local`; demonstrate the difference in behavior by having both touch a variable with the same name as one already used in the calling script.
3. Explain, using a concrete example, exactly why `${VAR:-default}` is needed under `set -u` for an optional variable, but not for one you always explicitly set earlier in the script.

## Further Reading

- `man bash` (search for "Arrays", "Compound Commands", "SHELL BUILTIN COMMANDS")
- [Bash Guide: Loops](https://mywiki.wooledge.org/BashGuide/TestsAndConditionals)
- [ShellCheck](https://www.shellcheck.net/) — flags most of this lesson's common pitfalls automatically
