# Lesson 4: File Search, Text Filtering & awk

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-3 (terminal basics, paths, permissions, viewing files)

## Learning Objectives

By end of lesson student can:
- locate files by name, type, size, age, or owner using `find`
- run `find -exec` to act on matched files
- search file content with `grep`, using common flags and basic regex
- chain commands with pipes, and redirect stdout/stderr to files or `/dev/null`
- use `xargs` and `tee` to build multi-step command pipelines
- extract and process columnar text data with `awk`

## Topics

- find: -name, -type (f/d/l), -size, -mtime, -user, -exec
- grep: -r, -i, -n, -v, -l, -c, -E; basic regex in grep
- Piping and redirection: |, > (overwrite), >> (append), 2> (stderr), < (stdin), /dev/null, tee; xargs
- awk basics: fields ($1, $2, $NF), NR, FS, print, printf
- awk patterns and conditions: /pattern/ matching, if/else, practical one-liners

## Concepts

### find — locating files by attribute

`find [path] [tests] [action]` — walks a directory tree and matches files against tests. Default path is current directory if omitted; default action is just printing the path.

**Common tests:**

| Test | Matches | Example |
|---|---|---|
| `-name "pattern"` | filename, case-sensitive, supports `*`/`?` glob wildcards | `find . -name "*.log"` |
| `-iname "pattern"` | filename, case-insensitive | `find . -iname "readme*"` |
| `-type f` | regular files only | `find /etc -type f` |
| `-type d` | directories only | `find /var -type d` |
| `-type l` | symbolic links only | `find /usr -type l` |
| `-size +10M` | larger than 10 MB (`+` over, `-` under, no sign = exact) | `find / -size +100M` |
| `-size -1k` | smaller than 1 KB | `find . -size -1k` |
| `-mtime -1` | modified within last 1 day | `find /var/log -mtime -1` |
| `-mtime +7` | modified more than 7 days ago | `find /tmp -mtime +7` |
| `-user name` | owned by given user | `find /home -user alice` |
| `-perm 777` | exact permission match | `find . -perm 777` |

Combining tests (implicit AND between tests): `find /var/log -type f -name "*.log" -mtime -7` — files, named `*.log`, modified in last week.

**-exec — act on every match:**

`find [tests] -exec command {} \;` — `{}` is replaced by each matched path, one at a time; `\;` (escaped semicolon) terminates the command. `+` instead of `\;` batches multiple matches into fewer command invocations (faster for commands that accept many arguments, like `rm`).

```bash
find . -name "*.tmp" -exec rm {} \;        # delete one at a time
find . -name "*.tmp" -exec rm {} +          # batch into fewer rm calls
find . -type f -name "*.sh" -exec chmod +x {} \;   # make all scripts executable
```

**`-delete`** is a shorthand alternative to `-exec rm {} \;` specifically for deletion — faster, but be very careful: always test with a plain `find` (no `-delete`) first to confirm exactly what would be removed.

### grep — searching file content

`grep [options] "pattern" file` — prints lines matching a pattern.

| Flag | Meaning |
|---|---|
| `-r` (or `-R`) | recursive — search all files under a directory |
| `-i` | case-insensitive |
| `-n` | show line numbers |
| `-v` | invert match — show lines that DON'T match |
| `-l` | list only filenames with a match, not the matching lines themselves |
| `-c` | count matching lines instead of printing them |
| `-E` | extended regex (enables `+`, `?`, `|`, `()` without backslash-escaping) |
| `-w` | match whole word only, not substring |
| `-A N` / `-B N` / `-C N` | show N lines of context After / Before / around (Context) each match |

```bash
grep "error" app.log                    # basic search
grep -i "ERROR" app.log                  # case-insensitive
grep -n "error" app.log                   # with line numbers
grep -v "debug" app.log                    # everything except debug lines
grep -r "TODO" ~/project                    # recursive across a whole directory
grep -l "error" *.log                        # which files contain "error"
grep -c "error" app.log                       # how many lines match
grep -E "error|warning" app.log                # basic regex OR, needs -E or escaped |
```

**Basic regex in grep:** `.` (any single char), `*` (zero or more of previous char), `^` (start of line), `$` (end of line), `[abc]` (character class), `[^abc]` (negated class), `[0-9]` (range). Without `-E`, some metacharacters (`+`, `?`, `|`, `()`) need backslash-escaping to have special meaning — `-E` (or `grep -P` for Perl-compatible regex) removes that friction. Regex depth expands significantly in a later lesson; today's goal is just recognizing and using the basics confidently.

### Piping and redirection

Every process has 3 standard streams, referenced by file descriptor number:

- **stdin** (0) — input a program reads
- **stdout** (1) — normal output
- **stderr** (2) — error output, separate from stdout so errors can be handled independently

**Redirection operators:**

| Operator | Effect |
|---|---|
| `>` | redirect stdout to a file, **overwriting** it |
| `>>` | redirect stdout to a file, **appending** |
| `2>` | redirect stderr only |
| `2>>` | append stderr |
| `&>` or `> file 2>&1` | redirect both stdout and stderr to same file |
| `2>/dev/null` | discard stderr entirely — `/dev/null` is a special device that discards anything written to it |
| `<` | redirect a file's content in as stdin |

```bash
ls /etc > listing.txt              # stdout to file, overwrite
ls /etc >> listing.txt              # stdout to file, append
ls /nonexistent 2> errors.txt        # only the error message goes to file
ls /etc /nonexistent > out.txt 2> err.txt   # split stdout and stderr separately
ls /nonexistent 2>/dev/null           # silently discard the error, script continues clean
command < input.txt                     # feed input.txt as stdin to command
```

**Piping (`|`)** — connects one command's stdout directly to the next command's stdin, without an intermediate file. This is the core idea behind "do one thing well, combine small tools" (Unix philosophy).

```bash
cat access.log | grep "404"                 # only 404 lines
cat access.log | grep "404" | wc -l           # count how many 404s
ls -la | grep "^d"                              # only directories (regex: line starts with d)
```

**`tee`** — reads stdin, writes it BOTH to stdout (so it continues down the pipe) AND to a file, simultaneously. Useful when you want to both see output live and save it.

```bash
cat access.log | grep "error" | tee errors-found.txt
# errors-found.txt now has a copy, AND matching lines still printed to terminal
```

**`xargs`** — takes lines from stdin and converts them into **arguments** for another command. Needed because many commands (like `find`) print a list of things, but you often want to run a separate command *on* each item — `xargs` bridges that gap when `-exec` isn't available or convenient.

```bash
find . -name "*.tmp" | xargs rm                  # delete every matched file
find . -name "*.log" -mtime +30 | xargs rm -f      # batch-delete old logs
echo "one two three" | xargs -n1 echo               # -n1: one argument per invocation
find . -name "*.txt" | xargs grep "TODO"              # search inside all matched files
```

Difference between `find -exec` and `find | xargs`: functionally similar for simple cases; `xargs` is more general-purpose (works with output from any command, not just `find`), while `-exec` stays entirely within `find`'s own syntax. `xargs` also batches many arguments into fewer command invocations by default (faster for huge lists), similar to `find -exec ... +`.

### awk basics

`awk` is a pattern-scanning and text-processing language — reads input line by line, automatically splits each line into **fields**, and lets you act on those fields.

**Fields and built-in variables:**

- `$0` — the whole current line
- `$1`, `$2`, ... — first field, second field, etc. (split by whitespace by default)
- `$NF` — the **last** field (NF = Number of Fields on current line); `$(NF-1)` = second-to-last
- `NR` — current line/record Number (running count as awk processes input)
- `NF` — Number of Fields on the current line
- `FS` — Field Separator (default: any whitespace); change with `-F` flag or `FS="..."` assignment

```bash
echo "alice 25 engineer" | awk '{print $1}'          # alice
echo "alice 25 engineer" | awk '{print $2, $3}'       # 25 engineer
echo "alice 25 engineer" | awk '{print $NF}'           # engineer (last field)

awk '{print NR, $0}' file.txt                            # prefix every line with its line number
awk -F: '{print $1}' /etc/passwd                           # custom field separator (colon), print usernames
awk 'BEGIN{FS=":"} {print $1, $7}' /etc/passwd                # username and login shell

awk '{printf "%-10s %s\n", $1, $2}' data.txt                    # printf for aligned/formatted output
```

`print` adds a newline and separates arguments with a space (or `OFS`, output field separator) automatically. `printf` gives full manual control over formatting (like C's printf) — no automatic newline, you add `\n` yourself.

### awk patterns and conditions

awk programs are really a series of `pattern { action }` pairs — for every input line, each pattern is tested, and if it matches, its action runs.

```bash
awk '/error/ {print}' app.log                # pattern = regex; only matching lines run print
awk '/error/' app.log                          # action defaults to {print} if omitted entirely
awk '!/debug/' app.log                          # negation — print lines NOT matching debug
```

**BEGIN and END blocks** — special patterns that run once before any input is read (`BEGIN`) or once after all input is processed (`END`) — useful for setup (like setting `FS`) and summaries (like totals).

```bash
awk 'BEGIN{count=0} /error/{count++} END{print "Total errors:", count}' app.log
```

**Conditionals inside actions:**

```bash
awk '{ if ($3 > 100) print $1, "is over limit" }' data.txt
awk '{ if (NR % 2 == 0) print $0 }' file.txt    # print only even-numbered lines
awk '$2 > 50 {print $1}' scores.txt               # pattern IS the condition, no explicit if needed
```

**Practical one-liners worth demoing live:**

```bash
awk '{sum += $2} END {print sum}' sales.txt              # sum a column
awk '{sum += $2; count++} END {print sum/count}' sales.txt  # average a column
awk 'length($0) > 80' file.txt                              # lines longer than 80 chars
```

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `find path -name X` | find by filename pattern | `find . -name "*.log"` |
| `find path -type f/d/l` | filter by type | `find . -type d` |
| `find path -size +N` | filter by size | `find . -size +10M` |
| `find path -mtime N` | filter by modified time (days) | `find . -mtime -1` |
| `find path -user X` | filter by owner | `find /home -user alice` |
| `find ... -exec cmd {} \;` | run command per match | `find . -name "*.tmp" -exec rm {} \;` |
| `find ... -delete` | delete matches directly | `find . -name "*.tmp" -delete` |
| `grep pattern file` | search content | `grep "error" app.log` |
| `grep -rniE` (combined flags) | recursive, case-insensitive, line numbers, extended regex | `grep -rniE "err(or)?" .` |
| `\|` | pipe stdout to next command | `cat f \| grep x` |
| `>` / `>>` | redirect stdout (overwrite/append) | `cmd > out.txt` |
| `2>` / `2>>` | redirect stderr | `cmd 2> err.txt` |
| `2>/dev/null` | discard stderr | `cmd 2>/dev/null` |
| `<` | redirect stdin from file | `cmd < in.txt` |
| `tee file` | split stdout: file + still print | `cmd \| tee out.txt` |
| `xargs cmd` | turn stdin lines into arguments | `find . -name "*.tmp" \| xargs rm` |
| `awk '{print $N}'` | print field N | `awk '{print $1}' file` |
| `awk -F sep` | set field separator | `awk -F: '{print $1}' /etc/passwd` |
| `awk '/pat/{action}'` | pattern-matched action | `awk '/error/{print}' log` |

## Examples / Walkthrough

```bash
# ── find: basics ──────────────────────────────────────────
mkdir -p ~/lab4/sub1 ~/lab4/sub2
cd ~/lab4
touch report.txt notes.log sub1/data.log sub2/old.tmp
find . -name "*.log"
find . -type f
find . -type d
find /etc -maxdepth 1 -type f     # only top-level files, no subdir recursion

# ── find: size and time ───────────────────────────────────
find /var/log -type f -size +1k        # log files over 1KB
find /var/log -type f -mtime -1         # modified in the last day
find /var/log -type f -mtime +30         # older than 30 days

# ── find: -exec and -delete ───────────────────────────────
find . -name "*.tmp"                      # preview what will match — ALWAYS do this first
find . -name "*.tmp" -exec rm {} \;        # now actually delete
find . -name "*.log" -exec chmod 644 {} \;   # fix permissions on all logs

# ── grep basics ────────────────────────────────────────────
cat /etc/passwd
grep "bash" /etc/passwd                  # users whose shell is bash
grep -i "BASH" /etc/passwd                 # case-insensitive, same result
grep -n "root" /etc/passwd                  # with line number
grep -v "nologin" /etc/passwd                 # exclude accounts with nologin shell
grep -c "bash" /etc/passwd                     # just the count
grep -rl "root" /etc/*.conf 2>/dev/null          # which config files mention "root", suppress permission errors

# ── redirection ────────────────────────────────────────────
ls -la /etc > /tmp/etc-listing.txt
head -n 5 /tmp/etc-listing.txt
ls -la /etc >> /tmp/etc-listing.txt      # append same listing again
wc -l /tmp/etc-listing.txt                # now double the lines

ls /no/such/path 2> /tmp/errors.txt        # capture just the error
cat /tmp/errors.txt
ls /etc /no/such/path 2>/dev/null            # suppress the error entirely, only real output shown

# ── piping ──────────────────────────────────────────────────
cat /etc/passwd | grep "bash"
cat /etc/passwd | grep "bash" | wc -l
ls -la /etc | grep "^d"                        # only directory entries (regex: line starts with d)

# ── tee ───────────────────────────────────────────────────────
cat /etc/passwd | grep "bash" | tee ~/lab4/bash-users.txt
cat ~/lab4/bash-users.txt                         # confirm it got saved

# ── xargs ───────────────────────────────────────────────────
find ~/lab4 -name "*.log"
find ~/lab4 -name "*.log" | xargs cat              # print content of every matched file
find ~/lab4 -name "*.log" | xargs ls -l               # long listing of every matched file
find ~/lab4 -name "*.log" | xargs rm                    # bulk delete

# ── awk basics ────────────────────────────────────────────
echo "alice 30 engineer" | awk '{print $1}'
echo "alice 30 engineer" | awk '{print $2, $3}'
echo "alice 30 engineer" | awk '{print $NF}'

awk -F: '{print $1}' /etc/passwd                  # usernames only
awk -F: '{print $1, $7}' /etc/passwd                # username + login shell
awk -F: '{print NR, $1}' /etc/passwd                  # numbered list of usernames

# ── awk patterns/conditions ───────────────────────────────
awk -F: '/bash/{print $1}' /etc/passwd            # username only where line matches "bash"
awk -F: '$7 == "/bin/bash" {print $1}' /etc/passwd   # exact field comparison
awk -F: '{ if ($3 > 1000) print $1, $3 }' /etc/passwd  # UID field over 1000 (non-system users)
awk -F: 'BEGIN{count=0} /bash/{count++} END{print "bash users:", count}' /etc/passwd

# ── cleanup ───────────────────────────────────────────────
cd ~
rm -r lab4
rm /tmp/etc-listing.txt /tmp/errors.txt
```

## Common Pitfalls

- **Forgetting to quote `-name` patterns** — `find . -name *.log` (unquoted) lets the *shell* expand `*.log` before `find` even sees it, matching only files in the current directory rather than letting `find` do the recursive glob matching itself. Always quote: `-name "*.log"`.
- **Running `find -exec rm` without previewing first** — always run the plain `find` (no `-exec`) first to see exactly what would match, before attaching a destructive action.
- **Forgetting `\;` after `-exec`** — `find` requires the terminator; forgetting it (or not escaping the semicolon) throws a syntax error.
- **`grep` pattern needing `-E` or escaping** — `grep "a|b"` searches literally for the string `a|b` (pipe character), NOT "a or b", unless you use `-E` or escape it as `a\|b`. Very common source of "why isn't my regex working" confusion.
- **Confusing `>` and `>>`** — `>` silently destroys existing file content; always double-check before overwriting a file you care about, especially in scripts run repeatedly.
- **Redirecting stdout when you meant stderr (or vice versa)** — `command > file.txt` will NOT capture error messages; they still print to terminal. Need `2>` (or `2>&1`) specifically to capture errors too.
- **Piping `ls` output and expecting it to be script-safe** — parsing `ls` output with `grep`/`awk` is fine for interactive exploration (as done in today's lesson) but becomes fragile for filenames with spaces/special characters in real scripts; better tools exist for that (`find -print0` + `xargs -0`) — worth a passing mention, not needed to master today.
- **`xargs` on filenames with spaces** — by default, `xargs` splits on whitespace, so a filename with a space is treated as two separate arguments and breaks. Fix (advanced, mention only): `find ... -print0 | xargs -0 ...`.
- **awk field separator confusion** — forgetting `-F:` when parsing colon-delimited files like `/etc/passwd` — default separator is whitespace, so `$1` on a passwd line would just be the entire line (since there's no whitespace splitting it).
- **Confusing `awk`'s `print` with `printf`** — `print` auto-adds spaces between arguments and a trailing newline; `printf` does neither automatically — a common source of squished-together or missing-newline output when switching between the two.
- **grep vs awk confusion** — `grep` finds/filters whole *lines* matching a pattern; `awk` is for extracting/transforming specific *fields* within lines. Overlap exists (`awk` can pattern-match too) but reach for `grep` for "does this line contain X", `awk` for "give me column 3 of every line".

## FAQ

**Q: What's the difference between `find` and `grep`?**
A: `find` locates *files* based on metadata (name, size, type, modification time, owner) — it doesn't look inside file content at all. `grep` searches *inside* file content for text matching a pattern. Use `find` to answer "which files exist matching X", `grep` to answer "which lines/files contain X text".

**Q: Why did `grep "a|b" file` not work like I expected?**
A: Plain `grep` treats `|` as a literal pipe character in the search string, not a regex "or". You need `-E` (extended regex) or `grep -P` (Perl regex), or escape it manually as `a\|b`, to get "match a OR b" behavior.

**Q: What's the actual difference between `>` and `>>`?**
A: `>` opens the target file and truncates it to zero length before writing — anything previously in the file is gone. `>>` opens the file in append mode — new output is added to the end, existing content stays intact.

**Q: Why would I ever want to throw output away with `/dev/null`?**
A: When a command's error messages (or output generally) are noise you don't care about — e.g. suppressing "Permission denied" spam while grepping through system directories you only have partial access to, so the real results aren't buried.

**Q: What's the difference between piping into `xargs` vs using `find -exec`?**
A: Functionally similar for simple per-file commands. `-exec` is `find`-specific syntax; `xargs` works with the output of *any* command, not just `find`, and by default batches many inputs into fewer command invocations (faster for large lists). Learn `-exec` first since it's simpler for `find`-only cases; reach for `xargs` once piping multiple tools together.

**Q: How does `awk` know where one field ends and the next begins?**
A: By the Field Separator (`FS`), whitespace by default (any amount of spaces/tabs treated as one separator). Change it with `-F` on the command line (e.g. `-F:` for colon-separated files like `/etc/passwd`) or `FS="..."` inside a `BEGIN` block.

**Q: What does `$NF` mean in awk, and why is it useful?**
A: `NF` is the *count* of fields on the current line; `$NF` dereferences that count as a field number, giving you the *last* field — useful when the number of fields varies per line and you just want "whatever's at the end" (e.g. `ls -l` output where the filename is always last but earlier columns can shift).

**Q: When would I use `awk` instead of `grep`?**
A: `grep` answers "which lines match a pattern" (whole-line filtering). `awk` answers "give me specific *columns/fields* from matching or all lines", or lets you compute (sums, averages, counts) across those fields. If you just need to filter lines, `grep` is simpler; if you need to extract or calculate from structured columns, `awk` is the right tool.

**Q: My `find -exec rm {} \;` command gave a syntax error — what did I do wrong?**
A: Most likely you forgot to escape the semicolon (just typing `;` instead of `\;` lets the shell interpret it as a command separator before `find` sees it), or forgot the `{}` placeholder entirely. Double-check both are present and the semicolon is escaped (or in quotes: `';'`).

**Q: What's the difference between `tee` and just using `>`?**
A: `> file` sends output ONLY to the file — nothing appears on screen. `tee file` sends output to BOTH the file AND the screen simultaneously, letting you watch progress live while still saving a copy.

## Practice / Exercise

**Core (everyone should finish):**
1. Under a lab directory, create a mix of `.txt`, `.log`, and `.tmp` files across nested subdirectories; use `find` to list only the `.log` files.
2. Use `find -type d` to list only directories in that lab tree.
3. Use `find -mtime -1` against `/var/log` to find anything modified in the last day.
4. Use `grep -n` to find and line-number every line in `/etc/passwd` containing `bash`.
5. Use `grep -v` to list every line in `/etc/passwd` that does NOT mention `nologin`.
6. Redirect the output of `ls -la /etc` into a file with `>`, then run it again with `>>` and confirm the file doubled in length (`wc -l`).
7. Run a command against a path you know doesn't exist, redirecting only stderr to a file with `2>`; confirm the file contains the error text.
8. Pipe `/etc/passwd` through `grep` then `wc -l` to count matching lines in one chained command.
9. Use `find | xargs rm` to bulk-delete every `.tmp` file in your lab directory (preview with plain `find` first).
10. Using `awk -F:`, print just the username field for every line of `/etc/passwd`.
11. Using `awk -F:`, print username and login shell (`$1` and last field) together.

**Stretch (fast finishers):**
12. Use `find -exec chmod +x {} \;` to make every `.sh` file in a directory executable in one command.
13. Write a `grep -E` command using `|` (OR) to match lines containing either "error" or "warning".
14. Build a 3-stage pipeline: `cat` a file → `grep` for a pattern → `tee` into a results file, confirming the results still print live AND get saved.
15. Using `awk`, compute the count of `/etc/passwd` entries whose UID (3rd field) is 1000 or greater (typically real human user accounts vs system accounts).
16. Using `awk` with a `BEGIN`/`END` block, count how many lines in `/etc/passwd` have `/bin/bash` as their shell and print a one-line summary.
17. Explain (in words, no need to run it) why `find . -name "*.txt" | xargs rm` could misbehave on filenames containing spaces, and what flag combination (`-print0` / `-0`) fixes it.

## Further Reading

- `man find`, `man grep`, `man awk` — full official reference for each
- `man 7 regex` — POSIX regex reference (basic vs extended)
- "The AWK Programming Language" (Aho, Kernighan, Weinberger) — the original book, short and highly practical
- explainshell.com — paste any `find`/`grep`/`awk` invocation to see flag-by-flag breakdown
