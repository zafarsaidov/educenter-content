# Lesson 19: Python Basics for DevOps

**Module:** Scripting (Bash & Python)
**Duration:** 120-150 min
**Prerequisites:** Lessons 16-18 (Shell Scripting Basics, Loops & Functions, Practice)

## Learning Objectives

By end of lesson student can:
- Explain when a task is better suited to Python than Bash, and set up an isolated environment with `venv`
- Use Python's core data types and f-strings for formatted output
- Write control flow (`if`/`for`/`while`) and list comprehensions
- Define functions, including `*args`/`**kwargs`, and organize code into importable modules
- Use `os`, `sys`, `subprocess`, and `pathlib` for common DevOps scripting tasks

## Topics

- Python vs Bash: when to use which; running scripts, shebang, venv basics
- Data types: `str`, `int`, `float`, `bool`, `list`, `dict`, `tuple`; `type()`, f-strings, string methods
- Control flow: `if`/`elif`/`else`, `for`/`while` loops, `break`/`continue`, list comprehensions
- Functions: `def`, arguments, `*args`, `**kwargs`, `return`; `import` and modules
- DevOps stdlib: `os` (path, environ, makedirs), `sys` (argv, exit), `subprocess` (run, check_output), `pathlib`

## Concepts

### Python vs Bash: when to reach for which

Bash excels at short, linear sequences of shell commands — piping tools together, quick file manipulation, glue between other programs. It gets awkward fast once a script needs real data structures, error handling with structured information (not just exit codes), or logic beyond a few conditionals. Python is the natural next step for anything involving: parsing structured data (JSON, YAML, CSV) meaningfully, calling APIs, complex branching logic, or code that needs to be tested and maintained by a team over time. A rough rule of thumb: if a Bash script would need more than a couple of associative-array-style workarounds or nested loops with complex conditions, it's often a sign Python would be clearer.

Neither replaces the other — in real DevOps work, Bash and Python scripts sit side by side, each used where it's the better tool.

### Running Python scripts, and virtual environments

A Python script also uses a shebang (`#!/usr/bin/env python3`) and can be made executable with `chmod +x` (Lesson 16), or simply run with `python3 script.py`. What's different from Bash is **dependencies**: Python scripts often need third-party packages (installed via `pip`), and installing those globally on a machine risks version conflicts between different projects' needs.

A **virtual environment** (`venv`) solves this: it's an isolated copy of the Python interpreter and its own package directory, scoped to one project. `python3 -m venv .venv` creates one; `source .venv/bin/activate` makes it the active Python for the current shell session (note: this `source` — same command from Lesson 18 — actually modifies your shell's `$PATH` so `python3`/`pip` point inside `.venv/`); `deactivate` returns to the system Python. Packages installed with `pip install` while a venv is active only affect that venv, leaving the system Python and every other project's venv untouched.

### Data types

Python is dynamically typed — a variable's type is determined by what's assigned to it, not declared up front. Core built-in types:

| Type | Example | Notes |
|---|---|---|
| `str` | `"hello"` | Text; immutable |
| `int` | `42` | Whole numbers, arbitrary precision |
| `float` | `3.14` | Decimal numbers |
| `bool` | `True`, `False` | Capitalized, unlike Bash's lowercase concept of true/false |
| `list` | `[1, 2, 3]` | Ordered, mutable, can hold mixed types |
| `dict` | `{"key": "value"}` | Key-value pairs, the workhorse for structured data (JSON maps directly onto it) |
| `tuple` | `(1, 2)` | Ordered, **immutable** — used for fixed, small groupings that shouldn't change |

`type(x)` returns a value's type at runtime — useful for debugging when a variable's type isn't obvious.

**f-strings** (`f"..."`) are Python's modern string formatting: `f"Hostname: {hostname}, load: {load:.2f}"` embeds expressions directly inside `{}`, optionally with formatting specifiers (like `.2f` for 2 decimal places) — the direct Python equivalent of Bash's `"$VAR"` interpolation, but considerably more capable.

Useful string methods: `.strip()` (remove leading/trailing whitespace — the Python equivalent of trimming a line read from a file), `.split()` (break a string into a list on a delimiter), `.join()` (the reverse — combine a list into one string), `.lower()`/`.upper()`, `.replace()`.

### Control flow

```python
if condition:
    ...
elif other_condition:
    ...
else:
    ...
```

Note: no `then`/`fi` like Bash — Python uses **indentation itself** to define blocks (conventionally 4 spaces), which is significant, not just cosmetic; inconsistent indentation is a syntax error, not a style nit.

`for` loops iterate directly over a sequence's items — there's no C-style counting form needed for the common case:

```python
for item in ["dev", "staging", "prod"]:
    print(item)

for i in range(5):          # range(5) generates 0,1,2,3,4 — Python's equivalent of Bash's {0..4}
    print(i)
```

`while` works the same conceptually as Bash's, condition-based. `break`/`continue` behave identically to their Bash counterparts (Lesson 17).

**List comprehensions** are a compact, idiomatic way to build a new list from an existing iterable in one line: `[x * 2 for x in range(5)]` produces `[0, 2, 4, 6, 8]`; `[x for x in items if condition]` filters while building. They're not strictly necessary (a regular `for` loop building a list works too) but are extremely common in real Python code, worth reading fluently even before writing them confidently.

### Functions

```python
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"
```

Parameters can have **default values** (`greeting="Hello"`), making them optional at the call site. `*args` collects any number of extra positional arguments into a tuple; `**kwargs` collects any number of extra named arguments into a dict — both used when a function needs to accept a flexible, variable number of inputs.

Unlike Bash functions (Lesson 17), Python's `return` hands back an actual value of any type directly — no need for the "echo and capture with command substitution" workaround Bash requires.

### Modules and `import`

Any `.py` file is itself a **module** — its functions/variables become available elsewhere with `import module_name` (bare name, no `.py` extension). This is Python's equivalent of Bash's `source`, but with an important difference: `import` does **not** automatically run everything at the top level of the imported file as if typed inline — it runs the module once, and only names explicitly used (`module_name.function()`) are accessed, keeping each file's own variables from silently colliding, unlike a Bash `source`.

### The DevOps standard library trio: `os`, `sys`, `subprocess`, `pathlib`

| Module | Purpose |
|---|---|
| `os` | Operating system interaction: environment variables (`os.environ`), path joining (`os.path.join`, though `pathlib` is now generally preferred), creating directories (`os.makedirs`) |
| `sys` | Interpreter-level access: `sys.argv` (command-line arguments, Python's equivalent of Bash's `$1`/`$@`), `sys.exit(code)` (equivalent of Bash's `exit N`) |
| `subprocess` | Running external commands from Python — `subprocess.run()` for most cases, capturing output/exit code; this is how a Python script shells out to `ls`, `df`, `curl`, or any other external tool when Python itself doesn't have a built-in equivalent |
| `pathlib` | Modern, object-oriented filesystem path handling (`Path("/var/log") / "app.log"`) — the current recommended replacement for most `os.path` string-based path manipulation |

## Commands / Syntax Reference

| Syntax | Purpose | Example |
|---|---|---|
| `python3 -m venv .venv` | Create a virtual environment | `python3 -m venv .venv` |
| `source .venv/bin/activate` | Activate it | `source .venv/bin/activate` |
| `deactivate` | Return to system Python | `deactivate` |
| `pip install` | Install a package into the active venv | `pip install requests` |
| `type(x)` | Get a value's type | `type(42)` |
| `f"..."` | Formatted string | `f"Value: {x}"` |
| `def name(args):` | Define a function | `def add(a, b): return a + b` |
| `import module` | Load a module | `import os` |
| `sys.argv` | Script's command-line arguments | `sys.argv[1]` |
| `subprocess.run()` | Run an external command | `subprocess.run(["ls", "-l"])` |
| `Path(...)` | Build a filesystem path | `Path("/var/log") / "app.log"` |

## Examples / Walkthrough

```bash
# --- venv setup ---
python3 -m venv .venv
source .venv/bin/activate                 # prompt changes to show (.venv) is active
pip install requests                        # installed only inside this venv
python3 --version
deactivate                                   # back to system Python
```

```python
#!/usr/bin/env python3
# --- data types, f-strings, string methods ---

hostname = "web-01"
cpu_load = 0.52
is_alerting = False
tags = ["prod", "web", "eu-west"]
config = {"interval": 30, "threshold": 80}

print(f"Host: {hostname}, load: {cpu_load:.1%}, alerting: {is_alerting}")
print(f"Tags: {', '.join(tags)}")           # join a list into one string
print(type(cpu_load))                        # <class 'float'>

raw_line = "  disk_usage=87  \n"
clean = raw_line.strip()                      # remove leading/trailing whitespace and the newline
key, value = clean.split("=")                  # split on "=", unpack into two variables
print(f"key={key!r} value={value!r}")
```

```python
#!/usr/bin/env python3
# --- control flow ---

cpu_load_pct = 85

if cpu_load_pct >= 90:
    print("CRITICAL")
elif cpu_load_pct >= 70:
    print("WARNING")
else:
    print("OK")

# for loop over a list
environments = ["dev", "staging", "production"]
for env in environments:
    print(f"Checking {env}...")

# for loop over a range (Bash's {0..4} equivalent)
for i in range(5):
    print(f"Attempt {i}")

# while loop with break
attempts = 0
while True:
    attempts += 1
    print(f"Attempt {attempts}")
    if attempts >= 3:
        break

# list comprehension: build a new list in one line
squares = [x * x for x in range(10)]
print(squares)

high_load_hosts = [h for h in [("web-01", 45), ("web-02", 92), ("db-01", 88)] if h[1] >= 80]
print(high_load_hosts)                          # only the hosts with load >= 80
```

```python
#!/usr/bin/env python3
# --- functions ---

def check_threshold(value, limit=80):
    """Return True if value crosses the limit."""
    return value >= limit

print(check_threshold(85))                       # uses default limit=80 -> True
print(check_threshold(85, limit=90))               # explicit limit -> False

def summarize(*services, **details):
    print(f"Services: {services}")                  # a tuple of positional args
    print(f"Details: {details}")                     # a dict of keyword args

summarize("nginx", "postgres", region="eu-west", replicas=3)
```

```python
#!/usr/bin/env python3
# --- os, sys, subprocess, pathlib ---
import os
import sys
import subprocess
from pathlib import Path

# sys.argv: command-line arguments, like Bash's $1/$@
if len(sys.argv) < 2:
    print("Usage: script.py <environment>")
    sys.exit(1)

environment = sys.argv[1]
print(f"Running against: {environment}")

# os.environ: read environment variables
home_dir = os.environ.get("HOME", "/tmp")         # .get() with a default, avoids a KeyError if unset
print(f"Home directory: {home_dir}")

log_dir = Path(home_dir) / "app-logs"               # pathlib: join paths with /, not string concatenation
os.makedirs(log_dir, exist_ok=True)                  # exist_ok=True: don't error if it already exists
print(f"Log directory ready: {log_dir}")

# subprocess.run: shell out to an external command, capture its output
result = subprocess.run(["df", "-h", "/"], capture_output=True, text=True)
print("Exit code:", result.returncode)
print("Output:\n", result.stdout)

if result.returncode != 0:
    print("df command failed:", result.stderr, file=sys.stderr)
    sys.exit(1)
```

## Common Pitfalls

- **Mixing tabs and spaces for indentation** — Python's indentation is meaningful; a file mixing tabs and spaces raises a `TabError` or, worse, silently produces different block structure than it visually appears to have. Configure your editor (vim/nano, Lesson 3) to always insert spaces for Python files.
- **Forgetting a venv is per-shell-session** — activating a venv only affects the current terminal session; a new terminal tab/window starts back at the system Python until `source .venv/bin/activate` is run there too.
- **Using `os.path` string concatenation instead of `pathlib`** — manually joining paths with `+` or forgetting a `/` separator is a classic source of bugs; `pathlib`'s `Path(a) / b` handles separators correctly and works the same across operating systems.
- **Not checking `subprocess.run()`'s return code** — by default, a failed external command doesn't raise an exception; `result.returncode` must be checked explicitly (or pass `check=True` to `subprocess.run()`, which raises `CalledProcessError` automatically on a non-zero exit).
- **Assuming `**kwargs`/`*args` are required** — a function using them can still be called normally with regular named arguments; `*args`/`**kwargs` only matter when the *caller* wants to pass a variable number of extra arguments.
- **Reassigning a built-in name** — naming a variable `list`, `str`, `type`, or `dict` shadows Python's own built-in of that name for the rest of the current scope, causing confusing errors later if that built-in is needed again.

## FAQ

**Q: Why does Python use indentation instead of `{ }` or `fi`/`done` like other languages?**
A: It's a deliberate design choice to force readable code — since indentation has to be consistent to be valid at all, there's no way for a script's visual structure to lie about its actual logical structure, unlike languages where indentation is just a style convention separate from the real block delimiters.

**Q: Do I need a venv for every single script?**
A: Not for a script using only the standard library (`os`, `sys`, `subprocess`, `pathlib` — everything in this lesson needs zero external packages). A venv matters once a script needs a third-party package via `pip install`, to keep that dependency isolated to this one project.

**Q: What's the practical difference between a `list` and a `tuple`?**
A: Both are ordered collections, but a `list` is mutable (items can be added/removed/changed after creation) and a `tuple` is immutable (fixed once created). Use a tuple for a small, fixed grouping that represents one conceptual "record" (like a coordinate pair), and a list for anything you'll be growing/modifying.

**Q: Why use `subprocess.run()` instead of just writing the equivalent logic in pure Python?**
A: Sometimes there simply isn't a pure-Python equivalent worth reimplementing — calling `df`, `systemctl`, or any other established CLI tool via `subprocess` is often more reliable and far less code than reimplementing that tool's logic in Python from scratch.

**Q: Is `import` the same thing as Bash's `source`?**
A: Conceptually similar (both bring another file's code into scope), but `import` is safer: it runs the imported module exactly once no matter how many times it's imported elsewhere, and requires explicitly prefixing accessed names (`module.function()`) rather than dumping everything directly into the current namespace the way `source` does.

## Practice / Exercise

**Core:**
1. Create and activate a virtual environment, confirm with `python3 --version` (and `which python3`) that it's using the venv's interpreter, then deactivate it.
2. Write a script that declares one variable of each core type (`str`, `int`, `float`, `bool`, `list`, `dict`, `tuple`) and prints each one's value and its `type()`.
3. Write a script using an f-string to print a formatted sentence combining at least 3 variables, one of them a float formatted to 2 decimal places.
4. Write a `for` loop over a list of at least 4 server names printing a status line for each, and a separate loop using `range()` to print numbers 0 through 9.
5. Write a list comprehension that filters a list of numbers to only those greater than 50.
6. Write a function with a default argument and a second version of a call that overrides the default; write a second function using `*args` to accept any number of arguments and print how many were given.
7. Write a script using `sys.argv` to accept a required argument, `os.environ` to read an environment variable with a default fallback, and `pathlib.Path` to construct and create a directory path.
8. Write a script using `subprocess.run()` to run `df -h`, capture its output, and print only the exit code and the first line of output.

**Stretch:**
1. Write a small module (`utils.py`) with 2 reusable functions, then write a second script that `import`s it and calls both functions.
2. Rewrite one of Lesson 18's homework scripts (system info report, or log analyzer) in Python instead of Bash, and note which parts felt easier or harder than the Bash version.
3. Use `subprocess.run(..., check=True)` deliberately against a command that will fail, and observe/explain the exception it raises compared to checking `returncode` manually.

## Further Reading

- [The Python Tutorial (official)](https://docs.python.org/3/tutorial/)
- [Python `venv` documentation](https://docs.python.org/3/library/venv.html)
- [Python `subprocess` documentation](https://docs.python.org/3/library/subprocess.html)
- [Real Python: pathlib](https://realpython.com/python-pathlib/)
