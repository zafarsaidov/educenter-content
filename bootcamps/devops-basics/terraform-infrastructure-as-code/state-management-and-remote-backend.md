# Lesson 22: State Management and Remote Backend

**Module:** Terraform – Infrastructure as Code
**Duration:** 120-150 min
**Prerequisites:** Lessons 20-21 (Terraform Basics, Variables/Outputs/Data Sources)

## Learning Objectives

By end of lesson student can:
- Explain what the Terraform state file is, what it tracks, and why it exists
- Use `terraform state` subcommands to inspect, move, and remove entries
- Import existing, unmanaged infrastructure into state
- Configure a remote backend with state locking, and explain why this matters for teams
- Use workspaces to isolate multiple environments, and explain when a separate state file is the better choice instead

## Topics

- Terraform state: `.tfstate` file structure, what it tracks, why it exists
- State commands: `terraform state list`, `show`, `mv`, `rm`; `terraform refresh`
- Remote backends: S3 + DynamoDB locking setup; GCS; Terraform Cloud
- `terraform import`: importing existing infra into state; limitations
- Workspaces: `terraform workspace new/list/select/delete`; isolation use cases vs separate state files

## Concepts

### What is the state file, and why does it exist

Every time Terraform creates, updates, or destroys a resource, it records the result in a **state file** (`terraform.tfstate` by default) — a JSON file mapping each resource block in your configuration to the real-world object it corresponds to, including every attribute Terraform knows about it (IDs, IPs, ARNs, and anything else the provider returned).

State exists because Terraform's declarative model (Lesson 20) needs a reference point: on every `plan`, Terraform compares three things — your configuration (desired state), the state file (what Terraform *believes* exists, last it checked), and (via a `refresh`, usually run automatically as part of `plan`) the real infrastructure's *actual* current state. Without a state file, Terraform would have no reliable way to know that `aws_instance.web` in your config corresponds to a specific, already-existing server rather than one that needs to be created from scratch — it's the mapping between "lines in your `.tf` files" and "specific real-world objects."

### Inspecting and manipulating state directly

| Command | Purpose |
|---|---|
| `terraform state list` | List every resource currently tracked in state |
| `terraform state show <resource>` | Show one tracked resource's full current attributes |
| `terraform state mv <old> <new>` | Rename a resource's address in state, without destroying/recreating it (e.g. after renaming a resource block in `.tf` files) |
| `terraform state rm <resource>` | Remove a resource from state *without* destroying the real infrastructure — Terraform simply forgets about it |
| `terraform refresh` | Reconcile state with real infrastructure's current attributes, without changing any actual resources |

`state mv` matters because simply renaming a `resource "aws_instance" "web"` block to `"webserver"` in your `.tf` files, with no corresponding `state mv`, would make Terraform think the old name's resource no longer exists in config (plan: destroy) and a brand new one needs creating (plan: create) — `state mv` tells Terraform "this is the same real object, just renamed in the config," avoiding an unnecessary and potentially disruptive destroy/recreate cycle.

`state rm` is useful when you want Terraform to stop managing something (e.g. handing a resource off to be managed manually, or by a different configuration) without actually destroying it — critically different from `terraform destroy`, which would actually tear the resource down.

### `terraform import`: adopting existing infrastructure

Sometimes a resource already exists in the real world — created manually through a console, or by a different tool — and you want Terraform to manage it going forward without recreating it. `terraform import <resource_address> <real_id>` adds that existing object into the state file, associated with a resource block you've already written (matching its type) in your configuration.

Import has real limitations worth knowing up front: it only populates *state* — you must still write a `.tf` resource block yourself with arguments that correctly match the real object's actual configuration, or the very next `plan` will show a confusing diff trying to "correct" the object back to whatever your (wrong) config says. In practice, importing usually means: write a best-guess resource block, import, run `plan`, and iteratively adjust the block's arguments until `plan` shows no changes — confirming the config now accurately describes the real object.

### Remote backends and state locking

By default, Terraform stores state as a **local file** on whoever's machine ran `apply` — this breaks down immediately for any team larger than one person: nobody else has the current state, and two people running `apply` around the same time could corrupt it or silently overwrite each other's changes.

A **remote backend** stores the state file centrally instead — common choices include an S3 bucket (AWS), Google Cloud Storage bucket (GCS), or HashiCorp's own Terraform Cloud. A remote backend solves the "shared access" problem; **state locking** solves the "concurrent access" problem on top of that: while one `apply` is in progress, the backend places a lock so a second, concurrent `apply` is blocked (or fails loudly) rather than racing against the first and corrupting state. The classic AWS setup pairs an S3 bucket (storage) with a DynamoDB table (locking), though newer Terraform/provider versions increasingly support S3-native locking without needing DynamoDB separately — worth checking current provider documentation, since this is an area that has evolved.

```hcl
terraform {
    backend "s3" {
        bucket         = "my-terraform-state-bucket"
        key            = "devops-basics/terraform.tfstate"
        region         = "eu-west-1"
        dynamodb_table = "terraform-locks"
        encrypt        = true
    }
}
```

Once a backend is configured, `terraform init` detects it and migrates any existing local state into the remote backend (with confirmation) — from then on, every `plan`/`apply` reads and writes state through the backend instead of a local file.

### Workspaces

A **workspace** lets one configuration manage multiple, independent sets of infrastructure using the *same* `.tf` files, each with its own separate state — `terraform workspace new staging` creates a new workspace; `terraform workspace select staging` switches to it; `terraform.workspace` (a built-in reference) can be used inside the configuration itself (e.g. `"${terraform.workspace}-instance"`) to vary resource naming per workspace.

Workspaces work well for near-identical, lightweight environment variants managed by the exact same config. For environments with meaningfully *different* configuration (different resource counts, different modules entirely, different approval processes) — a common real-world pattern — a **completely separate state file per environment** (one directory per environment, or `-var-file` + distinct backend `key` per environment, as hinted at in the S3 backend example above) is usually the better-understood, less error-prone choice; it's easy to accidentally run `apply` against the wrong workspace since the active workspace isn't always obvious at a glance.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `terraform state list` | List tracked resources | `terraform state list` |
| `terraform state show` | Show one resource's attributes | `terraform state show aws_instance.web` |
| `terraform state mv` | Rename a resource in state | `terraform state mv aws_instance.web aws_instance.webserver` |
| `terraform state rm` | Stop tracking a resource | `terraform state rm aws_instance.old` |
| `terraform refresh` | Reconcile state with real infra | `terraform refresh` |
| `terraform import` | Bring existing infra into state | `terraform import aws_instance.web i-0123456789` |
| `terraform workspace new` | Create a workspace | `terraform workspace new staging` |
| `terraform workspace select` | Switch workspace | `terraform workspace select staging` |
| `terraform workspace list` | List workspaces | `terraform workspace list` |
| `terraform workspace delete` | Remove a workspace | `terraform workspace delete staging` |

## Examples / Walkthrough

```bash
# --- inspecting state ---
terraform state list                                  # every resource Terraform currently tracks
terraform state show aws_instance.web                  # full attributes of one, as Terraform knows them

terraform refresh                                        # reconcile state with reality, no changes applied
```

```bash
# --- state mv: renaming a resource without destroy/recreate ---
# BEFORE: resource "aws_instance" "web" { ... } in main.tf
# you rename it in main.tf to: resource "aws_instance" "webserver" { ... }

terraform plan
# WITHOUT state mv first, this would show:
#   # aws_instance.web will be destroyed
#   # aws_instance.webserver will be created
# — Terraform thinks these are two unrelated resources!

terraform state mv aws_instance.web aws_instance.webserver
terraform plan
# now shows: No changes. Terraform correctly recognizes it as the same object, just renamed.
```

```bash
# --- terraform import: adopting a manually-created resource ---
# a server was created by hand through the AWS console, instance ID i-0a1b2c3d4e5f67890

# step 1: write a best-guess resource block matching it
cat >> main.tf <<'EOF'
resource "aws_instance" "legacy_web" {
    ami           = "ami-0123456789abcdef0"   # best guess, may need correcting
    instance_type = "t3.micro"                 # best guess, may need correcting
}
EOF

# step 2: import the real object into state, associated with that block
terraform import aws_instance.legacy_web i-0a1b2c3d4e5f67890

# step 3: plan, and iterate on the resource block until the diff disappears
terraform plan
# shows a diff wherever your written block doesn't match the real object's actual config —
# adjust instance_type/ami/etc. to match, re-plan, repeat until: No changes.
```

```hcl
# --- configuring a remote backend ---
terraform {
    backend "s3" {
        bucket         = "my-terraform-state-bucket"
        key            = "devops-basics/terraform.tfstate"
        region         = "eu-west-1"
        dynamodb_table = "terraform-locks"
        encrypt        = true
    }
}
```

```bash
terraform init
# Terraform detects the new backend block and offers to migrate existing local state:
# "Do you want to copy existing state to the new backend?"  -> yes

terraform state list                      # confirm everything is still tracked, now via the remote backend
```

```bash
# --- workspaces: one config, multiple isolated environments ---
terraform workspace list                   # shows "default" initially
terraform workspace new staging
terraform workspace new production

terraform workspace select staging
terraform apply -var-file="staging.tfvars"   # state for this apply is isolated to the "staging" workspace

terraform workspace select production
terraform apply -var-file="production.tfvars"  # completely separate state, same .tf files

terraform workspace list                    # default, staging, production — note the current one marked with *
```

## Common Pitfalls

- **Treating `terraform state rm` as a safe "undo"** — it only makes Terraform forget about a resource; the real infrastructure is untouched and keeps running, now completely unmanaged by any Terraform configuration — a common mistake is confusing this with `destroy`, which actually tears the resource down.
- **Running `terraform import` and assuming the job is done** — import only writes to state; if the `.tf` resource block's arguments don't actually match the real object, the next `plan` will show Terraform wanting to "correct" live infrastructure back to your (wrong) config — always iterate with `plan` until the diff is empty before considering an import complete.
- **Not using a remote backend with any team of more than one person** — local state means only the machine that ran `apply` has the current picture; a second person's Terraform run has no idea what the first person already created, leading to drift, conflicts, or duplicate resources.
- **Running `apply` without state locking under concurrent access** — without locking, two simultaneous `apply` runs can corrupt the state file or produce inconsistent infrastructure; a proper remote backend with locking (S3+DynamoDB, GCS's native locking, Terraform Cloud) prevents this by blocking the second run until the first finishes.
- **Losing track of which workspace is currently active** — `terraform workspace select production` followed later by forgetting it's still selected can lead to accidentally running a `dev`-intended `apply` against production's state. Always confirm with `terraform workspace list` (checking the `*` marker) before any apply in a multi-workspace setup, or prefer fully separate state files/directories for anything high-stakes.
- **Committing a local `.tfstate` file to Git** — beyond the `.gitignore` reasoning from Lesson 20, a state file can contain sensitive resource attributes in plain text; this is one of the strongest practical arguments for using a remote backend (with appropriate access controls) rather than ever having state touch version control at all.

## FAQ

**Q: Why can't Terraform just inspect real infrastructure directly on every `plan`, instead of needing a separate state file?**
A: For many resources it technically could re-discover *some* information, but state also tracks metadata that has no real-world equivalent to look up — which resource block in your configuration corresponds to which real object, dependency ordering information, and values from resources (like a randomly generated password) that aren't queryable from the provider's API after the fact. State is Terraform's own authoritative bridge between config and reality, not just a cache of what could be re-fetched.

**Q: What's the real difference between `terraform state rm` and `terraform destroy`?**
A: `state rm` only edits the state file — Terraform forgets the resource exists, but the real infrastructure is completely untouched and keeps running (now unmanaged). `destroy` actually calls the provider's API to tear the real resource down. They solve very different problems and should never be confused.

**Q: Is `terraform import` ever fully automatic?**
A: Not with the classic `import` command covered here — it always requires you to have (or iteratively write) a matching resource block. Newer versions of Terraform also support an `import` block in configuration that can generate a starting resource block automatically via `terraform plan -generate-config-out`, which reduces the guesswork — worth exploring further once comfortable with the manual process this lesson covers.

**Q: When should I use workspaces vs. entirely separate state files/directories per environment?**
A: Workspaces fit when environments are genuinely near-identical (same resources, same modules, differing mainly by variable values and scale). Once environments diverge meaningfully in actual resources/modules used, or need different access controls/approval processes, separate state (often separate directories, each with its own backend `key`) is clearer and harder to apply against the wrong target by mistake.

**Q: Does a remote backend protect against a bad `terraform apply` destroying something it shouldn't?**
A: No — a remote backend solves *access and concurrency* (shared visibility, locking against simultaneous runs), not *correctness* of what a given apply does. Reviewing `plan` output carefully (Lesson 20) and tools like `lifecycle { prevent_destroy = true }` are the actual safeguards against an unwanted destructive change.

## Practice / Exercise

**Core:**
1. After applying a configuration from an earlier lesson, run `terraform state list` and `terraform state show` on one resource; identify at least 3 attributes you didn't explicitly set yourself that Terraform recorded.
2. Rename one resource's local name in your `.tf` file, run `plan` without a `state mv` first and observe the destroy+create it proposes, then undo, correctly run `terraform state mv`, and confirm `plan` now shows no changes.
3. Practice `terraform state rm` on a disposable test resource, confirm (via your cloud console, or the `local` provider's created file) that the real resource still exists, then re-`import` it back into state.
4. Set up a remote backend (an S3 bucket if you have AWS access, or research/describe the equivalent steps for GCS/Terraform Cloud if you don't), run `terraform init`, and confirm state successfully migrates.
5. Create two workspaces (`dev`/`staging`), apply the same configuration with different `-var-file`s in each, and confirm with `terraform state list` in each workspace that they're tracking entirely separate resources.

**Stretch:**
1. Manually create a simple resource outside of Terraform (through a console, or using the `local_file` provider's underlying file directly), then write a matching resource block and `import` it, iterating until `plan` shows no changes.
2. Research and write up (in your own words, no need to actually configure it) how DynamoDB-based locking works mechanically — what happens if an `apply` crashes partway through while holding the lock, and how that's recovered from.
3. Compare, for a hypothetical 3-environment (dev/staging/production) project: the workspace approach vs. the separate-directories-per-environment approach, listing at least 2 concrete advantages and 2 concrete risks of each.

## Further Reading

- [Terraform: State](https://developer.hashicorp.com/terraform/language/state)
- [Terraform: Backends](https://developer.hashicorp.com/terraform/language/settings/backends/configuration)
- [Terraform: Import](https://developer.hashicorp.com/terraform/cli/import)
- [Terraform: Workspaces](https://developer.hashicorp.com/terraform/language/state/workspaces)
