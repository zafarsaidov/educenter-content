# Lesson 25: Inventory Files & Folder Structure

**Module:** Ansible – Configuration Management
**Duration:** 120-150 min
**Prerequisites:** Lesson 24 (Introduction to Ansible & YAML Syntax) — ad-hoc commands, `ansible.cfg` basics, first playbook, SSH access to managed hosts

## Learning Objectives

By end of lesson student can:
- Write a static inventory in INI format using groups, child groups, host ranges, and aliases
- Write the same inventory in YAML format and explain when each format is preferable
- Store per-group and per-host variables in `group_vars/` and `host_vars/` instead of the inventory file
- Explain what a dynamic inventory is, and how cloud inventory plugins and custom scripts feed Ansible
- Lay out an Ansible project directory in a conventional, scalable structure
- Configure how Ansible connects to each host using `ansible_host`, `ansible_user`, `ansible_ssh_private_key_file`, and related variables
- Verify an inventory with `ansible-inventory --list`, `ansible-inventory --graph`, and `ansible all -m ping`

## Topics

- Static inventory: INI format (groups, child groups, ranges, aliases)
- YAML inventory format; `host_vars/` and `group_vars/` directories
- Dynamic inventory: concept, AWS/GCP plugins, custom script approach
- Ansible project directory structure: playbooks, roles, `group_vars`, `host_vars`
- Connection variables: `ansible_host`, `ansible_user`, `ansible_ssh_private_key_file`

## Concepts

### What an inventory is, and why it exists

In Lesson 24 you ran ad-hoc commands and a first playbook against a handful of hosts. Ansible knew *which* machines to touch because of an **inventory**: a file (or script, or plugin) that lists the managed hosts and organizes them into **groups**. A playbook never says "connect to 10.0.0.11" — it says `hosts: webservers`, and the inventory resolves that name into real machines. That indirection is the whole point: the same playbook can target a 2-server dev environment or a 200-server production fleet, purely by pointing it at a different inventory.

The inventory location is set in `ansible.cfg` (`inventory = ./inventory/hosts.ini`, Lesson 24) or per-run with `-i`:

```bash
ansible all -i inventory/hosts.ini -m ping
```

`-i` also accepts a directory (Ansible then merges every inventory file inside it), or a comma-separated list of hosts for a quick one-off (`-i "10.0.0.5,"` — note the trailing comma, which tells Ansible it's a host list and not a filename).

Two implicit groups always exist, with no declaration needed: `all` (every host in the inventory) and `ungrouped` (hosts that belong to no other group).

### Static inventory: INI format

The INI format is the oldest and most common — plain text, a bare host list with optional `[group]` headers:

```ini
# ungrouped hosts go at the top, before any [group] header
bastion.example.com

[webservers]
web01.example.com
web02.example.com

[dbservers]
db01.example.com
```

Everything under a `[header]` belongs to that group until the next header. A host can appear in more than one group — it simply inherits from all of them.

**Ranges.** For numbered fleets, a bracketed range avoids typing every name:

```ini
[webservers]
web[01:10].example.com        # web01 .. web10 (zero-padding preserved from the start value)

[dbservers]
db-[a:c].example.com          # db-a, db-b, db-c  (alphabetic ranges work too)
```

**Aliases.** A host line can start with a short, memorable *alias* that is independent of its real address. The alias is what you refer to in playbooks and on the command line; `ansible_host` (covered below) tells Ansible where to actually connect:

```ini
[webservers]
web-prod-1 ansible_host=203.0.113.11
web-prod-2 ansible_host=203.0.113.12
```

**Child groups.** The special suffix `:children` makes a group whose *members are other groups*, creating a hierarchy:

```ini
[webservers]
web01.example.com
web02.example.com

[dbservers]
db01.example.com

[production:children]
webservers
dbservers
```

Now `hosts: production` targets every web and DB server, while `hosts: webservers` still targets only the web tier. Hierarchies like `production` / `staging`, or `eu` / `us`, layered over role-based groups like `webservers` / `dbservers`, are the standard way to slice a fleet along more than one axis.

**Inline and group variables.** INI lets you attach variables directly: per-host on the host line (`ansible_user=deploy`), and per-group in a `[group:vars]` section:

```ini
[webservers:vars]
http_port=8080
ansible_user=deploy
```

This works, but INI inventories become hard to read once they carry many variables — which is exactly the problem `group_vars/` and `host_vars/` (below) solve.

### YAML inventory format

The same structure can be written in YAML. It's more verbose for small inventories but handles nested hierarchies and variables more cleanly, and it round-trips naturally into other tooling (Lesson 24's YAML rules apply — indentation is structure):

```yaml
all:
  hosts:
    bastion.example.com:
  children:
    production:
      children:
        webservers:
          hosts:
            web01.example.com:
            web02.example.com:
          vars:
            http_port: 8080
        dbservers:
          hosts:
            db01.example.com:
              ansible_host: 10.0.0.21
```

Reading rules: every host is a *key* (hence the trailing colon, even when it has no variables); groups live under `children:`; hosts under `hosts:`; variables under `vars:` (group) or directly beneath the host key (host). Ranges work here too: `web[01:10].example.com:`.

**Which to choose?** INI for small, flat inventories and quick readability; YAML when the hierarchy is deep, when variables are numerous, or when other tools generate or consume the file. Both are first-class — pick one per inventory and be consistent. `ansible-inventory` converts between them (`ansible-inventory -i hosts.ini --list -y` prints the same inventory as YAML).

### `group_vars/` and `host_vars/`

Variables embedded in the inventory file get unwieldy fast. The convention: keep the inventory as a pure *list of hosts and groups*, and put variables in sibling directories that Ansible loads automatically, matched by name:

```
inventory/
├── hosts.ini
├── group_vars/
│   ├── all.yml            # applies to every host
│   ├── webservers.yml     # applies to the [webservers] group
│   └── dbservers.yml
└── host_vars/
    ├── web01.example.com.yml   # applies to this one host only
    └── db01.example.com.yml
```

- `group_vars/<groupname>.yml` — variables for every host in that group. `group_vars/all.yml` applies to every host.
- `host_vars/<hostname>.yml` — variables for one host; the filename must match the host's inventory name (or alias) exactly.
- A *directory* named after the group/host works too (`group_vars/webservers/` containing several `.yml` files — each is loaded and merged), which is useful once a single group's variables grow large and deserve to be split by topic.

These files are plain YAML maps:

```yaml
# group_vars/webservers.yml
http_port: 8080
app_user: www-data
```

**Precedence in one sentence.** The more specific scope wins: `host_vars` overrides `group_vars` for a group the host belongs to; a child group's `group_vars` overrides its parent's; and `group_vars/all.yml` is the weakest default. (Ansible has a long, formal precedence list covering many more sources — extra-vars, play vars, role defaults. Lesson 26 revisits precedence in the context of roles; for now, "specific beats general" is the working rule.)

`group_vars/` and `host_vars/` are located relative to the **inventory file** (or relative to the **playbook**, which is also searched) — which is why the layout above places them beside `hosts.ini`.

### Dynamic inventory

A static file works when hosts are long-lived and few. In cloud environments, servers appear and disappear constantly — an autoscaling group may add 5 instances at noon — and a hand-maintained file goes stale immediately. A **dynamic inventory** asks an external source for the current host list *at run time* instead of reading a fixed file.

Two mechanisms:

1. **Inventory plugins** (the modern, recommended way). A plugin is configured by a small YAML file whose name ends in a plugin-specific suffix; Ansible calls the cloud API and builds groups from instance metadata (tags, region, instance type). Examples:

```yaml
# inventory/aws_ec2.yml  — AWS EC2 plugin (amazon.aws.aws_ec2)
plugin: amazon.aws.aws_ec2
regions:
  - eu-west-1
filters:
  tag:Environment: production
keyed_groups:
  - key: tags.Role          # creates groups like role_web, role_db from the "Role" tag
    prefix: role
```

```yaml
# inventory/gcp.yml — GCP Compute plugin (google.cloud.gcp_compute)
plugin: google.cloud.gcp_compute
projects:
  - my-gcp-project
auth_kind: serviceaccount
service_account_file: ~/gcp-key.json
```

The plugin comes from a **collection** (`amazon.aws`, `google.cloud`) that you install with `ansible-galaxy collection install amazon.aws`, plus the cloud's Python SDK (`boto3` for AWS). Credentials come from the usual cloud mechanisms (environment variables, credentials files, instance roles) — never hardcoded in the inventory file that goes into Git.

2. **Custom inventory scripts.** Any executable that, when called with `--list`, prints a JSON document describing groups and hosts (and, when called with `--host <name>`, that host's variables) is a valid inventory source. Ansible runs it and parses the JSON. This is the escape hatch for homegrown CMDBs or internal APIs that no plugin covers:

```bash
#!/usr/bin/env python3
# inventory/custom.py  — must be executable (chmod +x)
import json, sys

inventory = {
    "webservers": {"hosts": ["web01", "web02"], "vars": {"http_port": 8080}},
    "_meta": {"hostvars": {"web01": {"ansible_host": "10.0.0.11"},
                           "web02": {"ansible_host": "10.0.0.12"}}},
}
print(json.dumps(inventory))
```

The `_meta.hostvars` key lets the script return every host's variables in one call (otherwise Ansible would invoke `--host` once per host, which is slow). A script like this is where Lesson 19's Python (`json`, `sys`, `subprocess`) pays off directly.

You can mix sources: pointing `-i` at a *directory* that holds a static `hosts.ini` plus an `aws_ec2.yml` plugin file merges both into one inventory.

### Ansible project directory structure

There is no mandatory layout, but a convention has emerged that scales from a handful of playbooks to a large codebase. A typical project:

```
ansible-project/
├── ansible.cfg
├── inventory/
│   ├── production/
│   │   ├── hosts.ini
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   └── webservers.yml
│   │   └── host_vars/
│   │       └── web01.example.com.yml
│   └── staging/
│       ├── hosts.ini
│       └── group_vars/
│           └── all.yml
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── dbservers.yml
└── roles/                  # reusable bundles of automation — Lesson 26 covers these
```

- **`ansible.cfg`** at the project root pins the inventory path and defaults, so anyone who clones the repo and runs `ansible` from the root gets identical behavior (Lesson 24).
- **`inventory/<environment>/`** — one self-contained inventory per environment, each with its own `group_vars`/`host_vars`. Switching environments is `-i inventory/staging` vs `-i inventory/production`; the playbooks stay identical, only the data changes. This is the layout that makes "the same playbook for every environment" real.
- **`playbooks/`** — the playbooks themselves. A common top-level `site.yml` is the "run everything" entry point that imports the per-tier playbooks.
- **`roles/`** — a directory reserved for roles, which package tasks, files, and variables for reuse. You will create the *directory* now so the layout is complete; what goes inside it is the subject of the next lesson.

Whole project lives in one Git repository (Lesson 14); `.gitignore` anything secret.

### Connection variables

Ansible must know *how* to reach each host. A set of reserved **connection variables** controls this. They can be set anywhere a variable can live — on a host line, in `[group:vars]`, in `group_vars/`, in `host_vars/`:

| Variable | Purpose | Default |
|---|---|---|
| `ansible_host` | The real address (IP or DNS name) to connect to, when it differs from the inventory name/alias | the inventory name |
| `ansible_port` | SSH port | `22` |
| `ansible_user` | The remote login user | the current local user |
| `ansible_ssh_private_key_file` | Path to the SSH private key to authenticate with | SSH default keys / agent |
| `ansible_connection` | How to connect: `ssh`, `local` (no SSH, run on the control node itself), `docker`, etc. | `ssh` |
| `ansible_become` | Whether to escalate privileges (sudo) for tasks | `false` |
| `ansible_python_interpreter` | Path to Python on the managed host | auto-discovered |

Together with Lesson 11's SSH material: `ansible_ssh_private_key_file` points at the key you generated with `ssh-keygen` and distributed with `ssh-copy-id`; `ansible_user` is the account whose `authorized_keys` holds it; `ansible_port` handles hardened hosts that moved SSH off port 22 in `sshd_config`.

Example host with everything spelled out:

```ini
app-prod-1 ansible_host=203.0.113.50 ansible_port=2222 ansible_user=deploy ansible_ssh_private_key_file=~/.ssh/prod_ed25519
```

Setting these once at group level (`[production:vars]` or `group_vars/production.yml`) avoids repeating them per host.

### Verifying an inventory

Never trust an inventory you haven't inspected. Three read-only commands, in increasing order of "reality":

| Command | What it proves |
|---|---|
| `ansible-inventory --graph` | The group/host *hierarchy* as a tree — confirms child groups and membership |
| `ansible-inventory --list` | Full JSON of every host with its **resolved variables** (merged from `group_vars`, `host_vars`, inline) — confirms variable scoping |
| `ansible all -m ping` | That Ansible can actually *connect* and run Python on every host — confirms SSH, user, key, and port are correct |

`ansible-inventory --host web01` prints the merged variables for a single host. The `ping` module here is Ansible's connectivity test (a successful reply is `"ping": "pong"`), not an ICMP ping.

## Commands / Syntax Reference

| Command / syntax | Purpose | Example |
|---|---|---|
| `-i <path>` | Choose the inventory (file, directory, or `host,` list) | `ansible all -i inventory/staging -m ping` |
| `ansible-inventory --graph` | Show group/host tree | `ansible-inventory -i inventory/production --graph` |
| `ansible-inventory --list` | Dump inventory + variables as JSON | `ansible-inventory -i hosts.ini --list` |
| `ansible-inventory --host` | Variables for one host | `ansible-inventory -i hosts.ini --host web01` |
| `ansible-inventory --list -y` | Dump as YAML | `ansible-inventory -i hosts.ini --list -y` |
| `ansible all -m ping` | Test connectivity to every host | `ansible all -m ping` |
| `ansible <group> -m ping` | Test one group | `ansible webservers -m ping` |
| `[group:children]` | INI: group made of other groups | `[production:children]` |
| `[group:vars]` | INI: variables for a whole group | `[webservers:vars]` |
| `host[01:10]` | INI/YAML host range | `web[01:10].example.com` |
| `ansible_host` | Real address for an alias | `web1 ansible_host=10.0.0.11` |
| `ansible_user` | Remote login user | `ansible_user=deploy` |
| `ansible_ssh_private_key_file` | SSH key path | `ansible_ssh_private_key_file=~/.ssh/id_ed25519` |
| `ansible-galaxy collection install` | Install an inventory plugin's collection | `ansible-galaxy collection install amazon.aws` |

## Examples / Walkthrough

```bash
# --- 1. Build the project skeleton ---
mkdir -p ~/ansible-project/{inventory/production/{group_vars,host_vars},inventory/staging/group_vars,playbooks,roles}
cd ~/ansible-project

cat > ansible.cfg <<'EOF'
[defaults]
inventory = ./inventory/staging
host_key_checking = False
EOF
# (host_key_checking = False is convenient for a lab; in real environments keep it enabled — Lesson 11's known_hosts)
```

```ini
# --- 2. inventory/production/hosts.ini ---
bastion ansible_host=203.0.113.5

[webservers]
web[01:03].example.com

[dbservers]
db01 ansible_host=10.0.0.21
db02 ansible_host=10.0.0.22

[production:children]
webservers
dbservers

[production:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/prod_ed25519
```

```bash
# --- 3. Inspect the hierarchy before touching any server ---
ansible-inventory -i inventory/production --graph
# @all:
#   |--@ungrouped:
#   |  |--bastion
#   |--@production:
#   |  |--@webservers:
#   |  |  |--web01.example.com
#   |  |  |--web02.example.com
#   |  |  |--web03.example.com
#   |  |--@dbservers:
#   |  |  |--db01
#   |  |  |--db02

ansible-inventory -i inventory/production --list      # full JSON incl. resolved variables
ansible-inventory -i inventory/production --host db01 # variables Ansible will use for db01
```

```yaml
# --- 4. inventory/production/group_vars/all.yml ---
ntp_server: pool.ntp.org
admin_email: ops@example.com
```

```yaml
# --- 5. inventory/production/group_vars/webservers.yml ---
http_port: 8080
app_user: www-data
```

```yaml
# --- 6. inventory/production/host_vars/web01.example.com.yml — override just for web01 ---
http_port: 9090
```

```bash
# --- 7. Confirm variable scoping: web01 should see 9090, web02 should see 8080 ---
ansible-inventory -i inventory/production --host web01.example.com | grep http_port    # "http_port": 9090
ansible-inventory -i inventory/production --host web02.example.com | grep http_port    # "http_port": 8080
```

```bash
# --- 8. Convert INI to YAML to compare formats ---
ansible-inventory -i inventory/production --list -y > /tmp/production-inventory.yml
less /tmp/production-inventory.yml
```

```bash
# --- 9. Test real connectivity (needs reachable hosts and working SSH keys) ---
ansible all -i inventory/production -m ping
# web01.example.com | SUCCESS => { "changed": false, "ping": "pong" }
# db01 | UNREACHABLE! => ...   <- read the message: wrong ansible_host? wrong user? key not authorized?

ansible webservers -i inventory/production -m ping     # only one group
```

```yaml
# --- 10. A simple playbook uses the inventory group, never hardcoded IPs ---
# playbooks/webservers.yml
- name: Basic web tier setup
  hosts: webservers
  become: true
  tasks:
    - name: Ensure nginx is installed
      apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure nginx is running and enabled
      service:
        name: nginx
        state: started
        enabled: true
```

```bash
ansible-playbook -i inventory/production playbooks/webservers.yml --check   # dry-run first (Lesson 24)
```

```bash
# --- 11. Dynamic inventory: a custom script ---
cat > inventory/custom.py <<'EOF'
#!/usr/bin/env python3
import json
print(json.dumps({
    "webservers": {"hosts": ["web01", "web02"]},
    "_meta": {"hostvars": {"web01": {"ansible_host": "10.0.0.11"},
                           "web02": {"ansible_host": "10.0.0.12"}}},
}))
EOF
chmod +x inventory/custom.py
ansible-inventory -i inventory/custom.py --graph

# --- 12. Dynamic inventory: AWS plugin (requires AWS credentials in the environment) ---
ansible-galaxy collection install amazon.aws
pip install boto3 botocore
ansible-inventory -i inventory/aws_ec2.yml --graph
```

## Common Pitfalls

- **Forgetting the trailing comma in an ad-hoc `-i` host list** — `ansible all -i 10.0.0.5 -m ping` makes Ansible treat `10.0.0.5` as an inventory *filename*, which doesn't exist, and fails with "Unable to parse". `-i "10.0.0.5,"` (trailing comma) marks it as a literal host list.
- **A `host_vars` filename that doesn't match the inventory name** — if the inventory says `web01` (alias) but the file is `host_vars/web01.example.com.yml`, the variables silently never load. The filename must equal the name *as written in the inventory*, not the DNS name behind it. Confirm with `ansible-inventory --host <name>`.
- **Using the alias in `ansible_host` confusion** — an alias such as `web-prod-1` is not resolvable by DNS; without `ansible_host=<ip>` Ansible tries to SSH to a hostname that doesn't exist, producing "Could not resolve hostname". The alias is a *label*, `ansible_host` is the *address*.
- **YAML inventory hosts missing the colon** — in a YAML inventory, a host with no variables must still end in `:` (`web01.example.com:`), because hosts are map keys. Omitting it is a parse error or turns the entry into something unintended.
- **Defining the same variable in many places and not knowing which wins** — a value set in `group_vars/all.yml`, a child group, and `host_vars` resolves to the most specific; people debug the wrong file. Use `ansible-inventory --host <name>` to see the final, merged value instead of guessing.
- **Placing `group_vars/` in the wrong location** — it must sit next to the inventory file (or next to the playbook). A `group_vars/` directory at the project root works only if your inventory file is also at the project root; with `inventory/production/hosts.ini`, the directory belongs in `inventory/production/`.
- **Committing credentials or private keys in inventory/`group_vars`** — `ansible_ssh_private_key_file` should point to a *path*, never contain key material; passwords (`ansible_password`) in plain-text inventory files end up in Git history. Keep secrets out of repositories (a later lesson in this module covers encrypting them).
- **Assuming a dynamic inventory is cached forever or never** — plugins query the cloud API on each run unless caching is configured; a slow run may just be the API call, and a stale result can be a cache. Re-run `ansible-inventory --graph` to see what Ansible currently believes.
- **Running playbooks against `all` by accident** — a stray `hosts: all` plus the wrong `-i` can change every machine. Always inspect with `ansible-inventory --graph`, and preview with `--check` first (Lesson 24).

## FAQ

**Q: What's the difference between an inventory alias and a hostname?**
A: The first token on an inventory line is the *inventory name* Ansible uses everywhere (playbooks, `host_vars` filenames, output). If it's a real DNS name, Ansible connects to it directly. If it's a made-up alias, you must add `ansible_host=<address>` to tell Ansible where to connect. Aliases let you use short, stable names even when IPs change — you update one `ansible_host` instead of every playbook.

**Q: When should I use INI versus YAML for my inventory?**
A: INI is easier to read and write for small, flat inventories and is the most widely seen. YAML handles deep nesting and many variables better and is friendlier to generated files. They are equivalent in power; pick one per inventory and stay consistent. `ansible-inventory --list -y` can convert INI to YAML if you change your mind.

**Q: Why use `group_vars` and `host_vars` instead of variables in the inventory file?**
A: It separates *what machines exist* from *how they're configured*. The inventory stays a short, readable host list; variables live in small YAML files you can review, diff, and organize per group or per host. Variables inline in a large INI file become unreadable, and `[group:vars]` can't hold complex structures (lists, nested maps) as cleanly as YAML files.

**Q: Which variable wins if the same name is in `group_vars/all.yml`, `group_vars/webservers.yml`, and `host_vars/web01.yml`?**
A: The most specific source: `host_vars/web01.yml` beats `group_vars/webservers.yml`, which beats `group_vars/all.yml`. Where a host is in multiple groups at the same level, ties are broken alphabetically by group name unless you set `ansible_group_priority`. When unsure, run `ansible-inventory --host web01` to see the winning value.

**Q: Do I need a dynamic inventory?**
A: Only when the host list changes faster than you can maintain a file — autoscaling groups, ephemeral cloud instances, or when a source of truth (cloud API, CMDB) already exists. For a stable set of physical servers or VMs, a static inventory in Git is simpler, auditable, and perfectly fine.

**Q: What does `ansible all -m ping` actually test?**
A: That Ansible can SSH to each host with the configured user/key/port and that the host has a usable Python interpreter to run Ansible's module. It is not an ICMP ping — a firewall blocking ICMP doesn't affect it. `pong` means the whole connection chain works; `UNREACHABLE` is an SSH/connection problem; `FAILED` with a Python message is a missing interpreter.

**Q: Can I run Ansible against the machine I'm sitting on?**
A: Yes. Set `ansible_connection=local` for that host (for example `localhost ansible_connection=local`) and Ansible runs modules directly, with no SSH. It's useful for testing a playbook before aiming it at remote servers.

**Q: What does `[production:children]` do, and does the child group get duplicated?**
A: It declares that group `production` contains the *groups* listed beneath it, so every host in those groups is also in `production`. Hosts aren't duplicated — each host appears once, belonging to multiple groups. Variables set on `production` apply to all those hosts and rank below the more specific child-group variables.

**Q: Is the `roles/` directory required in a project?**
A: No — it's only needed once you use roles. It appears in the standard layout so the structure is ready; the next lesson explains what goes inside it.

## Practice / Exercise

**Core:**
1. Create the project skeleton from the walkthrough (`ansible.cfg`, `inventory/production/`, `inventory/staging/`, `playbooks/`, `roles/`).
2. Write an INI inventory with at least two groups (`webservers`, `dbservers`), a host range (`web[01:03]`), one alias host with `ansible_host`, and a parent group using `:children`. Check it with `ansible-inventory --graph` and explain what each line of the tree shows.
3. Add `[production:vars]` with `ansible_user` and `ansible_ssh_private_key_file`, then run `ansible-inventory --host <name>` for one host and find those variables in the output.
4. Create `group_vars/all.yml`, `group_vars/webservers.yml`, and a `host_vars/<host>.yml` that overrides one variable for a single host. Prove the override with `ansible-inventory --host` against that host and a sibling host.
5. Run `ansible-inventory --list -y` to print your inventory as YAML, then hand-write the equivalent YAML inventory in a new file and confirm `ansible-inventory --graph` produces the same tree for both.
6. Point at real lab VMs (or containers/VMs from earlier lessons) and run `ansible all -m ping`. If anything is `UNREACHABLE`, diagnose it using only the error message and your connection variables.
7. Write a short playbook (apt + service from Lesson 24) that targets `hosts: webservers`, run it with `--check`, then for real, using your inventory.

**Stretch:**
1. Write a custom dynamic inventory script (Python, using Lesson 19's `json`) that returns two groups and `_meta.hostvars`, make it executable, and verify it with `ansible-inventory --graph` and `--host`.
2. Install the `amazon.aws` or `google.cloud` collection, write the matching plugin YAML file, and run `ansible-inventory --graph` (if you have a cloud account; otherwise write the file and explain what each key does, including how `keyed_groups` or filters would build groups).
3. Make `-i inventory/` point at a *directory* containing both a static `hosts.ini` and your custom script, and confirm the merged result shows hosts from both sources.
4. Move one group's variables into a *directory* (`group_vars/webservers/`) containing two separate `.yml` files, and confirm both files' variables appear in `ansible-inventory --host`.
5. Build a second environment (`inventory/staging/`) with different values for the same variable names, and demonstrate running the same playbook against staging versus production by changing only `-i`.

## Further Reading

- [Ansible: How to build your inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)
- [Ansible: Working with dynamic inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_dynamic_inventory.html)
- [Ansible: Using variables (group_vars / host_vars)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html)
- [Ansible: Sample directory layout](https://docs.ansible.com/ansible/latest/tips_tricks/sample_setup.html)
- `man ansible-inventory`, `ansible-doc -t inventory -l` (list available inventory plugins)
