# Lesson 29: Ansible Configuration, Tags, and Optimizations

**Module:** Ansible – Configuration Management
**Duration:** 120-150 min
**Prerequisites:** Lessons 24-28 (Ansible intro/YAML, inventory, roles, conditionals/loops, blocks/error handling/templating)

## Learning Objectives

By end of lesson student can:
- Tune `ansible.cfg` (forks, timeout, remote_user, roles_path, callback plugins) and explain config file precedence
- Tag tasks, plays, and roles, and run or skip subsets with `--tags` / `--skip-tags`, including the special `always` and `never` tags
- Restrict a run to specific hosts with `--limit` and resume a failed run with `--start-at-task`
- Speed up playbooks using pipelining, fact caching, `async` + `poll`, and the `free` strategy
- Encrypt secrets with Ansible Vault and use vaulted data in playbooks, including inline `encrypt_string` values

## Topics

- `ansible.cfg` deep dive: `forks`, `timeout`, `remote_user`, `roles_path`, `callback_plugins`
- Tags: adding to tasks/plays/roles, `--tags`, `--skip-tags`, special tags (`always`, `never`)
- Limiting: `--limit` (hostname, group, pattern, `!exclude`), `--start-at-task`
- Performance: pipelining, fact caching (jsonfile/redis), `async` + `poll`, `strategy: free`
- Ansible Vault: `encrypt`/`decrypt`/`view`/`edit`/`rekey` files; using vault in playbooks

## Concepts

### `ansible.cfg` deep dive

Ansible reads its settings from an INI-style config file. It searches in this order and uses the **first one found** (files are not merged):

| Order | Location |
|---|---|
| 1 | `ANSIBLE_CONFIG` environment variable (path to a file) |
| 2 | `./ansible.cfg` in the current working directory |
| 3 | `~/.ansible.cfg` in the user's home directory |
| 4 | `/etc/ansible/ansible.cfg` system-wide |

Per-project `./ansible.cfg` checked into Git (Lesson 14) is the usual choice: everyone running the project gets identical behavior. One security caveat: Ansible ignores a `./ansible.cfg` in a world-writable directory, to stop a malicious file planted in a shared folder from silently altering behavior.

A typical project config:

```ini
[defaults]
inventory          = ./inventory/hosts.ini
remote_user        = deploy
roles_path         = ./roles:~/.ansible/roles
forks              = 20
timeout            = 30
host_key_checking  = False
retry_files_enabled = False
stdout_callback    = yaml
callbacks_enabled  = timer, profile_tasks

[ssh_connection]
pipelining = True
ssh_args   = -o ControlMaster=auto -o ControlPersist=60s
```

Key settings:

| Setting | Meaning |
|---|---|
| `forks` | How many hosts Ansible works on **in parallel** (default 5). Raising it speeds up runs across many hosts, at the cost of more CPU/memory on the control node |
| `timeout` | SSH connection timeout in seconds — how long to wait before declaring a host unreachable |
| `remote_user` | Default SSH user to connect as (overridable per host/group via `ansible_user`, Lesson 25) |
| `roles_path` | Colon-separated directories searched for roles (Lesson 26) |
| `inventory` | Default inventory path, so `-i` is no longer needed on every command |
| `host_key_checking` | `False` skips the SSH known_hosts prompt — convenient for ephemeral lab VMs, **risky** in production (disables man-in-the-middle protection, same `known_hosts` concept as Lesson 11) |
| `stdout_callback` | Output format of the run (`default`, `yaml`, `json`, `minimal`, ...) |
| `callbacks_enabled` | Extra **callback plugins** that add behavior without replacing the main output — e.g. `timer` (total runtime) and `profile_tasks` (per-task timing) |

**Callback plugins** hook into events during a run (task started, task failed, play finished). `profile_tasks` is the most useful for optimization work: it prints the slowest tasks at the end, showing exactly where time goes before you start tuning anything.

Precedence among setting sources, highest first: command-line flags (`-u`, `-f`) > environment variables (`ANSIBLE_FORKS=20`) > `ansible.cfg` > built-in defaults. `ansible-config dump --only-changed` shows which settings differ from defaults and where each came from; `ansible-config view` prints the active config file.

### Tags: running part of a playbook

A **tag** is a label attached to a task, play, block, or role. Tags let you run or skip subsets of a large playbook without editing it.

```yaml
- hosts: webservers
  tags: [web]                # play-level tag: applies to every task in this play
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
      tags: [packages, nginx]

    - name: Deploy config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags: [config, nginx]

    - name: Gather build info
      ansible.builtin.debug:
        msg: "always runs"
      tags: [always]

    - name: Wipe web root (dangerous)
      ansible.builtin.file:
        path: /var/www/html
        state: absent
      tags: [never, wipe]
```

Roles accept tags too, either in the `roles:` list (`- { role: nginx, tags: ['nginx'] }`) or via `include_role`/`import_role` with `apply`/`tags`. Tags on a block (Lesson 28) apply to every task inside it. Tags are **inherited**: a play-level tag is effectively on every task in that play.

Selecting at run time:

| Flag | Effect |
|---|---|
| `--tags nginx` | Run only tasks tagged `nginx` (plus `always` tasks) |
| `--skip-tags config` | Run everything except tasks tagged `config` (`never` tasks stay skipped) |
| `--tags config,packages` | Comma-separated: run tasks matching **any** listed tag |
| `--list-tags` | Print all tags found in the playbook, without running anything |
| `--list-tasks` | Print the tasks that would run (combine with `--tags` to preview) |

**Special tags:**

- `always` — the task runs on every invocation, even when `--tags` selects something else (skippable only via an explicit `--skip-tags always`). Use for must-run setup such as gathering a variable other tasks depend on.
- `never` — the task is skipped by default and only runs when one of its **other** tags is explicitly requested (`--tags wipe`). This is the standard pattern for dangerous or rarely wanted tasks (wipes, forced re-initialization) living safely inside the same playbook.

`tagged` and `untagged` are also valid selectors: `--tags untagged` runs only untagged tasks; `--tags tagged` runs only tasks that have any tag.

### Limiting which hosts a run touches

`--limit` (or `-l`) narrows an already-selected set of hosts without editing the inventory or playbook:

| Pattern | Meaning |
|---|---|
| `--limit web01` | One host |
| `--limit webservers` | One inventory group |
| `--limit 'web01,web02'` | Several hosts (comma or colon) |
| `--limit 'web*'` | Wildcard on host names |
| `--limit 'webservers:&production'` | Intersection: in `webservers` **and** `production` |
| `--limit 'webservers:!web03'` | Exclusion: all `webservers` except `web03` |
| `--limit @failed_hosts.txt` | Read hostnames from a file (Ansible writes `.retry`-style files only if `retry_files_enabled` is on) |

Quote patterns containing `!` or `&` so the shell (Lesson 4) does not interpret them. `--limit` restricts hosts **only**; the play's own `hosts:` line still has to match, so the effective set is the intersection of both.

`--start-at-task "Task name"` begins the run at the first task with exactly that name, skipping everything before it — handy to resume after fixing the cause of a mid-playbook failure without repeating earlier (idempotent but slow) tasks. `--step` prompts before each task (y/n/c), useful for careful, interactive first runs. `--check` (Lesson 24) combined with `--limit` is a safe way to preview a change on one canary host before rolling out fleet-wide.

### Performance optimizations

Slow Ansible runs usually come from SSH overhead, fact gathering, and serial waiting. The main levers:

**1. `forks`** — parallelism across hosts (above). The default of 5 is conservative; 20-50 is common for fleets, bounded by control-node resources.

**2. SSH pipelining and ControlPersist** — by default Ansible copies a module file to the target, runs it, and deletes it: several SSH operations per task. `pipelining = True` (in `[ssh_connection]`) feeds the module over the existing SSH connection's stdin, cutting round trips substantially. It requires that `requiretty` is not set in the target's sudoers (Lesson 6). `ControlMaster`/`ControlPersist` in `ssh_args` keeps one SSH connection open and reuses it across tasks instead of re-handshaking each time.

**3. Fact gathering and caching** — `gather_facts` collects hundreds of system variables at the start of every play, which costs seconds per host. Options:

- `gather_facts: false` on plays that do not use facts (call `setup` manually, with `gather_subset`, if only some are needed).
- **Fact caching**: store facts between runs so repeated runs skip re-collection.

```ini
[defaults]
gathering        = smart
fact_caching     = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout    = 3600
```

`jsonfile` writes one JSON file per host in the given directory (simple, no extra service). `redis` stores facts in a Redis server (shared across multiple control nodes or CI runners; requires the `redis` Python library and a reachable server). `gathering = smart` gathers facts only if not already cached/fresh.

**4. `async` and `poll`** — normally Ansible waits for each task to finish on every host before moving to the next task. For long-running work (package upgrades, large downloads, reboots), `async` lets the task run in the background on the target, with `poll` controlling how Ansible checks back:

```yaml
- name: Long-running upgrade
  ansible.builtin.apt:
    upgrade: dist
  async: 1800        # allow up to 30 min before Ansible gives up
  poll: 15           # check every 15 seconds (blocking, but survives SSH timeouts)

- name: Fire and forget
  ansible.builtin.command: /usr/local/bin/long_job.sh
  async: 3600
  poll: 0            # do not wait: start it and move on immediately
  register: long_job

- name: Check on it later
  ansible.builtin.async_status:
    jid: "{{ long_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 30
```

`poll: 0` means "launch and continue"; combine with `async_status` (and `register`, Lesson 27) to collect the result later. This also lets one playbook kick off long jobs on many hosts concurrently within a single task.

**5. Strategies** — the default `linear` strategy runs each task on **all** hosts before any host starts the next task (the slowest host sets the pace for every task). `strategy: free` lets each host run through the play as fast as it can independently, so fast hosts finish early. Use `free` when hosts are independent; keep `linear` when later tasks depend on all hosts completing earlier ones. `serial` (rolling batches, e.g. `serial: 2` or `serial: "25%"`) is a related control for staged rollouts that limits how many hosts update at once.

```yaml
- hosts: all
  strategy: free
  gather_facts: false
  tasks: [...]
```

### Ansible Vault: encrypting secrets

Playbooks and variable files are committed to Git, but passwords, API tokens, and private keys must not appear there in plain text. **Ansible Vault** encrypts files (or individual values) with AES256 using a password you supply; Ansible decrypts them transparently at run time.

| Command | Purpose |
|---|---|
| `ansible-vault create secrets.yml` | Create a new encrypted file (opens `$EDITOR`, Lesson 3) |
| `ansible-vault encrypt vars.yml` | Encrypt an existing plain-text file in place |
| `ansible-vault view secrets.yml` | Print decrypted contents without modifying the file |
| `ansible-vault edit secrets.yml` | Decrypt to a temp file in your editor, re-encrypt on save |
| `ansible-vault decrypt secrets.yml` | Permanently convert back to plain text (use sparingly) |
| `ansible-vault rekey secrets.yml` | Change the vault password of an already-encrypted file |
| `ansible-vault encrypt_string 'value' --name 'db_pass'` | Encrypt a single value, producing a `!vault` block to paste into a normal YAML file |

An encrypted file starts with a header like `$ANSIBLE_VAULT;1.1;AES256` followed by hex data; it is safe to commit (security rests on the password's strength and secrecy).

Supplying the password at run time:

| Method | Example |
|---|---|
| Interactive prompt | `ansible-playbook site.yml --ask-vault-pass` |
| Password file | `ansible-playbook site.yml --vault-password-file ~/.vault_pass` (file must be `chmod 600`, Lesson 3, and **never** committed) |
| Config setting | `vault_password_file = ~/.vault_pass` in `ansible.cfg` |
| Environment/executable | A script that prints the password (can fetch it from a secrets manager) |

**Vault IDs** (`--vault-id dev@prompt`, `--vault-id prod@~/.vault_prod`) label different passwords so dev and prod secrets can be encrypted with different keys within one project.

Common layout: keep one plain variable file referencing a vaulted one, so variable *names* stay greppable while *values* stay secret:

```
group_vars/
└── production/
    ├── vars.yml      # db_password: "{{ vault_db_password }}"
    └── vault.yml     # vault_db_password: S3cr3t!   (encrypted)
```

`encrypt_string` suits a single secret inside an otherwise readable file:

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  6638316239...
```

Note: Vault protects secrets **at rest**. Values are decrypted in memory on the control node and sent to targets, and can leak into logs or output unless tasks use `no_log: true`.

### Module recap: how the Ansible lessons fit together

- **24** — agentless, SSH push model; YAML; ad-hoc commands; first playbook (`--check`, `-v`)
- **25** — inventory (INI/YAML, groups, `group_vars`/`host_vars`), project layout, connection variables
- **26** — roles, handlers, defaults vs vars, `import_role`/`include_role`
- **27** — `when`, loops, `register`, `set_fact`, `debug`
- **28** — `block`/`rescue`/`always`, `failed_when`/`changed_when`, Jinja2 templates, filters, lookups
- **29** — config tuning, tags, limiting, performance, Vault — turning a working playbook into one that is **fast, selectable, and safe with secrets**

## Commands / Syntax Reference

| Command / setting | Purpose | Example |
|---|---|---|
| `ansible-config dump --only-changed` | Show non-default settings and their source | `ansible-config dump --only-changed` |
| `forks` / `-f` | Parallel hosts | `ansible-playbook site.yml -f 20` |
| `--tags` / `--skip-tags` | Select/skip tagged tasks | `--tags nginx --skip-tags config` |
| `--list-tags` / `--list-tasks` | Preview without running | `ansible-playbook site.yml --list-tags` |
| `--limit` / `-l` | Restrict hosts | `-l 'webservers:!web03'` |
| `--start-at-task` | Resume at a named task | `--start-at-task "Deploy config"` |
| `--step` | Confirm each task interactively | `ansible-playbook site.yml --step` |
| `pipelining = True` | Fewer SSH round trips | `[ssh_connection]` section |
| `fact_caching = jsonfile` | Cache facts between runs | `[defaults]` section |
| `async` / `poll` | Background long tasks | `async: 1800` / `poll: 15` |
| `strategy: free` | Hosts proceed independently | play-level keyword |
| `ansible-vault create/encrypt/view/edit/decrypt/rekey` | Manage encrypted files | `ansible-vault edit vault.yml` |
| `ansible-vault encrypt_string` | Encrypt one value | `ansible-vault encrypt_string 'x' --name 'pw'` |
| `--ask-vault-pass` / `--vault-password-file` | Provide vault password at run time | `--vault-password-file ~/.vault_pass` |

## Examples / Walkthrough

```bash
# --- project config: ansible.cfg in the project root ---
cat > ansible.cfg <<'EOF'
[defaults]
inventory           = ./inventory/hosts.ini
remote_user         = deploy
roles_path          = ./roles
forks               = 20
timeout             = 30
retry_files_enabled = False
stdout_callback     = yaml
callbacks_enabled   = timer, profile_tasks
gathering           = smart
fact_caching        = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout    = 3600

[ssh_connection]
pipelining = True
ssh_args   = -o ControlMaster=auto -o ControlPersist=60s
EOF

ansible-config view                      # confirm Ansible picked up this file
ansible-config dump --only-changed       # list every non-default value and where it came from
```

```yaml
# --- site.yml: tags, always/never, strategy ---
- name: Web tier
  hosts: webservers
  strategy: free
  tags: [web]
  tasks:
    - name: Record deploy start
      ansible.builtin.set_fact:
        deploy_started: "{{ ansible_date_time.iso8601 }}"
      tags: [always]

    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
      tags: [packages, nginx]

    - name: Deploy nginx config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Reload nginx
      tags: [config, nginx]

    - name: Wipe web root (only on explicit request)
      ansible.builtin.file:
        path: /var/www/html
        state: absent
      tags: [never, wipe]

  handlers:
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

```bash
# --- using tags ---
ansible-playbook site.yml --list-tags                 # discover: always, config, nginx, never, packages, wipe, web
ansible-playbook site.yml --list-tasks --tags config  # preview which tasks would run

ansible-playbook site.yml                              # everything except [never] tasks
ansible-playbook site.yml --tags nginx                 # install + config (+ the always task)
ansible-playbook site.yml --skip-tags packages         # everything but package installs
ansible-playbook site.yml --tags wipe                  # the ONLY way the dangerous task runs
```

```bash
# --- limiting and resuming ---
ansible-playbook site.yml --limit web01 --check        # dry run on one canary host
ansible-playbook site.yml --limit web01                # real run on the canary
ansible-playbook site.yml --limit 'webservers:!web01'  # roll out to the rest
ansible-playbook site.yml --limit 'webservers:&production'   # intersection of two groups

# playbook failed at "Deploy nginx config" -- fix the template, resume without redoing earlier tasks:
ansible-playbook site.yml --start-at-task "Deploy nginx config"
ansible-playbook site.yml --step                       # confirm each task manually (y/n/c)
```

```yaml
# --- performance: async long task, collect result later ---
- hosts: appservers
  gather_facts: false
  tasks:
    - name: Start long maintenance job on every host at once
      ansible.builtin.command: /usr/local/bin/reindex.sh
      async: 3600
      poll: 0
      register: reindex_job

    - name: Do other work meanwhile
      ansible.builtin.ping:

    - name: Wait for the reindex jobs to finish
      ansible.builtin.async_status:
        jid: "{{ reindex_job.ansible_job_id }}"
      register: reindex_result
      until: reindex_result.finished
      retries: 120
      delay: 30
```

```bash
# --- Ansible Vault ---
ansible-vault create group_vars/production/vault.yml
# editor opens; type:   vault_db_password: S3cr3tP@ss
# prompts for a new vault password (twice)

cat group_vars/production/vault.yml                     # header $ANSIBLE_VAULT;1.1;AES256 + hex: safe to commit
ansible-vault view group_vars/production/vault.yml      # decrypted view, file unchanged
ansible-vault edit group_vars/production/vault.yml      # edit and re-encrypt on save
ansible-vault rekey group_vars/production/vault.yml     # change the vault password

# plain file that references the vaulted variable (names visible, values hidden)
cat > group_vars/production/vars.yml <<'EOF'
db_password: "{{ vault_db_password }}"
EOF

# inline single value
ansible-vault encrypt_string 'S3cr3tP@ss' --name 'api_token'

# password file (never commit it)
echo 'my-vault-password' > ~/.vault_pass
chmod 600 ~/.vault_pass

# run with vault
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# use no_log so secrets do not leak into output
# - name: Create DB user
#   community.mysql.mysql_user:
#     name: app
#     password: "{{ db_password }}"
#   no_log: true
```

## Common Pitfalls

- **Committing the vault password file** — `~/.vault_pass` (or a project-local `.vault_pass`) in Git defeats the whole point of Vault. Keep it outside the repo or add it to `.gitignore` (Lesson 14), and `chmod 600` it.
- **Secrets leaking despite Vault** — vaulted values are decrypted in memory; a task that prints or registers them (`debug`, `-v` output, failing task diagnostics) writes plain text to the terminal or CI logs. Use `no_log: true` on tasks handling secrets.
- **Assuming `--tags` runs *only* the tagged tasks** — tasks tagged `always` still run. Conversely, a task with no tags at all does not run under `--tags x`. Use `--list-tasks --tags x` to preview before running.
- **Forgetting the `never` tag needs an extra tag to be reachable** — `tags: [never]` alone can never be selected conveniently; pair it with a second descriptive tag (`[never, wipe]`) and run `--tags wipe`.
- **Unquoted `--limit` patterns** — `--limit webservers:!web03` can trigger shell history expansion (`!`) or interpret `&` as a background operator. Always single-quote patterns: `--limit 'webservers:!web03'`.
- **Cranking `forks` far beyond control-node capacity** — 200 forks on a small VM can exhaust memory/CPU or SSH connection limits and make runs slower or flaky. Raise gradually and watch resource usage.
- **Enabling `pipelining` where sudoers has `requiretty`** — modules fail with sudo errors. Remove `requiretty` from the target's sudoers (or leave pipelining off for those hosts).
- **Fact caching serving stale facts** — cached facts can be hours old; a task that depends on current memory, IPs, or package state may act on outdated data. Choose a sensible `fact_caching_timeout`, or refresh with `gather_facts: true`/`meta: clear_facts` when freshness matters.
- **Using `strategy: free` with tasks that depend on other hosts** — if host B needs a value set by host A in an earlier task, `free` can let B run ahead and read it before it exists. Keep `linear` for cross-host dependencies.

## FAQ

**Q: Which `ansible.cfg` does Ansible use if there are several?**
A: Only the first found, in this order: `ANSIBLE_CONFIG` env var, `./ansible.cfg`, `~/.ansible.cfg`, `/etc/ansible/ansible.cfg`. Files are not merged. `ansible-config dump --only-changed` or `ansible --version` shows which file is active.

**Q: Does `--limit` override the `hosts:` line in the playbook?**
A: No, it intersects with it. A host must match both the play's `hosts:` pattern and the `--limit` pattern to be touched. If `--limit db01` is used on a play targeting `webservers`, nothing runs.

**Q: What is the difference between `--tags always` and the `always` tag?**
A: The `always` tag is a property of a task: it runs on every execution regardless of `--tags` selection. Passing `--tags always` on the command line merely selects those tasks (and combines with other tags). Day to day, you just set `tags: [always]` on the task.

**Q: When should I choose `async` with `poll: 0` versus a positive `poll`?**
A: A positive `poll` keeps the task blocking but resilient to SSH timeouts (good for one long job such as an upgrade or reboot-adjacent action). `poll: 0` fires the task and returns immediately, so you can launch many long jobs across hosts and verify later with `async_status` (good for parallelism and for work that outlives the connection).

**Q: Is Ansible Vault safe enough for production secrets?**
A: It provides solid AES256 encryption at rest for secrets stored in Git, and is fine for many teams. Weaknesses: a shared password, no per-secret access control or audit trail, and manual rotation. Larger organizations often pair or replace it with a dedicated secrets manager (HashiCorp Vault, cloud KMS) accessed through lookup plugins (Lesson 28), and Kubernetes covers the same problem later with External Secrets (Lesson 43).

**Q: Why does my encrypted file show a different ciphertext every time I re-encrypt it?**
A: Vault uses a random salt per encryption, so output differs even for identical content. That is expected, and also why Git diffs of vault files are noisy — a reason to keep vaulted variables in small, separate files.

**Q: How do I encrypt a whole project's vars without retyping the password constantly?**
A: Use `vault_password_file` in `ansible.cfg` (pointing to a protected file or a script) or `--vault-password-file` per run. In CI, inject the password from the pipeline's masked secret variables rather than storing a file in the repo.

**Q: `profile_tasks` or `timer` shows nothing — why?**
A: Callbacks must be enabled in `callbacks_enabled` (older versions used `callback_whitelist`), and the plugin name must match exactly. Run `ansible-config dump | grep -i callback` to confirm the setting took effect.

## Practice / Exercise

**Core:**
1. Create a project `ansible.cfg` setting `inventory`, `remote_user`, `forks = 10`, `timeout = 20`, and `roles_path`; verify with `ansible-config dump --only-changed`.
2. Enable `callbacks_enabled = timer, profile_tasks`, run an existing playbook, and identify the slowest task from the output.
3. Add tags to a playbook: `packages`, `config`, a play-level tag, an `always` task, and a `never,wipe` task. Use `--list-tags`, then run with `--tags config`, `--skip-tags packages`, and `--tags wipe`, confirming each behaves as expected.
4. Run the playbook with `--limit` against a single host, then `--limit 'group:!host'`, and explain the resulting host sets.
5. Force a failure partway through (bad template path), fix it, and resume using `--start-at-task`.
6. Turn on `pipelining = True` and `ControlPersist`, then compare total runtime (via `timer`) before and after on at least 3 hosts.
7. Configure `fact_caching = jsonfile` with a timeout; run twice and observe that the second run skips fact gathering time. Inspect the cached JSON files.
8. Create a vaulted `vault.yml` with `ansible-vault create`, reference its variable from a plain `vars.yml`, and run the playbook with `--ask-vault-pass`. Practice `view`, `edit`, and `rekey`.
9. Use `ansible-vault encrypt_string` to embed one encrypted value inline in a vars file and use it in a task with `no_log: true`.

**Stretch:**
1. Write a play with a deliberately slow task (`sleep 30`) across several hosts; compare `strategy: linear` vs `strategy: free` and explain the timing difference.
2. Rewrite the slow task using `async` with `poll: 0` plus `async_status`, and measure total run time versus the blocking version.
3. Set up two vault IDs (`dev`, `prod`) with different passwords, encrypt one file with each, and run a playbook that decrypts both using `--vault-id`.
4. Use a small executable script as `vault_password_file` that reads the password from an environment variable, and use it in a simulated CI run.
5. Add a `serial: 2` rolling update with a `--check` canary on one host first; document the exact sequence of commands for a safe production rollout.

## Further Reading

- [Ansible configuration settings reference](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)
- [Tags](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tags.html)
- [Patterns: targeting hosts and groups](https://docs.ansible.com/ansible/latest/inventory_guide/intro_patterns.html)
- [Asynchronous actions and polling](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_async.html)
- [Controlling playbook execution: strategies and more](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html)
- [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
- `man ansible-playbook`, `man ansible-vault`, `ansible-doc -l`
