# Lesson 26: Ansible Roles for Reusability

**Module:** Ansible – Configuration Management
**Duration:** 120-150 min
**Prerequisites:** Lesson 24 (Introduction to Ansible & YAML Syntax), Lesson 25 (Inventory Files & Folder Structure)

## Learning Objectives

By end of lesson student can:
- Explain what an Ansible role is and why it exists (reuse, structure, sharing)
- Describe every standard directory in a role and what belongs in it
- Generate a role skeleton with `ansible-galaxy init` and follow role naming conventions
- Fill in Galaxy metadata in `meta/main.yml`
- Write handlers, trigger them with `notify`, group them with `listen`, and force them early with `flush_handlers`
- Explain the difference between role `defaults/` and `vars/`, and reason about variable precedence
- Use a role from a playbook with the `roles` keyword, `import_role`, and `include_role`
- Declare role dependencies in `meta/main.yml`

## Topics

- Role directory structure: `tasks/`, `handlers/`, `defaults/`, `vars/`, `templates/`, `files/`, `meta/`
- Creating roles: `ansible-galaxy init`, naming conventions, Galaxy metadata
- Handlers: `notify`, `listen`; handler ordering; `flush_handlers`
- Role defaults vs vars: variable precedence, when to use each
- Using roles in playbooks: `roles` keyword, `import_role`, `include_role`; role dependencies

## Concepts

### Why roles exist

In Lesson 24 you wrote a playbook as one flat list of tasks. That works for 10 tasks. At 100 tasks across several servers, a single playbook becomes a wall of YAML nobody can read, and the same "install and configure nginx" block gets copy-pasted into every project that needs a web server.

A **role** is Ansible's unit of reuse: a directory with a fixed, conventional layout that bundles everything needed to configure *one thing* (a web server, a database, a baseline of security settings) — its tasks, its default settings, its handlers, its static files. A playbook then becomes short and readable: "apply the `common` role, then the `nginx` role, to the `webservers` group." The detail lives inside each role.

Three payoffs:

| Payoff | What it means |
|---|---|
| **Reuse** | Write the nginx logic once; apply it to any host, any project, any environment |
| **Structure** | Everyone on the team knows where to look: tasks in `tasks/`, settings in `defaults/` |
| **Sharing** | A role is self-contained, so it can be shared through a Git repo or Ansible Galaxy (the public role/collection hub) |

### Role directory structure

Ansible loads each part of a role from a fixed directory name — no import statements needed, the layout *is* the interface:

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml        # the tasks this role runs (entry point)
    ├── handlers/
    │   └── main.yml        # handlers (actions triggered by "notify")
    ├── defaults/
    │   └── main.yml        # default variable values — lowest precedence, easy to override
    ├── vars/
    │   └── main.yml        # internal variable values — high precedence, hard to override
    ├── templates/          # Jinja2 template files (used with the `template` module)
    ├── files/              # static files copied as-is (used with the `copy` module)
    └── meta/
        └── main.yml        # role metadata: author, platforms, dependencies
```

| Directory | Holds | Loaded automatically? |
|---|---|---|
| `tasks/` | The role's task list. `main.yml` is the entry point; it can `import_tasks`/`include_tasks` other files in the same folder | Yes — `tasks/main.yml` |
| `handlers/` | Handlers — tasks that only run when notified (see below) | Yes — `handlers/main.yml` |
| `defaults/` | Default values for the role's variables; the *public settings* of the role | Yes — `defaults/main.yml` |
| `vars/` | Variables the role considers internal constants; not meant to be overridden casually | Yes — `vars/main.yml` |
| `files/` | Static files. A `copy` task inside the role finds `src:` paths here automatically — no path prefix needed | Looked up by `copy` |
| `templates/` | Template files rendered with variables. The `template` module that renders them is covered in the lessons right after this one; for now just know the directory exists and is where templates will live | Looked up by `template` |
| `meta/` | Role metadata and dependencies | Yes — `meta/main.yml` |

You only create the directories you need. A tiny role might have just `tasks/main.yml`. Missing directories are simply skipped.

### Creating a role: `ansible-galaxy init`

`ansible-galaxy init <role_name>` generates the full skeleton — every directory above, with placeholder `main.yml` files and a `README.md`. The command is part of the `ansible-galaxy` tool, which also installs shared roles from Galaxy; `init` is the offline, scaffolding half of it.

```bash
ansible-galaxy init roles/nginx
```

This creates `roles/nginx/` populated with `tasks/`, `handlers/`, `defaults/`, `vars/`, `templates/`, `files/`, `meta/`, `tests/`, and a `README.md`. Delete what you don't use to keep the role tidy.

### Naming conventions

| Convention | Example | Why |
|---|---|---|
| Lowercase, letters/digits/underscores | `nginx`, `common`, `postgres_backup` | Role names must be valid identifiers; hyphens cause problems on Galaxy and in some contexts |
| One role = one job | `nginx`, not `webstack` | Small focused roles compose well; giant roles don't |
| Prefix role variables with the role name | `nginx_port`, `nginx_worker_processes` | Role variables share one global namespace — prefixing prevents two roles colliding on a generic name like `port` |
| Descriptive task names | `Install nginx package` | Task names appear in run output; they are your log |

The variable-prefix rule is the most important of these in practice: without it, the `nginx` role's `port` and the `postgres` role's `port` silently overwrite each other when both are applied to the same host.

### Galaxy metadata: `meta/main.yml`

`meta/main.yml` describes the role to humans and to Galaxy, and also holds role dependencies:

```yaml
galaxy_info:
  author: your_name
  description: Install and configure nginx
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
  galaxy_tags:
    - web
    - nginx

dependencies: []
```

`galaxy_info` is only strictly needed if you publish the role to Galaxy, but filling it in documents which OS and Ansible versions the role was written for. `dependencies` is functional (see below), not just documentation.

### Handlers: actions that run only on change

A **handler** is a task that does *not* run on its own — it runs only when another task *notifies* it, and only if that notifying task actually changed something on the host. The canonical use case: "restart nginx, but only if the configuration file was actually modified."

```yaml
# tasks/main.yml
- name: Copy nginx site config
  ansible.builtin.copy:
    src: default.conf
    dest: /etc/nginx/conf.d/default.conf
  notify: Restart nginx

# handlers/main.yml
- name: Restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

This ties directly to idempotency from Lesson 24: if the config file is already identical to the source, the `copy` task reports `ok` (not `changed`), the notify never fires, and nginx is **not** restarted needlessly. Run the playbook ten times, nginx restarts only the one time the config actually changed.

Key handler behaviours:

| Behaviour | Detail |
|---|---|
| **Run once, at the end** | Handlers run after *all* tasks in the play finish, no matter how many tasks notified them. Three tasks notifying `Restart nginx` → one restart. |
| **Only if changed** | A task reporting `ok` or `skipped` does not notify. |
| **Order = definition order** | Handlers fire in the order they are *defined* in `handlers/main.yml`, **not** the order they were notified. |
| **Names are the link** | `notify:` matches the handler's `name:` exactly (case-sensitive). |

**`listen`** lets several handlers subscribe to one shared "topic" name, so one notification triggers a group:

```yaml
# handlers/main.yml
- name: Validate nginx config
  ansible.builtin.command: nginx -t
  listen: "reload web stack"

- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
  listen: "reload web stack"
```

A task with `notify: "reload web stack"` now triggers both handlers, in definition order (validate first, then reload). This decouples the task from knowing which handlers exist — it just announces the topic.

**`flush_handlers`** forces pending handlers to run *immediately* instead of at the end of the play. It is a special `meta` task:

```yaml
- name: Copy nginx site config
  ansible.builtin.copy:
    src: default.conf
    dest: /etc/nginx/conf.d/default.conf
  notify: Restart nginx

- name: Run handlers now
  ansible.builtin.meta: flush_handlers

- name: Confirm nginx is running
  ansible.builtin.service:
    name: nginx
    state: started
```

Use it when a later task depends on the handler having already run (e.g. the service must be restarted with the new config *before* the next task talks to it).

### Role defaults vs vars, and variable precedence

Both `defaults/main.yml` and `vars/main.yml` define variables for the role, but they sit at opposite ends of Ansible's **variable precedence** — the rule for which value wins when the same variable is defined in several places.

| Location | Precedence | Intended use |
|---|---|---|
| `defaults/main.yml` | **Lowest** — anything overrides it | The role's *public settings* with sensible default values. Users are expected to override these. |
| Inventory `group_vars/` and `host_vars/` (Lesson 25) | Higher than defaults | Per-group / per-host values |
| Play `vars:` | Higher still | Values set in the playbook itself |
| `vars/main.yml` | **High** — hard to override | The role's *internal constants* that should not be changed by callers (e.g. a list of OS-specific package names) |
| `-e` / `--extra-vars` on the command line | **Highest** — beats everything | One-off overrides at run time |

Simplified low-to-high ladder for this lesson: `defaults` → `group_vars` → `host_vars` → play `vars` → role `vars` → `-e`.

Rule of thumb:
- Anything a user might reasonably want to change → **`defaults/`** (e.g. `nginx_port: 80`).
- Anything that is an implementation detail and changing it would break the role → **`vars/`**.

Because `defaults` is the weakest, a value in `group_vars/webservers.yml` cleanly overrides a role default without touching the role's code — exactly what you want: the role stays generic, the inventory supplies the specifics.

### Using roles in playbooks

**1. The `roles` keyword** — the classic, simplest form. Roles listed here run *before* the play's own `tasks`:

```yaml
- name: Configure web servers
  hosts: webservers
  become: true
  roles:
    - common
    - nginx
```

**2. `import_role`** — a *static* import. Ansible reads and merges the role's tasks when it parses the playbook, before execution begins. Written as a task:

```yaml
- name: Configure web servers
  hosts: webservers
  become: true
  tasks:
    - name: Apply nginx role
      ansible.builtin.import_role:
        name: nginx
```

**3. `include_role`** — a *dynamic* include. The role is loaded only when execution reaches this task:

```yaml
  tasks:
    - name: Apply nginx role
      ansible.builtin.include_role:
        name: nginx
```

| | `roles:` keyword | `import_role` | `include_role` |
|---|---|---|---|
| When processed | Parse time | Parse time (static) | Run time (dynamic) |
| Where it sits | Play-level list | Inside `tasks:` | Inside `tasks:` |
| Execution order | Always before `tasks:` | Exactly where it appears among tasks | Exactly where it appears among tasks |
| Role tasks visible to `--list-tasks` | Yes | Yes | No (not known until run) |

Default choice: `roles:` for straightforward playbooks; `import_role` when you need a role to run at a specific point *between* other tasks; `include_role` when the role needs to be decided or loaded at run time.

Variables can be passed to a role at the call site, overriding its `defaults`:

```yaml
  roles:
    - role: nginx
      nginx_port: 8080
```

### Role dependencies

A role can declare that it needs other roles applied first, in `meta/main.yml`:

```yaml
dependencies:
  - role: common
  - role: firewall
    firewall_open_ports: [80, 443]
```

When the playbook applies `nginx`, Ansible first applies `common` and `firewall` automatically. Dependencies run before the role's own tasks, and by default each dependency role runs only once per play even if several roles depend on it. This lets a role declare "I assume the baseline is present" instead of every playbook author remembering to list `common` first.

## Commands / Syntax Reference

| Command / Syntax | Purpose | Example |
|---|---|---|
| `ansible-galaxy init` | Scaffold a new role | `ansible-galaxy init roles/nginx` |
| `roles:` | Apply roles in a play | `roles: [common, nginx]` |
| `import_role` | Static role import as a task | `ansible.builtin.import_role: { name: nginx }` |
| `include_role` | Dynamic role include as a task | `ansible.builtin.include_role: { name: nginx }` |
| `notify:` | Trigger a handler on change | `notify: Restart nginx` |
| `listen:` | Subscribe a handler to a topic | `listen: "reload web stack"` |
| `meta: flush_handlers` | Run pending handlers now | `ansible.builtin.meta: flush_handlers` |
| `dependencies:` (meta) | Declare roles that must run first | `dependencies: [{ role: common }]` |
| `ansible-playbook --syntax-check` | Validate playbook syntax before running | `ansible-playbook site.yml --syntax-check` |
| `ansible-playbook --list-tasks` | Show tasks that would run | `ansible-playbook site.yml --list-tasks` |
| `ansible-playbook --check` | Dry run (Lesson 24) | `ansible-playbook site.yml --check` |

## Examples / Walkthrough

Goal: build an `nginx` role, then apply it (together with a small `common` role) from a playbook.

```bash
# --- project layout (builds on Lesson 25's structure) ---
mkdir -p ~/ansible-roles-demo && cd ~/ansible-roles-demo
mkdir roles

ansible-galaxy init roles/common
ansible-galaxy init roles/nginx

# remove directories we won't use, to keep the roles tidy
rm -rf roles/common/{tests,templates,vars,files}
rm -rf roles/nginx/{tests,templates,vars}
tree roles
```

```yaml
# --- roles/nginx/defaults/main.yml : public settings, easy to override ---
---
nginx_package: nginx
nginx_service: nginx
nginx_port: 80
```

```text
# --- roles/nginx/files/index.html : static page copied as-is ---
<!DOCTYPE html>
<html>
  <head><title>Managed by Ansible</title></head>
  <body><h1>Hello from the nginx role</h1></body>
</html>
```

```text
# --- roles/nginx/files/default.conf : static nginx server config ---
server {
    listen 80;
    server_name _;
    root /var/www/html;
    index index.html;
}
```

```yaml
# --- roles/nginx/tasks/main.yml ---
---
- name: Install nginx package
  ansible.builtin.apt:
    name: "{{ nginx_package }}"
    state: present
    update_cache: true

- name: Copy site config
  ansible.builtin.copy:
    src: default.conf                    # found automatically in roles/nginx/files/
    dest: /etc/nginx/conf.d/default.conf
    owner: root
    group: root
    mode: "0644"
  notify: Restart nginx

- name: Copy index page
  ansible.builtin.copy:
    src: index.html
    dest: /var/www/html/index.html
    mode: "0644"

- name: Ensure nginx is enabled and running
  ansible.builtin.service:
    name: "{{ nginx_service }}"
    state: started
    enabled: true
```

```yaml
# --- roles/nginx/handlers/main.yml ---
---
- name: Restart nginx
  ansible.builtin.service:
    name: "{{ nginx_service }}"
    state: restarted
```

```yaml
# --- roles/nginx/meta/main.yml : metadata + dependency on common ---
---
galaxy_info:
  author: student
  description: Install and configure nginx
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
  galaxy_tags:
    - web
    - nginx

dependencies:
  - role: common
```

```yaml
# --- roles/common/tasks/main.yml : tiny baseline role ---
---
- name: Install baseline packages
  ansible.builtin.apt:
    name:
      - curl
      - vim
      - htop
    state: present
    update_cache: true
```

```yaml
# --- site.yml : the playbook is now tiny ---
---
- name: Configure web servers
  hosts: webservers
  become: true
  roles:
    - nginx            # 'common' runs first automatically, via nginx's meta dependency
```

```bash
# --- run it ---
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --list-tasks      # see the task order: common first, then nginx
ansible-playbook site.yml --check           # dry run
ansible-playbook site.yml                   # real run

# run again immediately: everything reports "ok", nothing "changed", nginx NOT restarted (idempotent)
ansible-playbook site.yml
```

```yaml
# --- overriding a role default from the inventory (Lesson 25 group_vars) ---
# inventory/group_vars/webservers.yml
---
nginx_port: 8080
```

```yaml
# --- site.yml variants ---

# a) pass a variable directly at the role call
- name: Configure web servers
  hosts: webservers
  become: true
  roles:
    - role: nginx
      nginx_port: 8080

# b) import_role: role runs exactly where it appears, between other tasks
- name: Configure web servers
  hosts: webservers
  become: true
  tasks:
    - name: Install a utility first
      ansible.builtin.apt:
        name: tree
        state: present
    - name: Apply nginx role now
      ansible.builtin.import_role:
        name: nginx

# c) include_role: dynamic, loaded when execution reaches this task
- name: Configure web servers
  hosts: webservers
  become: true
  tasks:
    - name: Apply nginx role
      ansible.builtin.include_role:
        name: nginx
```

```yaml
# --- flush_handlers: restart BEFORE the next task, not at end of play ---
- name: Copy site config
  ansible.builtin.copy:
    src: default.conf
    dest: /etc/nginx/conf.d/default.conf
  notify: Restart nginx

- name: Run handlers now
  ansible.builtin.meta: flush_handlers

- name: Make sure nginx is running after the restart
  ansible.builtin.service:
    name: nginx
    state: started
```

```yaml
# --- listen: one notification, several handlers ---
# handlers/main.yml
- name: Validate nginx config
  ansible.builtin.command: nginx -t
  listen: "reload web stack"

- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
  listen: "reload web stack"

# tasks/main.yml
- name: Copy site config
  ansible.builtin.copy:
    src: default.conf
    dest: /etc/nginx/conf.d/default.conf
  notify: "reload web stack"
```

## Common Pitfalls

- **Notify name doesn't match the handler name** — `notify: restart nginx` will not trigger a handler named `Restart nginx`. Matching is exact and case-sensitive, and a typo fails silently (the handler just never runs). Copy the name, or use `listen` with a topic string.
- **Expecting handlers to run immediately** — handlers run once at the *end* of the play. If a later task needs the restarted service, add `meta: flush_handlers` between them.
- **Assuming handlers run in notification order** — they run in the order they are *defined* in `handlers/main.yml`. Define dependent handlers (validate, then reload) in the order you want them to execute.
- **Putting overridable settings in `vars/` instead of `defaults/`** — `vars/main.yml` has very high precedence, so `group_vars` and play `vars` cannot override it, only `-e` can. Users then find the role "ignores" their inventory settings. Public settings belong in `defaults/`.
- **Generic variable names inside roles** — a role variable named `port` or `version` collides with another role's variable of the same name because role variables share one namespace. Prefix with the role name: `nginx_port`.
- **Handler fires even though the config looks unchanged — or never fires** — handlers only trigger when the notifying task reports `changed`. If a task is not idempotent (always reports changed) the handler restarts the service on every run; if the task reports `ok`, no restart happens even if you expected one. Read the play recap to see what actually changed.
- **Leaving `ansible-galaxy init` placeholder content in place** — the generated `README.md`, `tests/`, and sample `main.yml` comments look like real content. Delete unused directories and fill in the README, or the role misleads the next person.
- **Forgetting dependencies pull in extra roles** — applying `nginx` also applies `common` because of `meta/main.yml`. If `common` does something surprising, the cause is not visible in `site.yml`. Check `meta/` when a role does more than its own tasks explain.
- **Running a role with no `become: true` when tasks need root** — `apt`, writing under `/etc`, and managing services require privilege escalation; without `become` they fail with permission errors.

## FAQ

**Q: What is the difference between a role and a playbook?**
A: A playbook says *which hosts* get *which roles* (and optionally some extra tasks). A role is a reusable package of the tasks, defaults, handlers, and files to configure one thing. Playbooks are the entry point you run; roles are the building blocks they call.

**Q: Where does Ansible look for roles?**
A: First in a `roles/` directory next to the playbook, then in the paths listed in the `roles_path` setting in `ansible.cfg` (configuration is covered in Lesson 29), then in default system locations. Keeping a `roles/` folder beside your playbooks, as in this lesson, works without any configuration.

**Q: Do I have to use all the role directories?**
A: No. Only create what the role needs. A role with just `tasks/main.yml` is perfectly valid. `ansible-galaxy init` creates everything as a convenience; deleting unused folders is normal.

**Q: What is the difference between `files/` and `templates/`?**
A: `files/` holds static files copied byte-for-byte with the `copy` module. `templates/` holds files that contain variables and are rendered per host with the `template` module. The `template` module and Jinja2 syntax are introduced in the following lessons; in this lesson we use `files/` and `copy` only.

**Q: Why use `import_role` or `include_role` when the `roles:` keyword already works?**
A: `roles:` always runs roles before the play's `tasks`. If you need a role to run at a specific point *between* ordinary tasks, you must use `import_role` or `include_role` as a task. The difference between those two is timing: import is resolved when the playbook is parsed, include when execution reaches it.

**Q: Why does my handler run only once even though three tasks notified it?**
A: By design. Handlers are de-duplicated: no matter how many tasks notify the same handler, it runs a single time at the end of the play. This is what prevents three config changes from causing three restarts.

**Q: What happens to handlers if a task later in the play fails?**
A: By default, if a play fails on a host, handlers that were already notified do *not* run for that host, because the play aborts before reaching the handler phase. Use `meta: flush_handlers` before the risky step if a handler must run first. (Fine-grained failure handling is covered in a later lesson.)

**Q: Where should I put variables that differ per environment (dev vs prod)?**
A: In the inventory — `group_vars/` or `host_vars/` (Lesson 25) — not inside the role. The role's `defaults/` supplies sensible fallbacks, and inventory variables override them. That keeps the role generic and reusable across environments.

**Q: Can I install roles written by other people?**
A: Yes. `ansible-galaxy install <namespace.role_name>` downloads a role from Galaxy, or from a Git repository via a requirements file. Treat third-party roles like any dependency: read them before running them against your servers, and pin versions.

## Practice / Exercise

**Core:**
1. Run `ansible-galaxy init roles/nginx` and list the generated structure with `tree`. In a sentence each, state what goes in `tasks/`, `handlers/`, `defaults/`, `vars/`, `files/`, and `meta/`.
2. Delete the directories the role will not use (`tests/`, `templates/`, `vars/`).
3. Fill in `defaults/main.yml` with `nginx_package`, `nginx_service`, and `nginx_port`, using the role-name prefix convention.
4. Write `tasks/main.yml` that installs nginx with `apt`, copies a static `index.html` from `files/` with `copy`, and ensures the service is started and enabled with `service`.
5. Write a handler `Restart nginx` in `handlers/main.yml`, and make the config-copy task `notify` it. Run the playbook twice; confirm from the play recap that the handler fires on the first run only when the config changed, and not at all on the second run.
6. Write a `site.yml` that applies the role to a `webservers` group using the `roles:` keyword, then run `--syntax-check`, `--list-tasks`, `--check`, and finally a real run.
7. Override `nginx_port` from `group_vars/webservers.yml`, then again from the role call in `site.yml`, then again with `-e nginx_port=9090`. Run each and write down which value won and why, using the precedence ladder.
8. Create a second tiny role `common` that installs `curl` and `vim`, and declare it as a dependency in `roles/nginx/meta/main.yml`. Run `--list-tasks` and confirm `common` runs before `nginx` without being listed in `site.yml`.
9. Fill in `galaxy_info` in `roles/nginx/meta/main.yml` (author, description, license, platforms).

**Stretch:**
1. Rewrite `site.yml` three ways — `roles:`, `import_role`, `include_role` — and compare the output of `--list-tasks` for each. Explain why one of them shows fewer tasks.
2. Add two handlers sharing a `listen` topic ("validate config", then "reload nginx") and one task that notifies the topic. Confirm both run, in definition order.
3. Add a `meta: flush_handlers` task after the config copy, followed by a task that depends on the restart having already happened. Reorder the handlers in `handlers/main.yml` and observe how execution order changes.
4. Move one value (e.g. an OS-specific package name) from `defaults/` to `vars/` and show that `group_vars` can no longer override it. Explain, in your own words, which file the value belongs in and why.
5. Make `nginx_package` collide on purpose: give a second role a variable also called `nginx_package` with a different value, apply both roles, and describe the symptom. Then fix it with role-name prefixes.

## Further Reading

- [Ansible docs: Roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [Ansible docs: Handlers](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)
- [Ansible docs: Variable precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence)
- [Ansible Galaxy documentation](https://docs.ansible.com/ansible/latest/galaxy/user_guide.html)
- `man ansible-galaxy`, `ansible-galaxy init --help`
