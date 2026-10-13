# Lesson 24: Introduction to Ansible & YAML Syntax

**Module:** Ansible – Configuration Management
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-23 — especially SSH keys and `ssh-copy-id` (Lesson 11), `apt` (Lesson 7), systemd services (Lesson 5), users/sudo (Lesson 6), and Terraform's declarative model (Lesson 20)

## Learning Objectives

By end of lesson student can:
- Explain what configuration management is, how Ansible differs from Terraform, and what "agentless", "push model" and "idempotent" mean in practice
- Read and write valid YAML: indentation rules, scalars, lists, maps, and multi-line strings (`|` vs `>`)
- Install Ansible and create a minimal `ansible.cfg` and hosts file
- Run ad-hoc commands against hosts with `-m ping`, `-m shell`, `-m copy`
- Write a first playbook with plays, tasks and the core modules `apt`, `file`, `copy`, `service`, `user`
- Run a playbook with `ansible-playbook`, preview changes with `--check`, and raise verbosity with `-v`/`-vv`

## Topics

- Ansible concepts: agentless, SSH-based, push model, idempotency
- YAML syntax: indentation, scalars, lists, maps, multi-line strings (`|` vs `>`)
- Installation; `ansible.cfg` basics; ad-hoc commands (`-m ping`, `-m shell`, `-m copy`)
- First playbook: plays, tasks, modules (`apt`, `file`, `copy`, `service`, `user`)
- Running: `ansible-playbook`, `--check` (dry run), `-v`/`-vv` verbosity flags

## Concepts

### What Ansible is, and where it fits

Terraform (Lessons 20-23) *provisions* infrastructure — it creates the servers, networks and cloud resources. **Configuration management** is the next layer: once a server exists, something has to install packages on it, write config files, create users, and start services. **Ansible** is a configuration management and automation tool that does exactly that, by connecting to machines and bringing them to a described state.

A rough division of labor in a real project: Terraform creates 5 virtual machines; Ansible then configures all 5 identically (nginx installed, users created, firewall configured). The two tools overlap a little at the edges, but this "Terraform provisions, Ansible configures" split is the most common pairing.

### Agentless, SSH-based, push model

Three properties define how Ansible works, and they are the main reasons teams choose it:

| Property | Meaning |
|---|---|
| **Agentless** | Nothing needs to be installed on the managed machines (the **managed nodes**). Only the machine you run Ansible from (the **control node**) needs Ansible itself. Tools like Puppet and Chef traditionally require a permanently running agent on every node |
| **SSH-based** | Ansible reaches managed nodes over plain SSH (Lesson 11) — the same keys, `~/.ssh/config`, and `authorized_keys` you already know. Managed nodes only need SSH access and Python 3 installed (virtually every Linux server has both) |
| **Push model** | The control node *pushes* instructions out to nodes when you run a command. Nodes don't poll a central server on a schedule. Nothing changes on any machine until a human (or a CI pipeline) triggers a run |

The tradeoff: because nothing runs continuously on nodes, Ansible doesn't automatically correct drift between runs — if someone hand-edits a server tomorrow, Ansible won't notice until the next time you run it.

Under the hood, Ansible copies a small Python script (a **module**) to the node over SSH, runs it, collects the JSON result, and deletes the script. That is why Python is needed on the managed node.

### Idempotency

Same concept as Terraform (Lesson 20): running the same playbook once, twice, or ten times leaves the machine in the same final state. Ansible's modules are written to *check first, change only if needed*:

- Task says "ensure package `nginx` is installed": if already installed, Ansible reports **ok** and does nothing.
- If not installed, Ansible installs it and reports **changed**.

Every task result is one of: `ok` (already correct, nothing done), `changed` (Ansible modified something), `failed`, or `skipped`. A healthy second run of a playbook should show **zero** `changed`. This is a strong mental check: if a re-run keeps showing `changed`, something is not idempotent (this usually happens with `shell`/`command` modules, covered below).

The mindset shift: write tasks that describe a *desired state* ("user alice exists", "file has mode 0644"), not a *sequence of commands* ("run useradd alice"). Modules like `user`, `apt`, `file` are declarative; the `shell` module is a raw escape hatch that Ansible cannot reason about.

### YAML syntax

Ansible playbooks, and later inventories, variables and roles, are all written in **YAML** ("YAML Ain't Markup Language"). YAML is a human-friendly data format built from just a few constructs. Learn these well — most beginner Ansible errors are YAML errors.

**Indentation is structure.** YAML uses *spaces* (never tabs) to express nesting. The convention is 2 spaces per level. Items at the same indentation level are siblings; deeper indentation means "child of the line above". A single misaligned space changes the meaning or breaks the file.

**Scalars** are single values: strings, numbers, booleans.

```yaml
name: nginx            # string (quotes optional for simple text)
port: 8080             # integer
ratio: 0.75            # float
enabled: true          # boolean (also: false)
version: "1.10"        # quoted on purpose — see pitfalls
empty_value: null      # null (also: ~ or blank)
```

Strings only *need* quotes when they contain special characters (`:` followed by a space, `#`, leading `*`, `{`, `[`, `%`, etc.) or when you must force string type (`"yes"`, `"1.10"`, `"0755"`).

**Lists** (sequences) are ordered items, each starting with `- ` (dash then space):

```yaml
packages:
  - nginx
  - curl
  - git
```

Short inline form (flow style) is equivalent: `packages: [nginx, curl, git]`.

**Maps** (dictionaries) are `key: value` pairs, with a space after the colon:

```yaml
user:
  name: alice
  shell: /bin/bash
  uid: 1500
```

Inline form: `user: {name: alice, shell: /bin/bash}`.

**Lists of maps** are the shape of almost every Ansible playbook — a list where each item is a map:

```yaml
servers:
  - name: web-01
    port: 80
  - name: db-01
    port: 5432
```

Note how the dash belongs to the first key of the map, and the following keys line up under that first key.

**Multi-line strings** come in two forms, and the difference matters:

```yaml
literal_block: |
  line one
  line two
  line three
# value is "line one\nline two\nline three\n"  — newlines PRESERVED

folded_block: >
  this is one long
  sentence that is wrapped
  for readability
# value is "this is one long sentence that is wrapped for readability\n" — newlines become SPACES
```

Rule of thumb: `|` ("pipe") keeps line breaks — use it for scripts and file contents. `>` ("folded") joins lines into one paragraph — use it for long descriptions or very long single-line commands.

**Comments** start with `#`. **Document start** `---` marks the beginning of a YAML document; Ansible playbooks conventionally begin with it.

### Installation and `ansible.cfg`

On Ubuntu the common ways to install Ansible (control node only):

- `sudo apt install ansible` — simplest; version may lag behind upstream (Lesson 7's concerns about repository age apply)
- `pip install ansible` inside a virtual environment (Lesson 19's `venv`) — newest version, isolated from system Python
- The official PPA (`add-apt-repository --yes --update ppa:ansible/ansible`) — recent version through `apt`

Verify with `ansible --version`, which also prints which config file is in use and the Python version.

**`ansible.cfg`** is Ansible's settings file (INI format). Ansible looks for it in this order and uses the *first one found*: the `ANSIBLE_CONFIG` environment variable, `./ansible.cfg` (current directory), `~/.ansible.cfg`, then `/etc/ansible/ansible.cfg`. Putting an `ansible.cfg` in your project directory is the standard practice, so settings travel with the project (and with Git, Lesson 14).

A minimal project config:

```ini
[defaults]
inventory = ./hosts
remote_user = ubuntu
private_key_file = ~/.ssh/id_ed25519
host_key_checking = False

[privilege_escalation]
become = False
```

| Setting | Purpose |
|---|---|
| `inventory` | Path to the file listing managed hosts (default `/etc/ansible/hosts`) |
| `remote_user` | The SSH username to connect as |
| `private_key_file` | Which SSH private key to use |
| `host_key_checking` | `False` skips the "authenticity of host" prompt (convenient in labs; keep `True` in production) |

### A minimal inventory

The **inventory** tells Ansible which machines exist. At this stage a plain text file with one host per line is enough; the next lesson covers groups, YAML inventories and variables in depth.

```ini
# hosts
web-01 ansible_host=192.168.1.51
web-02 ansible_host=192.168.1.52

[webservers]
web-01
web-02
```

`ansible_host` gives the real IP/DNS name to connect to (the first column is just an alias). `[webservers]` defines a **group** you can target. Two names always exist automatically: `all` (every host) and `ungrouped`. Use `ansible-inventory --list` or `ansible all --list-hosts` to see what Ansible understands.

### Ad-hoc commands

An **ad-hoc command** runs a *single module* against hosts straight from the command line, without writing a playbook. Syntax:

```
ansible <host-pattern> -m <module> -a "<module arguments>" [options]
```

Ad-hoc commands suit quick one-off checks and fixes; anything you want to repeat or version-control belongs in a playbook.

| Module | What it does | Example |
|---|---|---|
| `ping` | Tests SSH connectivity *and* that Python works on the node (not an ICMP ping — returns `pong`) | `ansible all -m ping` |
| `shell` | Runs a command through a shell on the node (pipes, `$VARS`, redirection work) | `ansible all -m shell -a "uptime"` |
| `command` | Runs a command without a shell (the default module if `-m` is omitted; safer) | `ansible all -a "df -h"` |
| `copy` | Copies a local file to the node | `ansible all -m copy -a "src=./motd dest=/etc/motd" --become` |

`--become` (or `-b`) escalates to root with `sudo` on the node — needed for anything touching system files, packages, or services. `-u user` overrides the SSH user, `-i file` overrides the inventory, `-k` prompts for an SSH password if keys are not set up, `-K` prompts for the sudo password.

Ad-hoc output colors: **green** = ok/unchanged, **yellow** = changed, **red** = failed.

### Playbooks: plays and tasks

A **playbook** is a YAML file describing the desired state of your machines. Its structure:

- A playbook is a **list of plays**.
- A **play** maps a group of hosts to a list of tasks (`hosts:` plus `tasks:`).
- A **task** calls one module with arguments. Tasks run **in order**, top to bottom, on every targeted host.

```yaml
---
- name: Configure web servers          # play name (shown in output)
  hosts: webservers                    # which inventory group/hosts
  become: true                         # run tasks as root via sudo

  tasks:
    - name: Install nginx              # task name (shown in output)
      ansible.builtin.apt:             # module (fully qualified name)
        name: nginx
        state: present
        update_cache: true
```

Always give plays and tasks a descriptive `name:` — it is what you read in the run output and in logs.

Each module takes its own arguments; the core `state:` argument usually expresses desired state (`present`, `absent`, `started`, `latest`, `directory`...). The five core modules for this lesson:

| Module | Purpose | Key arguments |
|---|---|---|
| `apt` | Manage Debian/Ubuntu packages | `name`, `state` (`present`/`absent`/`latest`), `update_cache` |
| `file` | Manage files, directories, links, permissions | `path`, `state` (`file`/`directory`/`absent`/`link`/`touch`), `mode`, `owner`, `group` |
| `copy` | Copy a file to the node, or write inline `content:` | `src`, `dest`, `content`, `mode`, `owner` |
| `service` | Manage services (systemd under the hood, Lesson 5) | `name`, `state` (`started`/`stopped`/`restarted`/`reloaded`), `enabled` |
| `user` | Manage user accounts (Lesson 6's `useradd`/`usermod`) | `name`, `state`, `shell`, `groups`, `append`, `create_home` |

**Fully Qualified Collection Names (FQCN):** modern Ansible recommends `ansible.builtin.apt` instead of just `apt`. Short names still work, and both appear in documentation and real projects; this lesson uses FQCN in the main walkthrough and the short form where noted.

A play may also define a simple `vars:` map of play-level variables, referenced with `"{{ variable_name }}"` (quotes required when a value *starts* with `{{`):

```yaml
  vars:
    web_package: nginx
  tasks:
    - name: Install {{ web_package }}
      ansible.builtin.apt:
        name: "{{ web_package }}"
        state: present
```

### Running playbooks, dry runs and verbosity

```
ansible-playbook site.yml
```

Output is organized as `PLAY` -> `TASK` lines, then a **PLAY RECAP** per host summarizing counts: `ok`, `changed`, `unreachable`, `failed`, `skipped`, `rescued`, `ignored`. A successful run has `failed=0` and `unreachable=0`.

| Flag | Effect |
|---|---|
| `--check` | **Dry run**: reports what *would* change without changing anything. Great for a safety review before touching production |
| `--diff` | Shows line-by-line before/after for file changes (pairs well with `--check`) |
| `-v` | Verbose: also prints each task's return data |
| `-vv` | More verbose: adds task file/line info |
| `-vvv` | Connection-level detail (SSH commands); very useful for debugging unreachable hosts |
| `--syntax-check` | Validates YAML/playbook structure without running anything |
| `--list-tasks` | Lists the tasks that would run |

`--check` caveat: modules that cannot predict their result (notably `shell` and `command`) are *skipped* in check mode, and later tasks that depend on earlier ones having really run might report misleading results. Treat `--check` as a strong hint, not a guarantee.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `ansible --version` | Show version, config file, Python | `ansible --version` |
| `ansible <pattern> -m ping` | Test connectivity + Python on nodes | `ansible all -m ping` |
| `ansible <pattern> -m shell -a` | Run a shell command | `ansible all -m shell -a "uptime \| awk '{print \$1}'"` |
| `ansible <pattern> -m copy -a` | Copy a file | `ansible web-01 -m copy -a "src=a.txt dest=/tmp/a.txt"` |
| `ansible ... --become` / `-b` | Escalate to root (sudo) | `ansible all -m apt -a "name=curl" -b` |
| `ansible ... -i` | Use a specific inventory file | `ansible all -i hosts -m ping` |
| `ansible-inventory --list` | Show parsed inventory | `ansible-inventory -i hosts --list` |
| `ansible-playbook` | Run a playbook | `ansible-playbook site.yml` |
| `ansible-playbook --check` | Dry run | `ansible-playbook site.yml --check --diff` |
| `ansible-playbook -v/-vv/-vvv` | Increase verbosity | `ansible-playbook site.yml -vv` |
| `ansible-playbook --syntax-check` | Validate playbook syntax | `ansible-playbook site.yml --syntax-check` |
| `ansible-doc` | Read module docs offline | `ansible-doc ansible.builtin.apt` |

## Examples / Walkthrough

```bash
# --- installation ---
sudo apt update
sudo apt install ansible                 # or: python3 -m venv .venv && . .venv/bin/activate && pip install ansible
ansible --version                        # note "config file = None" until we create one

# --- project directory ---
mkdir ~/ansible-lab && cd ~/ansible-lab

# --- SSH prerequisites (Lesson 11) — control node must reach nodes with a key ---
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""        # skip if you already have a key
ssh-copy-id ubuntu@192.168.1.51
ssh-copy-id ubuntu@192.168.1.52
ssh ubuntu@192.168.1.51 'python3 --version'              # confirm Python 3 exists on the node
```

```ini
# --- ansible.cfg (in ~/ansible-lab) ---
[defaults]
inventory = ./hosts
remote_user = ubuntu
host_key_checking = False
```

```ini
# --- hosts (in ~/ansible-lab) ---
web-01 ansible_host=192.168.1.51
web-02 ansible_host=192.168.1.52

[webservers]
web-01
web-02
```

```bash
# --- confirm Ansible reads our files ---
ansible --version                         # now shows: config file = /home/you/ansible-lab/ansible.cfg
ansible-inventory --list                  # JSON view of the parsed inventory
ansible all --list-hosts

# --- ad-hoc commands ---
ansible all -m ping                       # expect green SUCCESS / "ping": "pong" from each host

ansible webservers -m shell -a "uptime"
ansible webservers -a "df -h /"            # command module (default), no shell features
ansible webservers -a "whoami"            # -> ubuntu
ansible webservers -a "whoami" --become   # -> root

echo "Managed by Ansible" > motd.txt
ansible webservers -m copy -a "src=motd.txt dest=/etc/motd mode=0644" --become
# first run: yellow "changed": true
ansible webservers -m copy -a "src=motd.txt dest=/etc/motd mode=0644" --become
# second run: green "changed": false  <- idempotency in action

ansible webservers -m apt -a "name=tree state=present update_cache=true" --become
ansible webservers -m service -a "name=cron state=started enabled=true" --become
```

```yaml
# --- site.yml : first playbook, uses all five core modules ---
---
- name: Configure web servers
  hosts: webservers
  become: true

  vars:
    web_package: nginx

  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: "{{ web_package }}"
        state: present
        update_cache: true
        cache_valid_time: 3600          # skip apt update if cache is under 1 hour old

    - name: Create web content directory
      ansible.builtin.file:
        path: /var/www/lab
        state: directory
        mode: "0755"
        owner: www-data
        group: www-data

    - name: Write a landing page
      ansible.builtin.copy:
        dest: /var/www/lab/index.html
        content: |
          <h1>Hello from {{ inventory_hostname }}</h1>
          <p>Configured by Ansible.</p>
        mode: "0644"

    - name: Create a deploy user
      ansible.builtin.user:
        name: deploy
        shell: /bin/bash
        groups: sudo
        append: true
        create_home: true
        state: present

    - name: Ensure nginx is running and starts on boot
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

```bash
# --- running the playbook ---
ansible-playbook site.yml --syntax-check      # validate before doing anything
ansible-playbook site.yml --check --diff      # dry run: what WOULD change?
ansible-playbook site.yml                      # apply for real
# PLAY RECAP: web-01 : ok=6 changed=5 ...

ansible-playbook site.yml                      # run again immediately
# PLAY RECAP: web-01 : ok=6 changed=0 ...   <- idempotent: nothing left to change

ansible-playbook site.yml -v                   # shows each task's returned data
ansible-playbook site.yml -vv                  # adds file/line context
ansible-playbook site.yml -vvv                 # SSH-level detail, for debugging connectivity

curl http://192.168.1.51/lab/                  # hmm — default nginx root is /var/www/html; see pitfalls

# --- offline module documentation ---
ansible-doc ansible.builtin.apt
ansible-doc -s ansible.builtin.file            # short snippet form
```

```yaml
# --- YAML practice: spot the structure ---
---
company:
  name: Example Corp
  founded: 2015
  active: true
  offices:
    - city: Tashkent
      employees: 40
    - city: Samarkand
      employees: 12
  motto: >
    Ship small changes
    often and safely.
  login_banner: |
    Authorized use only.
    All activity is logged.
```

## Common Pitfalls

- **Tabs in YAML** — YAML forbids tab characters for indentation; one stray tab produces `found character '\t' that cannot start any token`. Configure vim/nano (Lesson 3) to insert spaces (`:set expandtab tabstop=2 shiftwidth=2` in vim).
- **Inconsistent indentation** — a task indented 3 spaces under a list indented 2 silently changes structure or errors with `mapping values are not allowed here`. Pick 2 spaces and keep it everywhere; the dash `- ` of a list item counts as part of its indentation.
- **Missing space after a colon or dash** — `name:nginx` is parsed as a single string, not a key/value pair; `-nginx` is a string, not a list item. Always `key: value` and `- item`.
- **Unquoted values YAML "helpfully" converts** — `mode: 0755` is read as an *octal number* (493 decimal) in some contexts, `version: 1.10` becomes the float `1.1`, and bare `yes`/`no`/`on`/`off` become booleans. Quote file modes (`mode: "0755"`), versions (`"1.10"`) and anything that must stay a string.
- **Unquoted `{{ variable }}` at the start of a value** — `name: {{ web_package }}` is parsed as the start of a YAML map and fails. Quote it: `name: "{{ web_package }}"`.
- **Using `shell`/`command` for things a module already does** — `shell: apt install -y nginx` runs every time and always reports `changed`, breaking idempotency and `--check`. Use `apt`, `user`, `file`, `service`. Reach for `shell` only when no module fits.
- **Forgetting `become: true`** — installing packages, writing under `/etc` or `/var/www`, or managing services as an unprivileged SSH user fails with `Permission denied` or `Failed to lock apt for exclusive operation`. Add `become: true` at play level (or `--become` for ad-hoc).
- **Assuming `ping` is ICMP** — Ansible's `ping` module logs in over SSH and runs Python; `pong` means SSH, keys, and Python all work. A failure can mean any of those, not just a network problem — rerun with `-vvv` to see which.
- **Not running `--check` before production** — a typo in `state: absent` against a real path or package is only caught if you look first. Make `--syntax-check` then `--check --diff` a habit before every real run.
- **Expecting Ansible to correct drift automatically** — it is push-based, so a hand-edited server stays modified until the next playbook run. Schedule runs (cron, Lesson 9, or a CI pipeline later in the course) if you need continuous enforcement.
- **Wrong default web root** — nginx on Ubuntu serves `/var/www/html` by default; the walkthrough's `/var/www/lab` directory is created but not served until nginx's site config points there (nginx config management by playbook comes with templating in a later lesson). For now, write to `/var/www/html/index.html` to see the page immediately.

## FAQ

**Q: Do I need to install anything on the servers I want to manage?**
A: No Ansible software — that's the "agentless" property. The nodes need SSH access and Python 3 (preinstalled on Ubuntu server images). Only the control node needs Ansible.

**Q: What's the difference between Ansible and Terraform?**
A: Terraform creates and destroys infrastructure resources (servers, networks, DNS) and tracks them in state. Ansible configures what runs *inside* those machines (packages, files, users, services) and keeps no state file — it asks each node what its current state is every run. They are complementary, not competitors, though each can do a little of the other's job.

**Q: Why is my second playbook run showing `changed` again?**
A: A task is not idempotent. Usually it uses `shell`/`command` (which always report changed), or something rewrites a file every run. Check the PLAY RECAP and the yellow tasks with `-v` to find which one, then replace it with a proper module or add module-specific guards.

**Q: What's the difference between `command` and `shell`?**
A: `command` runs the program directly — no pipes, redirects, `&&`, or `$VARIABLES`; safer and the default. `shell` runs through `/bin/sh`, so those features work, but it is also more prone to quoting mistakes and injection. Prefer `command`, use `shell` only when you need shell features.

**Q: When should I use `|` and when `>` for multi-line text?**
A: `|` keeps line breaks exactly — use it for file contents and scripts, as with `copy: content: |`. `>` folds lines into one long line separated by spaces — use it for long descriptions or a very long command you want to wrap across lines for readability.

**Q: Is `--check` completely safe and accurate?**
A: It is safe (changes nothing) but not always accurate: tasks using `shell`/`command` are skipped, and tasks depending on a previous task's real effects can misreport. It is a strong preview, not a guarantee, so keep a staging environment for important changes.

**Q: What is the difference between a play and a playbook?**
A: A playbook is the YAML file; it contains one or more plays. Each play targets one set of hosts with its own tasks. A playbook with two plays could, for example, configure `webservers` in the first play and `databases` in the second.

**Q: Should I write `apt:` or `ansible.builtin.apt:`?**
A: Both work. The fully qualified `ansible.builtin.apt` is the modern recommendation because it is unambiguous when third-party collections are installed. Older tutorials and many existing projects use the short form, so recognize both.

**Q: Where does Ansible look for `ansible.cfg`?**
A: In order: `$ANSIBLE_CONFIG`, `./ansible.cfg`, `~/.ansible.cfg`, `/etc/ansible/ansible.cfg`; the first one found wins and files are not merged. `ansible --version` prints the one in use.

## Practice / Exercise

**Core:**
1. Install Ansible on a control node and confirm with `ansible --version`.
2. Prepare 2 lab VMs (or containers/VMs from earlier lessons) reachable over SSH with key authentication; confirm `ssh user@host 'python3 --version'` works on each.
3. Create a project directory with an `ansible.cfg` and a `hosts` file containing both machines in a `[webservers]` group; verify with `ansible-inventory --list`.
4. Run `ansible all -m ping`. Then deliberately break it (wrong `ansible_host` IP) and observe the `UNREACHABLE` error, then fix it.
5. Using ad-hoc commands: check `uptime` with `shell`, check free disk with `command`, and compare `whoami` with and without `--become`.
6. Use `-m copy` to place a file in `/tmp` on both nodes, run the same command twice, and explain the difference between the two outputs.
7. Write the YAML for a map describing a server (name, ip, ports as a list, and a multi-line `description` using `>`), and validate it with `python3 -c "import yaml,sys; print(yaml.safe_load(open('file.yml')))"` or `ansible-playbook --syntax-check` in a minimal playbook.
8. Write `site.yml` using `apt`, `file`, `copy`, `user`, and `service` modules as in the walkthrough. Run `--syntax-check`, then `--check --diff`, then a real run, then a second run, and confirm the second shows `changed=0`.
9. Re-run the playbook with `-v` and `-vv` and note what extra information each level adds.

**Stretch:**
1. Rewrite the ad-hoc `copy` and `service` commands as tasks in a second playbook, and compare how much easier it is to read, repeat and keep in Git.
2. Change one task to `ansible.builtin.shell: apt install -y nginx` and observe in two consecutive runs why it breaks idempotency; then switch back to the `apt` module.
3. Introduce three deliberate YAML errors (a tab, a missing space after a colon, inconsistent indentation) one at a time, record each exact error message, and learn to recognize them.
4. Use `ansible-doc ansible.builtin.user` to find an argument you did not use (for example `password_expire_max` or `comment`) and add it to your playbook.
5. Create a playbook with two plays: one targeting `webservers` and a second targeting `localhost` with `connection: local`; run it and explain what each play's PLAY RECAP shows.

## Further Reading

- [Ansible documentation — Getting started](https://docs.ansible.com/ansible/latest/getting_started/index.html)
- [Ansible Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/index.html)
- [YAML syntax in Ansible](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)
- [Introduction to ad-hoc commands](https://docs.ansible.com/ansible/latest/command_guide/intro_adhoc.html)
- `ansible-doc -l` — list all modules available offline; `man ansible`, `man ansible-playbook`
