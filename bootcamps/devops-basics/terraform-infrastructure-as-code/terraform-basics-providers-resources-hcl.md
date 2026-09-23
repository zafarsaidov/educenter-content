# Lesson 20: Terraform Basics – Providers, Resources, HCL

**Module:** Terraform – Infrastructure as Code
**Duration:** 120-150 min
**Prerequisites:** Linux Basics, Git Version Control, Scripting modules

## Learning Objectives

By end of lesson student can:
- Explain infrastructure as code, and the difference between imperative and declarative provisioning
- Read and write basic HCL: blocks, arguments, expressions, and Terraform's core types
- Configure a provider and write a resource block
- Explain the most common meta-arguments (`depends_on`, `lifecycle`)
- Run the full core Terraform CLI workflow: `init`, `validate`, `fmt`, `plan`, `apply`, `destroy`

## Topics

- IaC concepts: imperative vs declarative, idempotency, desired state
- HCL syntax: blocks, arguments, expressions, types (`string`, `number`, `bool`, `list`, `map`)
- Provider configuration: `required_providers`, version constraints, credentials setup
- Resource block: type, name, arguments, meta-arguments (`depends_on`, `lifecycle`)
- CLI workflow: `terraform init`, `validate`, `fmt`, `plan`, `apply`, `destroy`; `.terraform` directory

## Concepts

### Infrastructure as Code: imperative vs declarative

**Infrastructure as Code (IaC)** means defining infrastructure (servers, networks, DNS records, cloud resources) in text files instead of clicking through a web console or running one-off commands — the files become the source of truth, can be version-controlled (Lesson 14), reviewed, and reproduced.

Two fundamentally different styles exist:

- **Imperative** — a script of *steps* to reach a result ("create a server, then attach a disk, then configure networking"). A Bash script calling cloud CLI commands is imperative: it describes *how* to get there.
- **Declarative** — a description of the *desired end state* ("I want exactly one server, with this disk, on this network"). The tool figures out *how* to get there itself, including what to change if the current state doesn't match.

Terraform is declarative. You describe what infrastructure *should* exist; Terraform compares that against what currently exists and computes the difference.

### Idempotency and desired state

**Idempotency** means running the same operation multiple times produces the same result as running it once — no matter how many times you `terraform apply` the same configuration, the outcome is the same target state, not duplicated resources. This is a direct consequence of the declarative model: Terraform isn't replaying a list of creation steps each time, it's continuously reconciling *actual* infrastructure toward the *desired state* described in your files. If nothing changed in the config and nothing drifted in reality, `apply` simply reports "no changes."

### HCL: blocks, arguments, expressions

Terraform's configuration language is **HCL** (HashiCorp Configuration Language). Its basic building unit is the **block**:

```hcl
block_type "label_one" "label_two" {
    argument_name = value
}
```

- `block_type` — what kind of thing this is (`resource`, `provider`, `variable`, etc.)
- Labels — zero, one, or two quoted strings identifying this specific block (a `resource` block always has two: its type and its local name)
- Inside the `{}` — **arguments**, each an `name = value` assignment

Values can be literals, or **expressions** — references to other values, computed at plan/apply time: `other_resource.attribute`, string interpolation `"${var.name}-suffix"` (though inside plain strings this can often be simplified, e.g. `"${var.env}"` is equivalent to just `var.env` when it's the entire string value).

Core HCL types:

| Type | Example |
|---|---|
| `string` | `"hello"` |
| `number` | `42`, `3.14` |
| `bool` | `true`, `false` |
| `list` | `["a", "b", "c"]` |
| `map` | `{ key = "value" }` |

### Providers

A **provider** is a plugin that lets Terraform talk to a specific platform's API — AWS, GCP, Azure, Hetzner, even non-cloud things like GitHub or Kubernetes. Every resource block belongs to a provider, which is what actually translates "create this resource" into real API calls.

```hcl
terraform {
    required_providers {
        aws = {
            source  = "hashicorp/aws"
            version = "~> 5.0"
        }
    }
}

provider "aws" {
    region = "eu-west-1"
}
```

`required_providers` declares which provider(s) this configuration needs and pins a **version constraint** (`~> 5.0` means "any 5.x version, but not 6.0") — pinning matters because a provider's behavior can change between major versions, and an unpinned config could silently start behaving differently after a routine `terraform init` months later. Credentials are typically supplied outside the config file itself (environment variables, a credentials file, or a cloud instance's attached identity) rather than hardcoded — never commit real credentials into a `.tf` file that goes into Git.

### Resource blocks

A **resource** block declares one piece of infrastructure to manage:

```hcl
resource "aws_instance" "web" {
    ami           = "ami-0123456789"
    instance_type = "t3.micro"
}
```

- `"aws_instance"` — the resource *type*, defined by the provider
- `"web"` — the resource's *local name*, used to reference it elsewhere in this configuration (e.g. `aws_instance.web.id`) — this name exists only within your Terraform files, it's not the actual cloud resource's name/tag
- The arguments inside — specific to what `aws_instance` accepts, documented by the provider

### Meta-arguments: `depends_on` and `lifecycle`

Certain arguments work the same way across *any* resource type, regardless of provider — these are **meta-arguments**:

- `depends_on = [other_resource]` — explicitly forces an ordering dependency. Terraform normally infers dependencies automatically from references between resources (if resource B's config references resource A's attribute, Terraform already knows to create A first) — `depends_on` is only needed for the rarer case where a dependency exists but isn't visible through any attribute reference.
- `lifecycle { ... }` — a nested block controlling how Terraform manages changes to this specific resource:

| Lifecycle setting | Effect |
|---|---|
| `create_before_destroy = true` | Create the replacement resource before destroying the old one (default is destroy-then-create) — avoids downtime for resources that can't have duplicates existing simultaneously by name, when order matters |
| `prevent_destroy = true` | Refuses any plan that would destroy this resource — a safety rail for critical, hard-to-recreate infrastructure |
| `ignore_changes = [attribute]` | Tells Terraform to stop caring if a specific attribute drifts from what's in the config (e.g. a tag another system manages) |

### The CLI workflow

| Command | What it does |
|---|---|
| `terraform init` | Downloads required providers/modules, sets up the `.terraform/` working directory; run this first, and again whenever providers/modules change |
| `terraform validate` | Checks the configuration is syntactically valid and internally consistent — no API calls, purely local |
| `terraform fmt` | Auto-formats `.tf` files to the canonical style (indentation, alignment) |
| `terraform plan` | Compares desired state (your config) against current real state, and shows exactly what would change — nothing is actually applied yet |
| `terraform apply` | Executes the plan (after confirmation, or with `-auto-approve`) — this is the step that actually creates/modifies/destroys real infrastructure |
| `terraform destroy` | Tears down every resource this configuration manages |

The `.terraform/` directory (created by `init`) holds downloaded provider plugins and module code — it's local working state, not meant to be committed to Git (add it to `.gitignore`, Lesson 14).

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `terraform init` | Initialize working directory | `terraform init` |
| `terraform validate` | Check config validity | `terraform validate` |
| `terraform fmt` | Auto-format `.tf` files | `terraform fmt -recursive` |
| `terraform plan` | Preview changes | `terraform plan` |
| `terraform apply` | Apply changes | `terraform apply` |
| `terraform destroy` | Tear down managed resources | `terraform destroy` |
| `terraform show` | Show current state in human-readable form | `terraform show` |
| `terraform version` | Show Terraform + provider versions | `terraform version` |

## Examples / Walkthrough

```hcl
# --- main.tf: a minimal, complete configuration ---
terraform {
    required_providers {
        aws = {
            source  = "hashicorp/aws"
            version = "~> 5.0"
        }
    }
}

provider "aws" {
    region = "eu-west-1"
}

resource "aws_instance" "web" {
    ami           = "ami-0123456789abcdef0"
    instance_type = "t3.micro"

    tags = {
        Name        = "devops-basics-web"
        Environment = "training"
    }
}

resource "aws_instance" "web_replica" {
    ami           = "ami-0123456789abcdef0"
    instance_type = "t3.micro"

    depends_on = [aws_instance.web]      # explicit ordering, even though nothing references web's attributes here

    lifecycle {
        create_before_destroy = true
        prevent_destroy        = false
    }

    tags = {
        Name = "devops-basics-web-replica"
    }
}
```

```bash
# --- the CLI workflow, in order ---
terraform init                       # downloads the aws provider plugin into .terraform/
terraform fmt                        # normalize formatting before committing
terraform validate                   # confirm the config is syntactically/internally valid

terraform plan                       # shows: 2 to add, 0 to change, 0 to destroy — REVIEW before applying
terraform apply                      # prompts "yes" to confirm, then actually creates the instances

terraform show                       # inspect current real state in readable form

# make a change, e.g. edit instance_type to "t3.small" in main.tf, then:
terraform plan                        # shows exactly what will change: instance_type: "t3.micro" -> "t3.small"
terraform apply

terraform destroy                     # tears down everything this config manages, with confirmation prompt
```

```bash
# --- .gitignore for a Terraform project ---
cat > .gitignore <<'EOF'
.terraform/
*.tfstate
*.tfstate.backup
.terraform.lock.hcl
EOF
# (note: .terraform.lock.hcl is sometimes committed deliberately in team projects — covered in a later lesson)
```

## Common Pitfalls

- **Running `terraform apply` without reading the `plan` output first** — `apply` runs its own plan internally and prompts for confirmation, but skimming past that confirmation (or using `-auto-approve` habitually) is how unintended destructive changes slip through. Always actually read what's being added/changed/destroyed.
- **Not pinning a provider version** — an unconstrained `required_providers` entry can silently pull a newer major version on a future `terraform init`, potentially changing resource behavior or argument names unexpectedly. Always set a version constraint.
- **Committing `.terraform/` or `.tfstate` to Git** — `.terraform/` is regenerable local cache; `.tfstate` files can contain sensitive resource attributes (sometimes even secrets, depending on the resource) and represent real infrastructure state that shouldn't live in plain Git history. `.gitignore` both (state management gets its own dedicated lesson).
- **Confusing a resource's local name with its actual cloud-side name/tag** — `resource "aws_instance" "web"` — `"web"` is purely a Terraform-file reference name; the actual instance's name as seen in the AWS console comes entirely from whatever `tags = { Name = "..." }` (or equivalent) you set.
- **Assuming `depends_on` is usually needed** — Terraform infers most dependencies automatically from attribute references between resources; explicitly adding `depends_on` everywhere "just in case" adds unnecessary rigidity and often signals a design that could instead just reference the needed attribute directly.
- **Forgetting `terraform init` after adding a new provider or module** — editing `required_providers` or adding a `module` block doesn't take effect until `init` is run again to download what's newly required; `plan`/`apply` will error out until then.

## FAQ

**Q: What's the practical difference between `plan` and `apply`?**
A: `plan` is entirely read-only — it computes and shows what *would* change, touching nothing real. `apply` actually performs those changes against real infrastructure (after generating and confirming its own internal plan). Always treat `plan` as the safe way to preview, and `apply` as the point of no return.

**Q: Why does Terraform need a provider plugin instead of just calling cloud APIs directly from its core?**
A: Keeping providers as separate plugins lets Terraform support dozens of platforms (cloud and non-cloud) without every user needing every platform's SDK bundled into Terraform's core; you only download the providers your configuration actually references.

**Q: Is idempotency guaranteed no matter what I do?**
A: Only if you always go through Terraform. If someone manually changes or deletes a resource Terraform manages (via a cloud console, say), that's **drift** — the next `plan` will detect the difference between real state and desired state and propose bringing it back in line, but idempotency assumes Terraform is the only thing making changes.

**Q: When would I actually need `lifecycle { prevent_destroy = true }`?**
A: For infrastructure that would be catastrophic or very costly to accidentally destroy — a production database, a resource holding data with no easy backup. It's a deliberate safety rail, not something needed on most resources.

**Q: Why does `.terraform.lock.hcl` exist separately from `required_providers`' version constraint?**
A: The constraint (`~> 5.0`) allows a *range* of versions; the lock file records the *exact* version actually selected and downloaded, so that everyone running `terraform init` against the same configuration gets the identical provider version, not just "something matching the range" — this becomes more relevant once team collaboration is covered later in this module.

## Practice / Exercise

**Core:**
1. Install Terraform (or confirm it's already available), and run `terraform version` to check.
2. Write a minimal configuration with a `required_providers` block and a `provider` block for any provider you have credentials for (or use the `local` provider, which needs no credentials, to practice safely without touching real infrastructure).
3. Add one resource block, run `terraform init`, `terraform validate`, and `terraform plan`, and read the plan output carefully before doing anything else.
4. Run `terraform apply`, then `terraform show` to inspect the resulting state.
5. Change one argument on the resource, run `plan` again, and identify exactly which line of output shows the change.
6. Run `terraform destroy` and confirm the resource is removed.
7. Write a `.gitignore` for the project covering `.terraform/`, `*.tfstate`, and `*.tfstate.backup`.

**Stretch:**
1. Add a second resource with an explicit `depends_on` referencing the first, even though no attribute is actually shared between them, and explain in your own words when this would be necessary in a real project versus when it's unnecessary.
2. Add a `lifecycle { create_before_destroy = true }` block to a resource, make a change that forces replacement, and (reading the plan output) explain the difference in behavior this causes compared to the default.
3. Deliberately remove the version constraint from `required_providers`, run `terraform init -upgrade`, and observe what version gets selected — then restore a pinned constraint and explain why pinning matters for a team.

## Further Reading

- [Terraform documentation](https://developer.hashicorp.com/terraform/docs)
- [HCL syntax reference](https://developer.hashicorp.com/terraform/language/syntax/configuration)
- [Terraform Registry](https://registry.terraform.io/) — browse available providers and their resource documentation
