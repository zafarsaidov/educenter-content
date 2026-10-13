# Lesson 27: Conditionals and Loops in Ansible

**Module:** Ansible – Configuration Management
**Duration:** 120-150 min
**Prerequisites:** Lessons 24-26 (Introduction to Ansible & YAML, Inventory & Folder Structure, Roles) — playbooks, modules, inventory groups, and roles/handlers are assumed

## Learning Objectives

By end of lesson student can:
- Control whether a task runs using `when`, including simple comparisons, variable checks, and combined `and`/`or` conditions
- Use Ansible facts such as `ansible_facts['os_family']` to branch playbook behavior per host
- Repeat a task over a list with `loop`, over a dictionary with `dict2items`, and recognize legacy `with_items` syntax
- Capture a task's output with `register` and read `.stdout`, `.stderr`, `.rc`, and `.results`
- Create dynamic variables with `set_fact`, including from registered output
- Use the `debug` module (`var`, `msg`) to inspect values and troubleshoot a playbook

## Topics

- `when`: simple conditions, variable checks, combining conditions with `and`/`or`
- Loops: `loop` with a list, `loop` with `dict2items`, `with_items` (legacy)
- `register`: capturing task output; accessing `.stdout`, `.rc`, `.stderr`, `.results`
- `set_fact`: creating dynamic variables; combining with `register` output
- `debug` module: `var`, `msg`; practical use in troubleshooting playbooks

## Concepts

### Why conditionals and loops

A playbook that runs identical tasks on every host works for a uniform fleet, but real inventories are rarely uniform: some hosts run Ubuntu and others Rocky Linux, some are web servers and others databases, some already have a service installed and others don't. Without conditional logic, you'd write a separate playbook per case. Without loops, installing 10 packages means 10 near-identical tasks. `when` and `loop` are Ansible's answer to the same two needs you already met in Bash (Lessons 16-17: `if` and `for`) — but expressed declaratively in YAML, per task, rather than as procedural control flow around commands.

### Facts: the data `when` usually checks

Before running tasks, Ansible by default runs an implicit **fact gathering** step (the `setup` module) on every host, collecting details about it: OS family, distribution, version, CPU count, memory, IP addresses, and more. These are available as `ansible_facts`, accessed with bracket syntax:

| Fact | Example value | Meaning |
|---|---|---|
| `ansible_facts['os_family']` | `Debian`, `RedHat` | Broad OS family — the most common branching key (Ubuntu and Debian are both `Debian`; Rocky, CentOS, Fedora are `RedHat`) |
| `ansible_facts['distribution']` | `Ubuntu`, `Rocky` | Specific distribution name |
| `ansible_facts['distribution_major_version']` | `22`, `9` | Major version |
| `ansible_facts['memtotal_mb']` | `3921` | Total RAM in MB (an integer) |
| `ansible_facts['processor_vcpus']` | `2` | Number of virtual CPUs |
| `ansible_facts['hostname']` | `web-01` | Host's short name |

You can see every fact for a host with an ad-hoc command: `ansible web-01 -m setup` (Lesson 24's ad-hoc syntax), optionally narrowed with `-a "filter=ansible_distribution*"`.

### `when`: conditional task execution

`when` is a task keyword whose value is a **Jinja2 expression evaluated to true or false**. If it's true for a given host, the task runs on that host; otherwise the task is **skipped** for that host (reported as `skipping`, not as a failure). Crucially, `when` is evaluated **separately per host** — in one play, a task can run on the Ubuntu hosts and be skipped on the Rocky ones.

Notice that `when` does *not* use `{{ }}` around variables — it's already treated as a Jinja2 expression, so you write `when: ansible_facts['os_family'] == "Debian"`, not `when: "{{ ... }}"`. (Using `{{ }}` inside `when` is a common mistake and triggers a warning.)

```yaml
- name: Install nginx on Debian-family hosts
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_facts['os_family'] == "Debian"
```

**Comparison operators** work as in most languages: `==`, `!=`, `>`, `<`, `>=`, `<=`. Strings must be quoted; numbers and booleans are not.

**Variable checks** use Jinja2 *tests* — a different construct from filters (tests answer a yes/no question about a value):

| Expression | True when |
|---|---|
| `my_var is defined` | The variable exists |
| `my_var is not defined` | The variable does not exist |
| `my_var` (bare, a boolean var) | The variable is `true` |
| `not my_var` | The variable is `false` |
| `"nginx" in my_list` | The value is an element of the list |
| `"error" in some_string` | The substring appears in the string |

**Combining conditions** uses `and`, `or`, `not`, and parentheses:

```yaml
when: ansible_facts['os_family'] == "Debian" and ansible_facts['distribution_major_version'] == "22"
```

For several conditions that must **all** be true, Ansible also accepts a YAML list — each item is combined with an implicit `and`, which reads far more cleanly than one long line:

```yaml
when:
  - ansible_facts['os_family'] == "Debian"
  - ansible_facts['memtotal_mb'] >= 2048
  - install_monitoring
```

There is no list form for `or` — use the explicit keyword: `when: env == "dev" or env == "staging"`.

### Loops: `loop`

`loop` repeats a single task once per item in a list. Inside the task, the current item is available as the variable **`item`**:

```yaml
- name: Install several packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - git
    - curl
```

The loop can also reference a variable holding a list: `loop: "{{ packages }}"`. (Here the `{{ }}` *is* required, because the whole value is otherwise a plain string — a good contrast with `when`.)

Loop items don't have to be plain strings — a list of **dictionaries** is very common, and you read their keys with dot or bracket notation:

```yaml
- name: Create users with specific shells
  ansible.builtin.user:
    name: "{{ item.name }}"
    shell: "{{ item.shell }}"
  loop:
    - { name: alice, shell: /bin/bash }
    - { name: bob,   shell: /bin/zsh }
```

### Looping over a dictionary: `dict2items`

A `loop` needs a *list*. If your data is a **dictionary** (a map of keys to values), the `dict2items` filter converts it into a list of `{key: ..., value: ...}` entries that `loop` can iterate:

```yaml
vars:
  service_ports:
    http: 80
    https: 443
    ssh: 22

tasks:
  - name: Show each service and its port
    ansible.builtin.debug:
      msg: "{{ item.key }} listens on {{ item.value }}"
    loop: "{{ service_ports | dict2items }}"
```

`dict2items` is the one filter this lesson uses, because it is the standard, explicit bridge between maps and loops. Each iteration's `item` has two attributes, `item.key` and `item.value`.

### `with_items` and friends: legacy loop syntax

Before `loop` existed (Ansible 2.5), loops were written as `with_<something>`: `with_items`, `with_dict`, `with_sequence`, `with_fileglob`, and so on. You'll still meet `with_items` constantly in older playbooks, roles, and blog posts:

```yaml
- name: Install packages (legacy style)
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  with_items:
    - nginx
    - git
```

`with_items` still works and behaves like `loop` for a simple list (one subtle difference: `with_items` silently flattens nested lists one level, while `loop` does not). For new code, prefer `loop`; for reading old code, recognize `with_items` as "the same idea, older spelling."

### `register`: capturing a task's result

Every task produces a result structure. `register` saves it into a variable you name, so later tasks can inspect it:

```yaml
- name: Check which user Ansible runs as
  ansible.builtin.command: whoami
  register: whoami_result
```

The registered variable is a dictionary. The keys you'll use most often:

| Key | Holds |
|---|---|
| `.stdout` | Standard output of a command-type task, as one string |
| `.stdout_lines` | The same output, already split into a list of lines |
| `.stderr` | Standard error output as a string |
| `.rc` | The command's return code (exit status) — `0` means success |
| `.changed` | Whether Ansible considered the task to have changed something |
| `.failed` | Whether the task failed |

Which keys exist depends on the module: `command`/`shell` give `stdout`/`stderr`/`rc`; `stat` gives `.stat.exists`; `package` modules give `changed`. Use `debug` (below) to see exactly what a given module returns.

When a task uses `loop`, the registered variable has a different shape: it contains a **`.results`** list with one entry per iteration, each entry being a full result dictionary for that item (including the original `item`):

```yaml
- name: Run a command per item
  ansible.builtin.command: "echo {{ item }}"
  loop: [alpha, beta, gamma]
  register: echo_results

# echo_results.results[0].stdout == "alpha"
# echo_results.results[1].stdout == "beta"
```

### `set_fact`: creating variables during the play

`set_fact` defines a new variable at runtime, scoped to the current host for the rest of the play (and available to later plays in the same run). Its main use is turning raw task output into a clean, named, reusable value, or computing a value that depends on facts discovered during the run:

```yaml
- name: Remember whether this host is a "large" server
  ansible.builtin.set_fact:
    is_large_server: "{{ ansible_facts['memtotal_mb'] >= 8192 }}"
```

Combined with `register`, it captures a result in a form later conditions can use:

```yaml
- name: Check if the app config exists
  ansible.builtin.stat:
    path: /etc/myapp/config.yml
  register: config_stat

- name: Record the result as a simple flag
  ansible.builtin.set_fact:
    config_present: "{{ config_stat.stat.exists }}"

- name: Only runs when the config is already there
  ansible.builtin.debug:
    msg: "Config found, skipping first-time setup"
  when: config_present
```

`set_fact` values live only for the run (they aren't written to disk) and are **per host** — each host in the inventory gets its own independent copy.

### The `debug` module

`debug` prints information during a run and changes nothing on the target. It is the Ansible equivalent of `echo` / `print` while troubleshooting. It takes one of two parameters:

- `msg:` — prints a message, which can contain `{{ }}` interpolation: `msg: "Host {{ inventory_hostname }} has {{ ansible_facts['memtotal_mb'] }} MB"`
- `var:` — prints a variable by name (no `{{ }}` — pass the bare name), displaying its full structure, which is invaluable for exploring what a registered result actually contains: `var: whoami_result`

Use `var:` to *discover* the structure, then use `msg:` (or `when`) to consume the specific key you need. Running a playbook with `-v`, `-vv`, or `-vvv` (Lesson 24) adds even more detail, but a targeted `debug` task is usually clearer.

## Commands / Syntax Reference

| Syntax | Purpose | Example |
|---|---|---|
| `when: expr` | Run the task only if the expression is true (per host) | `when: ansible_facts['os_family'] == "Debian"` |
| `when:` (list) | All listed conditions must be true (implicit `and`) | see walkthrough |
| `and` / `or` / `not` | Combine conditions | `when: a == 1 or b == 2` |
| `is defined` / `is not defined` | Test whether a variable exists | `when: my_var is defined` |
| `in` | Membership test for lists and substrings | `when: "web" in group_names` |
| `loop: [list]` | Repeat a task for each list item (`item`) | `loop: [nginx, git]` |
| `loop: "{{ dict \| dict2items }}"` | Loop over a dictionary (`item.key`, `item.value`) | see walkthrough |
| `with_items:` | Legacy list loop | `with_items: [nginx, git]` |
| `register: name` | Save a task's result to a variable | `register: result` |
| `result.stdout` / `.stdout_lines` | Command output (string / list of lines) | `result.stdout` |
| `result.rc` | Return code of a command | `when: result.rc == 0` |
| `result.results` | Per-item results when `register` is used with a loop | `result.results[0].stdout` |
| `set_fact:` | Define a variable at runtime | `set_fact: role_name="web"` |
| `debug: var=` | Print a variable's full structure | `debug: var=result` |
| `debug: msg=` | Print a message with interpolation | `debug: msg="Hello {{ name }}"` |
| `ansible <host> -m setup` | Show all gathered facts for a host | `ansible web-01 -m setup` |

## Examples / Walkthrough

```yaml
# --- conditionals-demo.yml: when with facts ---
- name: Conditionals demo
  hosts: all
  become: true

  vars:
    install_monitoring: true
    env_name: staging

  tasks:
    - name: Show the facts we will branch on
      ansible.builtin.debug:
        msg: >-
          {{ inventory_hostname }} is {{ ansible_facts['distribution'] }}
          {{ ansible_facts['distribution_major_version'] }}
          ({{ ansible_facts['os_family'] }}) with {{ ansible_facts['memtotal_mb'] }} MB RAM

    - name: Install nginx on Debian-family hosts
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
      when: ansible_facts['os_family'] == "Debian"

    - name: Install nginx on RedHat-family hosts
      ansible.builtin.dnf:
        name: nginx
        state: present
      when: ansible_facts['os_family'] == "RedHat"

    - name: Variable check - only if install_monitoring is true
      ansible.builtin.debug:
        msg: "Monitoring agent would be installed here"
      when: install_monitoring

    - name: Combined conditions as a list (all must be true)
      ansible.builtin.debug:
        msg: "Big Debian host in a non-dev environment"
      when:
        - ansible_facts['os_family'] == "Debian"
        - ansible_facts['memtotal_mb'] >= 2048
        - env_name != "dev"

    - name: or condition (explicit keyword)
      ansible.builtin.debug:
        msg: "This is a pre-production environment"
      when: env_name == "dev" or env_name == "staging"

    - name: Only if an optional variable was supplied
      ansible.builtin.debug:
        msg: "Custom banner is {{ custom_banner }}"
      when: custom_banner is defined

    - name: Group membership test
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} is in the webservers group"
      when: "'webservers' in group_names"
```

```yaml
# --- loops-demo.yml: loop, dict2items, with_items ---
- name: Loops demo
  hosts: all
  become: true

  vars:
    packages:
      - git
      - curl
      - htop
    service_ports:
      http: 80
      https: 443
      ssh: 22
    app_users:
      - { name: alice, shell: /bin/bash }
      - { name: bob,   shell: /bin/sh }

  tasks:
    - name: Install packages from a list variable
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages }}"
      when: ansible_facts['os_family'] == "Debian"

    - name: Create users from a list of dictionaries
      ansible.builtin.user:
        name: "{{ item.name }}"
        shell: "{{ item.shell }}"
        state: present
      loop: "{{ app_users }}"

    - name: Loop over a dictionary with dict2items
      ansible.builtin.debug:
        msg: "{{ item.key }} -> port {{ item.value }}"
      loop: "{{ service_ports | dict2items }}"

    - name: Legacy syntax - you will see this in older roles
      ansible.builtin.debug:
        msg: "legacy item: {{ item }}"
      with_items:
        - one
        - two

    - name: Loop and condition together (when is evaluated per item)
      ansible.builtin.debug:
        msg: "{{ item.name }} uses bash"
      loop: "{{ app_users }}"
      when: item.shell == "/bin/bash"
```

```yaml
# --- register-demo.yml: register, set_fact, debug ---
- name: register and set_fact demo
  hosts: all

  tasks:
    - name: Run a command and register the result
      ansible.builtin.command: whoami
      register: whoami_result

    - name: Discover the full structure of the registered variable
      ansible.builtin.debug:
        var: whoami_result          # no {{ }} with var:

    - name: Use specific keys
      ansible.builtin.debug:
        msg: "Ansible ran as {{ whoami_result.stdout }} (rc={{ whoami_result.rc }})"

    - name: Act on the return code
      ansible.builtin.debug:
        msg: "Command succeeded"
      when: whoami_result.rc == 0

    - name: Check whether a file exists using stat
      ansible.builtin.stat:
        path: /etc/hostname
      register: hostname_file

    - name: Turn the result into a simple, reusable flag
      ansible.builtin.set_fact:
        hostname_file_present: "{{ hostname_file.stat.exists }}"

    - name: Use the flag later in the play
      ansible.builtin.debug:
        msg: "/etc/hostname is present"
      when: hostname_file_present

    - name: Compute a value with set_fact from facts
      ansible.builtin.set_fact:
        server_size: "{{ 'large' if ansible_facts['memtotal_mb'] >= 8192 else 'small' }}"

    - name: Show the computed fact
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} is a {{ server_size }} server"

    - name: register combined with loop produces .results
      ansible.builtin.command: "echo {{ item }}"
      loop:
        - alpha
        - beta
      register: echo_results

    - name: Inspect the per-item results
      ansible.builtin.debug:
        msg: "item={{ item.item }} stdout={{ item.stdout }} rc={{ item.rc }}"
      loop: "{{ echo_results.results }}"
```

```bash
# --- running the demos ---
ansible-playbook -i inventory.ini conditionals-demo.yml
ansible-playbook -i inventory.ini loops-demo.yml --check        # dry run (Lesson 24)
ansible-playbook -i inventory.ini register-demo.yml

# exploring facts to decide what to put in a `when`
ansible web-01 -i inventory.ini -m setup | less
ansible web-01 -i inventory.ini -m setup -a "filter=ansible_os_family"
ansible web-01 -i inventory.ini -m setup -a "filter=ansible_memtotal_mb"

# need more detail while troubleshooting a task result
ansible-playbook -i inventory.ini register-demo.yml -v
```

## Common Pitfalls

- **Wrapping the `when` expression in `{{ }}`** — `when: "{{ my_var }}"` is wrong; `when` is already evaluated as a Jinja2 expression, so write `when: my_var`. The templated form triggers a warning and can behave unexpectedly with non-boolean values.
- **Mistaking a skipped task for a failure (or the reverse)** — `skipping` means the `when` condition was false for that host; it's normal and expected. A task that is skipped on every host usually points to a typo in a fact name or a wrong comparison value (e.g. comparing `os_family` to `"debian"` in lowercase when the fact value is `"Debian"` — string comparison is case-sensitive).
- **Quoting problems with YAML and `when`** — a `when` value starting with a quote or containing `: ` (colon-space) can break YAML parsing. If a condition contains tricky characters, wrap the whole expression in quotes (`when: "'webservers' in group_names"`), keeping the inner string literals in the other quote type.
- **Using `loop` on a dictionary** — `loop` requires a list. Passing a dict directly fails or iterates unexpectedly; convert it with `| dict2items` first, then use `item.key` / `item.value`.
- **Forgetting `item` is the loop variable (and clobbering it)** — inside a looped task the current element is always `item`. If a loop task includes another construct that also uses `item` (nested loops via included tasks, for example), the names collide. Keep loops flat while learning, and know that `loop_control: loop_var:` exists to rename it when needed.
- **Registering from a loop and reading `.stdout` directly** — with `loop`, the registered variable has no top-level `.stdout`; the outputs live in `.results[N].stdout`. Use `debug: var=` first to see the real shape rather than guessing.
- **Treating `set_fact` values as permanent** — they exist only for the duration of the run, per host, and are gone the next time the playbook starts. For data that must persist, put it in inventory variables (Lesson 25's `host_vars`/`group_vars`) or on the target itself.
- **Comparing numbers as strings** — `ansible_facts['memtotal_mb'] >= 2048` works because the fact is an integer. But values captured from command output (`.stdout`) are *strings*; comparing `"10" >= "9"` as strings gives the wrong answer. Prefer comparing return codes (`.rc == 0`), or capture facts that are already numeric.
- **Leaving noisy `debug` tasks in production playbooks** — `debug` is a troubleshooting tool. Remove it, or guard it, once the playbook works, so real runs stay readable.

## FAQ

**Q: Why doesn't `when` need `{{ }}` but `loop: "{{ packages }}"` does?**
A: `when` is parsed as a raw Jinja2 expression, so variable names are already resolved inside it. A `loop:` value, like most other task keywords and module parameters, is a plain YAML string unless you template it — so a variable reference there must be written as `"{{ packages }}"`, and quoted so YAML doesn't misread the braces.

**Q: What's the difference between `loop` and `with_items`?**
A: They do the same job for a simple list. `with_items` is the older syntax from before Ansible 2.5 and also auto-flattens nested lists one level; `loop` is the modern, unified keyword and doesn't flatten. Use `loop` in new code; recognize `with_items` when reading older playbooks and roles.

**Q: How do I see what a registered variable contains?**
A: Add a task right after it: `debug: var=your_register_name`. It prints the full dictionary — `stdout`, `stderr`, `rc`, `changed`, and any module-specific keys. Do this whenever you're unsure which key to use rather than guessing.

**Q: Does `when` run once per playbook or once per host?**
A: Once per host (and once per item, if the task also has a `loop`). That's why the same task can run on Ubuntu hosts and skip on Rocky hosts within a single play — each host evaluates the condition against its own facts and variables.

**Q: What's the difference between `register` and `set_fact`?**
A: `register` saves a *task's whole result structure* under a name, exactly as the module returned it. `set_fact` creates a variable whose value *you* define — usually a simplified or computed value (often derived from a registered result). Typical pattern: `register` to capture raw output, `set_fact` to distill it into a clean flag or value.

**Q: Can I use `when` with a `loop`?**
A: Yes, and the condition is evaluated separately for every item, with `item` available inside it — so you can filter which items the task actually acts on (e.g. `when: item.shell == "/bin/bash"`). Items that fail the condition are reported as skipped individually.

**Q: Why is a task showing `ok` but my `command` task always says `changed`?**
A: `command` and `shell` can't know whether running a command modified anything, so they report `changed` every time. Idempotent modules (`apt`, `user`, `file`, `stat`, etc.) report accurately. For tasks that only read information, prefer a read-only module such as `stat`, and keep this in mind when you see `changed` on a command task. (Overriding that reporting is a topic for the next lesson.)

**Q: When should I use a fact vs. my own variable in `when`?**
A: Use facts for things that are properties of the host itself and discovered automatically (OS, memory, CPU). Use your own variables (from inventory, `vars`, or `-e`) for decisions that are *yours* — environment name, feature switches, which role a host plays.

## Practice / Exercise

**Core:**
1. Run `ansible <host> -m setup -a "filter=ansible_os_family"` against two or three hosts, and record the `os_family`, `distribution`, `distribution_major_version`, and `memtotal_mb` values for each.
2. Write a playbook with two tasks that install the same package using `apt` on Debian-family hosts and `dnf` on RedHat-family hosts, each guarded with a `when` on `ansible_facts['os_family']`. Run it and confirm each host skips the task for the other family.
3. Add a task guarded by a **list** of three conditions (OS family, minimum memory, and a boolean variable you define) and confirm it only runs on hosts matching all three. Then change one condition so it's skipped, and confirm the `skipping` output.
4. Add a task using `or` (e.g. environment is `dev` or `staging`), and a task using `is defined` against a variable you deliberately do not set — confirm it skips cleanly instead of erroring.
5. Install three packages with a single task using `loop`, then rewrite the same task using the legacy `with_items` and confirm identical behavior.
6. Create two users from a list of dictionaries (`name`, `shell`) using `loop` and `item.name` / `item.shell`.
7. Define a dictionary of service names to ports and iterate it using `dict2items`, printing `item.key` and `item.value` via `debug: msg=`.
8. Register the output of `command: whoami`, print the whole structure with `debug: var=`, then print only `.stdout` and `.rc` with `debug: msg=`.
9. Use `stat` on a file, register the result, convert `.stat.exists` into a variable with `set_fact`, and run a later task only when that variable is true.
10. Register a looped `command` task and print each item's `stdout` by looping over `.results`.

**Stretch:**
1. Compute a `server_size` fact (`large`/`small`) from `memtotal_mb` using `set_fact` and a conditional expression, then branch two later tasks on it.
2. Write a play that combines `loop` and `when` so that only users whose shell is `/bin/bash` get an extra task, and explain what the output shows for the skipped items.
3. Deliberately write a `when` that compares `os_family` to the lowercase string `"debian"`, observe every host skipping, and explain why (case sensitivity) before fixing it.
4. Capture a command's numeric output in `.stdout`, then demonstrate (with `debug`) why comparing it directly as a string against a number gives surprising results, and describe in your own words a safer approach using return codes or numeric facts.
5. Refactor one of your `when`/`loop` tasks into a role's `tasks/main.yml` (Lesson 26), keeping the list of packages in the role's `defaults/main.yml`, and run it from a playbook.

## Further Reading

- [Ansible docs: Conditionals](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html)
- [Ansible docs: Loops](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html)
- [Ansible docs: Discovering variables — facts and magic variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html)
- [Ansible docs: `ansible.builtin.debug` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)
- [Ansible docs: `ansible.builtin.set_fact` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/set_fact_module.html)
