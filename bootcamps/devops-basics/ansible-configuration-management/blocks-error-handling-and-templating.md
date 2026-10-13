# Lesson 28: Blocks, Error Handling, and Templating

**Module:** Ansible – Configuration Management
**Duration:** 120-150 min
**Prerequisites:** Lessons 24-27 (Intro & YAML, Inventory, Roles & Handlers, Conditionals/Loops/register/set_fact/debug)

## Learning Objectives

By end of lesson student can:
- Group tasks with `block` and handle failures with `rescue` and `always`, and explain when this beats `ignore_errors`
- Control what counts as failed or changed with `ignore_errors`, `failed_when`, and `changed_when`
- Render configuration files from Jinja2 templates using the `template` module, including loops and conditionals inside the template
- Transform values with Jinja2 filters (`default`, `upper`, `lower`, `join`, `selectattr`, `map`, `to_json`)
- Pull data from outside the play with `lookup()` (`file`, `env`, `template`, `password`)

## Topics

- `block`/`rescue`/`always`: structured error handling; use cases vs `ignore_errors`
- `ignore_errors`, `failed_when`, `changed_when`: customizing task behavior
- Jinja2 templates: `template` module, `.j2` files, variable substitution in files
- Jinja2 filters: `default()`, `upper()`, `lower()`, `join()`, `selectattr()`, `map()`, `to_json`
- Lookups: `lookup('file')`, `lookup('env')`, `lookup('template')`, `lookup('password')`

## Concepts

### Why error handling matters in Ansible

By default, the moment a task fails on a host, Ansible stops running the remaining tasks **for that host** (other hosts continue) and marks the host as failed. That default is safe, but real automation often needs finer control: "try this risky step, and if it fails, roll back and notify," or "this command returns exit code 1 when there's nothing to do — that's not a real failure." Ansible gives three layers of control: **blocks** (group tasks and attach recovery logic), **per-task overrides** (`ignore_errors`, `failed_when`, `changed_when`), and **handlers** (Lesson 26) for reacting to change.

### `block`, `rescue`, `always`

A `block` groups several tasks into one unit. It can carry directives that apply to **every task inside** (a shared `when`, `become`, `vars`, `environment`) — a real readability win over repeating the same line on five tasks. Blocks also bring structured error handling, modeled on try/catch/finally from programming languages:

| Section | Runs when |
|---|---|
| `block` | Always starts here; stops at the first failed task inside it |
| `rescue` | Only if a task in `block` failed — the recovery path |
| `always` | Every time, whether `block` succeeded, failed, or was rescued — the cleanup path |

```yaml
- name: Deploy with rollback
  block:
    - name: Upload new release
      ansible.builtin.copy:
        src: app-v2.tar.gz
        dest: /opt/app/releases/

    - name: Restart application
      ansible.builtin.service:
        name: myapp
        state: restarted

  rescue:
    - name: Report failure
      ansible.builtin.debug:
        msg: "Deploy failed on {{ inventory_hostname }}, rolling back"

    - name: Restore previous release
      ansible.builtin.command: /opt/app/rollback.sh

  always:
    - name: Record deploy attempt
      ansible.builtin.lineinfile:
        path: /var/log/deploys.log
        line: "{{ ansible_date_time.iso8601 }} deploy attempted"
        create: true
```

Two details worth teaching explicitly:

- **A successfully handled `rescue` makes the play continue.** Once `rescue` completes without failing, Ansible treats the block as recovered — the host is *not* marked failed, and the next task after the block runs. This is the key behavioral difference from simply failing.
- **Inside `rescue`, special variables describe what went wrong:** `ansible_failed_task` (the task that failed, including `.name`) and `ansible_failed_result` (its full result, including error message/`msg` and return code) — useful for building a meaningful error report.

**Block vs `ignore_errors`:** `ignore_errors: true` on a task says "if this fails, pretend it didn't and keep going" — the failure is silently swallowed, nothing reacts to it, and later tasks may run on a broken foundation. `block`/`rescue` says "if this fails, *do something specific about it*." Use `ignore_errors` only for genuinely optional steps; use `rescue` whenever the failure deserves a response.

### `ignore_errors`, `failed_when`, `changed_when`

Per-task controls that redefine what Ansible reports:

| Directive | Effect |
|---|---|
| `ignore_errors: true` | Task failure is recorded (shows as `...ignoring`) but doesn't stop the play for that host |
| `failed_when: <expression>` | Replaces Ansible's default failure rule (non-zero exit code / module error) with your own condition |
| `changed_when: <expression>` | Replaces the default "did this change something" rule — `false` makes a task never report `changed` |

`failed_when` and `changed_when` shine with `command`/`shell` modules, which know nothing about your command's semantics: they report `changed` on every run and `failed` on any non-zero exit code. Real tools often disagree with those defaults — `grep` returns exit 1 for "no match" (not an error), a custom script prints "already configured" on stdout (not a change). Combined with `register` (Lesson 27), you inspect `result.rc`, `result.stdout`, or `result.stderr` and decide:

```yaml
- name: Check whether feature flag is set
  ansible.builtin.command: grep -q FEATURE_X /etc/app.conf
  register: flag_check
  failed_when: flag_check.rc not in [0, 1]    # rc 1 = "not found" is fine; rc 2+ is a real error
  changed_when: false                          # a read-only check never "changes" anything
```

`changed_when: false` on read-only commands is a best practice: it keeps the play recap honest (and keeps `--check` runs and idempotency checks clean).

### Jinja2 templates and the `template` module

Ansible uses **Jinja2** as its templating engine — the same `{{ variable }}` syntax already used in play YAML (Lesson 24-27). A **template file** (conventionally `.j2` extension, stored in a role's `templates/` directory — Lesson 26) is a text file containing static text plus Jinja2 expressions; the `template` module renders it **on the control node** using the host's variables, then copies the finished file to the target host.

```yaml
- name: Render nginx config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
    validate: nginx -t -c %s
  notify: Reload nginx
```

`template` vs `copy`: `copy` transfers a file byte-for-byte; `template` runs it through Jinja2 first, so one template produces correct, host-specific files across a whole inventory. `template` is idempotent — it re-renders each run and only reports `changed` (and only triggers the `notify` handler) when the rendered output differs from what's on disk. The optional `validate` parameter runs a check command against the rendered file *before* it replaces the live one (`%s` is the temp file path) — a failed validation aborts the change, so a typo in a template can't take down nginx.

Jinja2 has three delimiter types:

| Syntax | Purpose |
|---|---|
| `{{ expression }}` | Output a value (variable, filter result, expression) |
| `{% statement %}` | Control logic: `if`, `for`, `set`, `include` — outputs nothing itself |
| `{# comment #}` | Template comment, not rendered into the output |

Inside templates you can use variables from inventory, `group_vars`/`host_vars`, play `vars`, role defaults, registered results, and **facts** (`ansible_hostname`, `ansible_default_ipv4.address`, `ansible_processor_vcpus`, etc.). Control structures:

```jinja
{% if enable_gzip %}
gzip on;
{% else %}
gzip off;
{% endif %}

{% for server in backend_servers %}
server {{ server.host }}:{{ server.port }};
{% endfor %}
```

Whitespace note: `{%` statements leave blank lines in the output by default. The `-` modifier (`{%- ... -%}`) trims surrounding whitespace, and the `template` module's `trim_blocks`/`lstrip_blocks` options (on by default in Ansible) already remove the newline after a block tag and leading spaces before one — usually enough for tidy output.

### Jinja2 filters

A **filter** transforms a value, applied with the pipe: `{{ value | filter }}`. Filters chain left to right and take arguments in parentheses.

| Filter | What it does | Example |
|---|---|---|
| `default(x)` | Fallback when a variable is undefined (add `true` to also fall back on empty/falsey values) | `{{ http_port \| default(80) }}` |
| `upper` / `lower` | Change case | `{{ env_name \| upper }}` → `PROD` |
| `join(sep)` | Combine a list into one string | `{{ ['a','b','c'] \| join(',') }}` → `a,b,c` |
| `selectattr(attr, test, val)` | Keep list items whose attribute passes a test | `{{ users \| selectattr('active') }}` |
| `map(attribute='x')` | Extract one attribute from every item | `{{ users \| map(attribute='name') }}` |
| `to_json` / `to_nice_json` | Serialize a variable as JSON (nice = indented) | `{{ config \| to_nice_json }}` |
| `to_yaml` / `to_nice_yaml` | Serialize as YAML | `{{ config \| to_nice_yaml }}` |

`selectattr` and `map` are almost always used together to slice structured data: "of all users, keep the active ones, then give me just their names":

```jinja
{{ users | selectattr('active') | map(attribute='name') | join(', ') }}
```

`selectattr` and `map` return generators — add `| list` when you need an actual list (for example to feed `loop`). `default()` is the filter you'll use most: it makes templates resilient to optional variables instead of failing with "undefined variable".

### Lookups

A **lookup plugin** fetches data from **outside the play's variables**, evaluated on the **control node** (not the managed host), via `lookup('plugin', 'argument')`:

| Lookup | Returns |
|---|---|
| `lookup('file', 'path')` | Contents of a file on the control node (e.g. a public SSH key) |
| `lookup('env', 'NAME')` | Value of an environment variable on the control node |
| `lookup('template', 'file.j2')` | A rendered template as a string (rather than writing it to a file) |
| `lookup('password', 'path length=20')` | Generates a random password on first use, **stores it in a file on the control node**, and returns that same value on every later run — giving idempotent, stable generated credentials |

```yaml
- name: Install admin SSH key
  ansible.posix.authorized_key:
    user: admin
    key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

- name: Set API token from controller environment
  ansible.builtin.lineinfile:
    path: /etc/app/env
    line: "API_TOKEN={{ lookup('env', 'APP_API_TOKEN') }}"
```

Because lookups run on the control node, the file/env var must exist **there**, not on the target. `lookup()` returns a string; use `query()` (or `lookup(..., wantlist=True)`) when you want a list.

## Commands / Syntax Reference

| Syntax | Purpose | Example |
|---|---|---|
| `block:` | Group tasks, share directives | `block: [ ...tasks ]` |
| `rescue:` | Recovery tasks if block failed | `rescue: [ ...tasks ]` |
| `always:` | Tasks that run regardless | `always: [ ...tasks ]` |
| `ignore_errors: true` | Continue past a failed task | task-level |
| `failed_when:` | Custom failure condition | `failed_when: r.rc != 0 and 'ok' not in r.stdout` |
| `changed_when:` | Custom changed condition | `changed_when: false` |
| `ansible_failed_task` / `ansible_failed_result` | Failure details inside `rescue` | `{{ ansible_failed_result.msg }}` |
| `template:` module | Render `.j2` to a host | `src: app.conf.j2` / `dest: /etc/app.conf` |
| `{{ }}` / `{% %}` / `{# #}` | Jinja2 output / logic / comment | see walkthrough |
| `\| default(x)` | Fallback value | `{{ port \| default(8080) }}` |
| `\| selectattr \| map \| join` | Filter/extract/combine data | see walkthrough |
| `\| to_json` / `\| to_nice_json` | Serialize to JSON | `{{ cfg \| to_nice_json }}` |
| `lookup('file'\|'env'\|'template'\|'password', ...)` | External data on control node | `lookup('env','HOME')` |

## Examples / Walkthrough

```yaml
# --- playbook: error handling with block/rescue/always ---
# file: deploy.yml
- name: Deploy application with rollback safety
  hosts: webservers
  become: true

  tasks:
    - name: Deploy block
      block:
        - name: Stop application
          ansible.builtin.service:
            name: myapp
            state: stopped

        - name: Install new version
          ansible.builtin.copy:
            src: files/myapp-2.0.jar
            dest: /opt/myapp/myapp.jar
            backup: true               # keeps a timestamped copy for rollback

        - name: Start application
          ansible.builtin.service:
            name: myapp
            state: started

        - name: Health check
          ansible.builtin.uri:
            url: "http://localhost:8080/health"
            status_code: 200
          retries: 5
          delay: 3
          register: health
          until: health.status == 200

      rescue:
        - name: Show what failed
          ansible.builtin.debug:
            msg: >-
              Task '{{ ansible_failed_task.name }}' failed on {{ inventory_hostname }}:
              {{ ansible_failed_result.msg | default('no message') }}

        - name: Roll back to previous jar
          ansible.builtin.command: cp /opt/myapp/myapp.jar.bak /opt/myapp/myapp.jar   # simplified example

        - name: Restart on old version
          ansible.builtin.service:
            name: myapp
            state: restarted

      always:
        - name: Append to deploy log
          ansible.builtin.lineinfile:
            path: /var/log/deploys.log
            line: "{{ ansible_date_time.iso8601 }} deploy attempted on {{ inventory_hostname }}"
            create: true
```

```yaml
# --- shared directives on a block (DRY) ---
- name: Database tasks
  block:
    - name: Install postgres
      ansible.builtin.apt: { name: postgresql, state: present }
    - name: Ensure postgres running
      ansible.builtin.service: { name: postgresql, state: started, enabled: true }
  when: "'db' in group_names"
  become: true
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

```yaml
# --- ignore_errors vs failed_when vs changed_when ---
- name: Optional cleanup (don't care if it fails)
  ansible.builtin.command: rm -rf /tmp/old-cache
  ignore_errors: true

- name: Search config for a setting (grep: rc 1 = no match, which is OK)
  ansible.builtin.command: grep -q "max_connections" /etc/postgresql/postgresql.conf
  register: grep_result
  failed_when: grep_result.rc > 1       # only rc >= 2 is a genuine error
  changed_when: false                    # reading never changes anything

- name: Run migration script that prints 'nothing to do' when up-to-date
  ansible.builtin.command: /opt/app/migrate.sh
  register: migrate
  changed_when: "'nothing to do' not in migrate.stdout"
  failed_when: "'ERROR' in migrate.stdout"

- name: Report
  ansible.builtin.debug:
    msg: "Migration changed state: {{ migrate.changed }}"
```

```
# --- project layout for the template example ---
nginx-site/
├── site.yml
├── group_vars/
│   └── webservers.yml
└── templates/
    └── nginx.conf.j2
```

```yaml
# --- group_vars/webservers.yml ---
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_gzip: true
server_name: example.com
document_root: /var/www/example
upstream_servers:
  - { host: 10.0.0.11, port: 8080, weight: 3 }
  - { host: 10.0.0.12, port: 8080, weight: 1 }
  - { host: 10.0.0.13, port: 8080, weight: 1, backup: true }
ssl_enabled: false
# ssl_certificate intentionally left undefined to demonstrate default()
```

```jinja
{# --- templates/nginx.conf.j2 --- #}
{# {{ ansible_managed }} is a built-in string warning humans not to hand-edit this file #}
# {{ ansible_managed }}
# Generated for {{ inventory_hostname }} ({{ ansible_default_ipv4.address | default('unknown-ip') }})

worker_processes {{ nginx_worker_processes | default('auto') }};

events {
    worker_connections {{ nginx_worker_connections | default(512) }};
}

http {
    {% if nginx_gzip | default(false) %}
    gzip on;
    {% else %}
    gzip off;
    {% endif %}

    upstream app_backend {
    {% for srv in upstream_servers %}
        server {{ srv.host }}:{{ srv.port }} weight={{ srv.weight | default(1) }}{{ ' backup' if srv.backup | default(false) else '' }};
    {% endfor %}
    }

    server {
        listen {{ '443 ssl' if ssl_enabled | default(false) else '80' }};
        server_name {{ server_name | lower }};
        root {{ document_root }};

        {% if ssl_enabled | default(false) %}
        ssl_certificate     {{ ssl_certificate | default('/etc/ssl/certs/default.pem') }};
        ssl_certificate_key {{ ssl_certificate_key | default('/etc/ssl/private/default.key') }};
        {% endif %}

        location / {
            proxy_pass http://app_backend;
            proxy_set_header Host $host;
        }
    }
}
{# Summary comment (not rendered): active backends = {{ upstream_servers | selectattr('backup', 'undefined') | map(attribute='host') | join(', ') }} #}
```

```yaml
# --- site.yml: apply the template, validate, and reload via handler ---
- name: Configure nginx from template
  hosts: webservers
  become: true

  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Render nginx.conf
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: "0644"
        validate: nginx -t -c %s          # reject a broken render before it goes live
      notify: Reload nginx

  handlers:
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

```yaml
# --- filters in plays ---
- name: Filter demonstrations
  hosts: localhost
  gather_facts: false
  vars:
    env_name: Production
    ports: [80, 443, 8080]
    users:
      - { name: alice, active: true,  role: admin }
      - { name: bob,   active: false, role: dev }
      - { name: carol, active: true,  role: dev }
    app_config:
      debug: false
      workers: 4
  tasks:
    - ansible.builtin.debug:
        msg:
          - "{{ env_name | upper }}"                                   # PRODUCTION
          - "{{ env_name | lower }}"                                   # production
          - "{{ ports | join(',') }}"                                  # 80,443,8080
          - "{{ undefined_var | default('fallback') }}"                # fallback
          - "{{ users | selectattr('active') | map(attribute='name') | join(', ') }}"   # alice, carol
          - "{{ users | selectattr('role', 'equalto', 'dev') | map(attribute='name') | list }}"
          - "{{ app_config | to_json }}"                               # {"debug": false, "workers": 4}

    - name: Write config as pretty JSON using a filter
      ansible.builtin.copy:
        content: "{{ app_config | to_nice_json }}"
        dest: /tmp/app-config.json
```

```yaml
# --- lookups ---
- name: Lookup demonstrations
  hosts: webservers
  become: true
  tasks:
    - name: Deploy controller's public key
      ansible.posix.authorized_key:
        user: deploy
        key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

    - name: Show a controller-side environment variable
      ansible.builtin.debug:
        msg: "Deploying as {{ lookup('env', 'USER') }}"

    - name: Generate (once) and reuse a stable DB password
      ansible.builtin.debug:
        msg: "{{ lookup('password', 'credentials/db_pass.txt length=20 chars=ascii_letters,digits') }}"
        # first run: creates credentials/db_pass.txt on the CONTROL node; later runs return the same value

    - name: Render a template into a variable instead of a file
      ansible.builtin.copy:
        content: "{{ lookup('template', 'motd.j2') }}"
        dest: /etc/motd
```

```bash
# run it
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --check --diff    # preview template changes without applying
```

## Common Pitfalls

- **Using `ignore_errors: true` as a general "make it green" switch** — the failure is swallowed, nothing reacts, and later tasks run on a broken foundation. Prefer `block`/`rescue` when the failure deserves a response, or `failed_when` to define the *correct* failure condition for commands whose exit codes don't match their meaning.
- **Expecting `rescue` to run when a *rescue* task itself fails, or after `always`** — `rescue` only triggers on failure inside `block`; a failure inside `rescue` marks the host failed (the block was not recovered). `always` runs regardless, but a failing task inside `always` also fails the host.
- **Forgetting that `command`/`shell` always report `changed`** — every run shows `changed=1`, handlers fire unnecessarily, and idempotency checks fail. Use `changed_when` (and `creates:`/`removes:` arguments where they fit) so the recap tells the truth; use `changed_when: false` for pure reads.
- **Writing `failed_when` without `register`** — `failed_when` conditions usually inspect the task's own result (`result.rc`, `result.stdout`), which requires `register: result` on that task. Without it, the variable doesn't exist and the task errors with an undefined-variable message.
- **Unquoted Jinja2 at the start of a YAML value** — `msg: {{ var }}` is parsed by YAML as the start of a dictionary and fails. Always quote: `msg: "{{ var }}"`.
- **Whitespace mess in rendered templates** — `{% for %}`/`{% if %}` tags can leave blank lines or break indentation (fatal for YAML targets). Use `{%- -%}` trimming where needed, test with `--check --diff`, and view the result with `cat -A` (Lesson 1/4 tools) when indentation matters.
- **Referencing an undefined variable in a template without `default()`** — the play fails with "'xyz' is undefined". For optional settings always provide `default(...)`; remember `default(x, true)` is needed to also replace *empty strings*.
- **Assuming lookups run on the target host** — `lookup('file', ...)` and `lookup('env', ...)` read from the **control node**. A path that exists only on the managed host fails with "could not locate file in lookup". To read remote files use `slurp` or `command` + `register`.
- **Treating `lookup('password')` as secret storage** — it writes the generated password in **plain text** to a file on the control node. It's convenient for stable generated values, but protecting that file (permissions, `.gitignore`) is on you; never commit it.
- **Skipping `validate:` on templates for critical services** — a bad render of `nginx.conf` or `sshd_config` can lock you out or take a service down; `validate:` stops the file from being replaced if the check fails.

## FAQ

**Q: What's the difference between `ignore_errors` and `rescue`?**
A: `ignore_errors` hides a failure and continues — nothing responds to it. `rescue` is the recovery path: when something in the `block` fails, the `rescue` tasks run (roll back, notify, clean up), and if they succeed the host continues as *not failed*. Use `ignore_errors` for truly optional steps, `rescue` when you need to react.

**Q: Does `always` run if the block succeeds?**
A: Yes — `always` runs in every case: block succeeded, block failed and was rescued, or block failed with no/failed rescue. It's for cleanup that must happen no matter what (remove temp files, release locks, log the attempt).

**Q: If `rescue` succeeds, does the play report the host as failed?**
A: No. A successful `rescue` clears the failure; the host isn't counted as failed and the play continues. The recap does show a `rescued=` counter so you can still see that a recovery happened.

**Q: When should I use `failed_when` versus `ignore_errors`?**
A: When the tool's exit code doesn't match what "failure" means to you. `grep` returning 1 for "no match" isn't an error — `failed_when: result.rc > 1` encodes that precisely, while `ignore_errors` would also hide *real* errors (rc 2). Prefer `failed_when` for precision.

**Q: What's the difference between the `copy` and `template` modules?**
A: `copy` transfers a file unchanged. `template` renders a Jinja2 file on the control node with the host's variables, then transfers the result — so one source file generates correct per-host configs. Both are idempotent.

**Q: Where do template files go, and what should I name them?**
A: In a role, put them in `templates/` and reference by bare filename (`src: nginx.conf.j2`); in a standalone playbook, a `templates/` directory next to the playbook works the same way. The `.j2` extension is a convention (it helps editors highlight Jinja2), not a requirement.

**Q: What does `default()` do, and when do I need `default(x, true)`?**
A: `default(x)` substitutes `x` only when the variable is **undefined**. `default(x, true)` also substitutes when the value is empty/falsey (`""`, `0`, `[]`, `false`, `None`). Use the `true` form when a variable might be defined but blank.

**Q: What's the difference between `{{ }}` and `{% %}` in a template?**
A: `{{ }}` outputs a value into the rendered file. `{% %}` runs logic (`if`, `for`, `set`) and outputs nothing itself. `{# #}` is a comment that never reaches the output.

**Q: Why do `selectattr`/`map` sometimes show weird `<generator object ...>` output?**
A: They return lazy generators. Append `| list` to materialize them — mandatory when passing the result to `loop`, `length`, or indexing.

**Q: How is `lookup()` different from a variable or `register`?**
A: Variables and registered results exist inside the play. A lookup reaches *outside* — a file, environment variable, template, or generated password on the control node — at the moment it's evaluated. Because lookups are re-evaluated on every use, `lookup('password')` is designed to return a stable stored value rather than a new random one.

**Q: Can I put secrets directly in templates?**
A: You can render a secret *into* a file via a template, but never hardcode the secret *value* inside the template or playbook committed to Git. Pass it in via an environment-sourced lookup or a protected variable file kept out of version control. (Encrypting variable files properly is covered in the next lesson.)

## Practice / Exercise

**Core:**
1. Write a play with a `block` that (a) installs a package, (b) writes a config file, and (c) runs a deliberately failing command (e.g. `ansible.builtin.command: /bin/false`). Add a `rescue` that prints `ansible_failed_task.name` with `debug`, and an `always` that appends a line to a log file. Run it and read the recap — identify the `rescued=` count.
2. Add a block-level `when:` and `become: true` that apply to all tasks inside, and confirm tasks are skipped on hosts that don't match the condition.
3. Register the output of `grep -q somepattern /etc/hostname` and use `failed_when` so that return code 1 (no match) is **not** a failure but return code 2 is. Add `changed_when: false`.
4. Write an `nginx.conf.j2`-style template (or a simpler `app.conf.j2`) containing at least one `{% for %}` loop over a list of dicts and one `{% if %}` conditional. Render it with the `template` module to `/tmp/app.conf` and inspect the output with `cat`.
5. Add `default()` to at least three variables in the template, delete one of them from `group_vars`, and confirm the play still succeeds using the default.
6. Re-run the template task and confirm it reports `ok` (not `changed`); then edit one variable and confirm it reports `changed`. Use `--check --diff` to preview a change before applying it.
7. In a `debug` task, apply `upper`, `lower`, `join`, and `to_json` to sample variables and read the results.
8. Given a list of users with `name` and `active` fields, produce a comma-separated string of only the active users' names using `selectattr`, `map`, and `join`.
9. Use `lookup('file', ...)` to read a local text file into a variable and display it, and `lookup('env', 'HOME')` to display an environment variable — then explain why the *target* host's own files wouldn't work here.

**Stretch:**
1. Use `lookup('password', 'credentials/db.txt length=16')` twice in two separate runs; confirm the same value is returned both times and find the file it created on the control node. Explain why that file must be excluded from Git.
2. Add `validate: nginx -t -c %s` (or `sshd -t -f %s` for an `sshd_config` template) to a template task, then deliberately break the template and confirm the validation rejects it without touching the live file.
3. Wrap a multi-step "deploy" in `block`/`rescue`/`always` where `rescue` uses `ansible_failed_result.msg` to build a notification message, and verify that after a successful rescue the *next* play task still runs on that host.
4. Write a template that renders a YAML file from a variable using `to_nice_yaml`, and compare it with the same data rendered via `to_nice_json`.
5. Use `lookup('template', 'motd.j2')` as the `content:` of a `copy` task and explain when that approach is preferable to using the `template` module directly.

## Further Reading

- [Ansible: Blocks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html)
- [Ansible: Error handling in playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html)
- [Ansible: Templating (Jinja2)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html)
- [Ansible: Filters](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_filters.html)
- [Ansible: Lookup plugins](https://docs.ansible.com/ansible/latest/plugins/lookup.html)
- [Jinja2 Template Designer Documentation](https://jinja.palletsprojects.com/en/latest/templates/)
