# Lesson 21: Variables, Outputs & Data Sources

**Module:** Terraform – Infrastructure as Code
**Duration:** 120-150 min
**Prerequisites:** Lesson 20 (Terraform Basics – Providers, Resources, HCL)

## Learning Objectives

By end of lesson student can:
- Declare input variables with types, defaults, descriptions, and validation rules
- Supply variable values via `.tfvars` files, `-var`, and environment variables, and explain precedence
- Use `locals` for computed/derived values
- Declare and read output values, including marking one `sensitive`
- Query existing infrastructure with a `data` source and reference it in a resource

## Topics

- Input variables: `variable` block, types, `default`, `description`, `validation` block
- Variable files: `.tfvars`, `.auto.tfvars`, `-var` flag, `TF_VAR_` environment variables
- Locals: local values, computed expressions, use cases vs input variables
- Output values: `output` block, `sensitive = true`, `terraform output` command, using in scripts
- Data sources: `data` block, filtering arguments, referencing `data.*.attribute` in resources

## Concepts

### Input variables

A `variable` block declares a named input the configuration expects from outside — the equivalent of a function parameter for a whole Terraform configuration:

```hcl
variable "instance_type" {
    type        = string
    default     = "t3.micro"
    description = "EC2 instance type for the web server"
}
```

- `type` — constrains what kind of value is acceptable (`string`, `number`, `bool`, `list(string)`, `map(string)`, etc.); Terraform rejects a mismatched value at plan time rather than letting a wrong type silently cause a confusing failure later.
- `default` — optional; if omitted, the variable becomes *required* — Terraform will prompt for it interactively (or fail in non-interactive contexts like CI) if no value is supplied another way.
- `description` — not functionally required, but genuinely important documentation, especially for a variable other people (or future you) will need to fill in without reading the whole module's internals.

A `validation` block adds a custom rule beyond just the type:

```hcl
variable "environment" {
    type = string
    validation {
        condition     = contains(["dev", "staging", "production"], var.environment)
        error_message = "environment must be one of: dev, staging, production."
    }
}
```

This catches a typo'd or invalid value immediately at `plan`/`apply` time with a clear message, rather than letting it silently propagate into resource names or tags.

### Supplying variable values: `.tfvars`, `-var`, and environment variables

A `variable` block only *declares* a variable — its actual value comes from somewhere else, and Terraform checks several sources in a defined precedence order (highest wins when the same variable is set in more than one place):

| Source | Precedence |
|---|---|
| `-var="key=value"` on the CLI | Highest |
| `-var-file="file.tfvars"` on the CLI | |
| `*.auto.tfvars` (auto-loaded, no flag needed) | |
| `terraform.tfvars` (auto-loaded, no flag needed) | |
| `TF_VAR_<name>` environment variable | |
| `default` in the `variable` block | Lowest |

In practice: `.tfvars` files are the standard way to supply a full set of values for one environment (`dev.tfvars`, `prod.tfvars`), loaded with `-var-file`. Files literally named `terraform.tfvars` or ending in `.auto.tfvars` are picked up automatically without any flag — convenient, but also a common source of "why did this apply differently than I expected" confusion if such a file exists unnoticed in the directory. `TF_VAR_name=value` environment variables are useful for CI pipelines injecting secrets without writing them to a file at all.

### `locals`: computed values

A `locals` block defines named values computed *within* the configuration — unlike a `variable`, a local isn't an external input, it's derived from other values (variables, resource attributes, or plain expressions):

```hcl
locals {
    name_prefix = "${var.environment}-${var.project_name}"
    common_tags = {
        Environment = var.environment
        ManagedBy   = "terraform"
    }
}
```

Use a `variable` for something that genuinely needs to be supplied from outside; use a `locals` value for something computed *from* those inputs, to avoid repeating the same expression (like a tag map, or a naming convention) across many resource blocks.

### Output values

An `output` block exposes a value after `apply` — often a resource attribute that isn't known until the resource is actually created (like an assigned IP address, or a generated ID):

```hcl
output "instance_ip" {
    value       = aws_instance.web.public_ip
    description = "Public IP address of the web server"
}

output "db_password" {
    value     = random_password.db.result
    sensitive = true
}
```

`sensitive = true` prevents the value from being printed in `plan`/`apply`'s console output (shown as `(sensitive value)` instead) — it does **not** encrypt or remove it from the state file, which still holds the real value in plain form (state file security is its own later lesson's concern). `terraform output` prints every output after an apply; `terraform output -raw name` prints one value without quotes, useful for piping into a script (Lesson 16/17's territory) rather than for human reading.

### Data sources: reading existing infrastructure

A `data` block doesn't create or manage anything — it **reads** information about something that already exists, outside this configuration's own management, and makes it available to reference:

```hcl
data "aws_ami" "ubuntu" {
    most_recent = true
    owners      = ["099720109477"]     # Canonical's official AWS account ID

    filter {
        name   = "name"
        values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-*"]
    }
}

resource "aws_instance" "web" {
    ami           = data.aws_ami.ubuntu.id     # reference the data source's result
    instance_type = var.instance_type
}
```

This is the standard pattern for referencing something Terraform shouldn't manage the lifecycle of but still needs to know about — an existing VPC created by another team, the latest AMI ID (which changes over time and shouldn't be hardcoded), an existing DNS zone. `data` sources are read-only and re-evaluated on every `plan`, so they always reflect current real-world state at that moment.

## Commands / Syntax Reference

| Syntax | Purpose | Example |
|---|---|---|
| `variable "name" { }` | Declare an input variable | see walkthrough |
| `-var="key=value"` | Set one variable via CLI | `terraform apply -var="env=dev"` |
| `-var-file="file"` | Load a `.tfvars` file | `terraform apply -var-file="dev.tfvars"` |
| `TF_VAR_name` | Set a variable via environment | `export TF_VAR_environment=dev` |
| `locals { }` | Declare computed local values | see walkthrough |
| `output "name" { }` | Declare an output | see walkthrough |
| `terraform output` | Print all outputs | `terraform output` |
| `terraform output -raw` | Print one output, unquoted | `terraform output -raw instance_ip` |
| `data "type" "name" { }` | Query existing infrastructure | see walkthrough |

## Examples / Walkthrough

```hcl
# --- variables.tf ---
variable "environment" {
    type        = string
    description = "Deployment environment name"

    validation {
        condition     = contains(["dev", "staging", "production"], var.environment)
        error_message = "environment must be one of: dev, staging, production."
    }
}

variable "instance_type" {
    type        = string
    default     = "t3.micro"
    description = "EC2 instance type"
}

variable "instance_count" {
    type        = number
    default     = 1
    description = "Number of instances to create"
}
```

```hcl
# --- locals.tf ---
locals {
    name_prefix = "${var.environment}-devops-basics"

    common_tags = {
        Environment = var.environment
        ManagedBy   = "terraform"
        Project     = "devops-basics-training"
    }
}
```

```hcl
# --- data.tf: reading the latest Ubuntu AMI instead of hardcoding an ID ---
data "aws_ami" "ubuntu" {
    most_recent = true
    owners      = ["099720109477"]

    filter {
        name   = "name"
        values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-*"]
    }
}
```

```hcl
# --- main.tf: putting it all together ---
resource "aws_instance" "web" {
    count = var.instance_count

    ami           = data.aws_ami.ubuntu.id      # from the data source above
    instance_type = var.instance_type

    tags = merge(local.common_tags, {
        Name = "${local.name_prefix}-web-${count.index}"
    })
}
```

```hcl
# --- outputs.tf ---
output "instance_ids" {
    value       = aws_instance.web[*].id
    description = "IDs of all created instances"
}

output "ami_used" {
    value       = data.aws_ami.ubuntu.id
    description = "AMI ID resolved by the data source"
}
```

```bash
# --- dev.tfvars: one file per environment ---
cat > dev.tfvars <<'EOF'
environment     = "dev"
instance_type   = "t3.micro"
instance_count  = 1
EOF

# --- supplying variables, in increasing precedence ---
terraform plan                                       # uses only defaults + any *.auto.tfvars present
terraform plan -var-file="dev.tfvars"                 # explicit file
terraform plan -var-file="dev.tfvars" -var="instance_count=2"   # -var overrides the file's value

export TF_VAR_environment="dev"                        # environment variable, useful in CI
terraform plan -var-file="dev.tfvars"                    # -var-file still wins over TF_VAR_ for the same key

terraform apply -var-file="dev.tfvars"

terraform output                                          # print every output
terraform output -raw ami_used                              # print one value, unquoted, script-friendly

# capturing an output into a shell variable — ties directly back to Lesson 16/17
INSTANCE_IDS=$(terraform output -json instance_ids)
echo "Created instances: $INSTANCE_IDS"
```

## Common Pitfalls

- **A required variable with no default failing in CI** — a `variable` block with no `default` prompts interactively when run by hand, but a CI pipeline (Lesson 59+'s territory) has no interactive prompt to answer, so the run just fails. Always supply required variables explicitly via `-var-file` or `TF_VAR_` in automated contexts.
- **An unnoticed `terraform.tfvars` or `*.auto.tfvars` file silently changing behavior** — since these load automatically with no flag, a leftover file from earlier experimentation (or one a teammate added) can cause `plan`/`apply` to behave differently than someone expects who isn't aware it exists. Keep the working directory's `.tfvars` files intentional and visible.
- **Assuming `sensitive = true` hides a value from the state file** — it only suppresses console/log output; the real value is still stored in plain form in `.tfstate`. Treat state files as sensitive regardless of which outputs are marked `sensitive`.
- **Overusing `locals` for values that should really be `variable`s** — if a value genuinely needs to differ between environments or be supplied by whoever runs the configuration, it belongs in `variable`, not hardcoded inside a `locals` block that then needs editing directly to change.
- **Referencing a `data` source before understanding it re-runs on every plan** — unlike a `resource` (created once, then only updated on drift/config change), a `data` source's query is re-evaluated fresh every single `plan`; if the underlying real-world data changes (e.g. a new AMI is published), the next plan can show an unexpected diff purely from the data source resolving to a new value, without any config change on your part.
- **Forgetting `merge()` when combining tag maps** — writing `tags = local.common_tags, { Name = "x" }` isn't valid HCL; combining two maps requires the `merge()` function, as shown in the walkthrough.

## FAQ

**Q: Why bother with a `validation` block instead of just documenting the expected values?**
A: Documentation can be missed or ignored; a `validation` block makes an invalid value a hard `plan`-time error with a clear message, catching a mistake immediately instead of letting a typo'd environment name silently create resources named `dev` when `production` was intended.

**Q: When should I use a `locals` value instead of just repeating an expression?**
A: As soon as the same computed expression (a naming convention, a merged tag map) would otherwise appear in more than one resource block — `locals` centralizes it, so a future change only needs to happen in one place.

**Q: Why would I use a `data` source instead of just hardcoding a value like an AMI ID?**
A: A hardcoded ID goes stale (a newer, patched AMI gets published regularly) and isn't portable across regions/accounts where the same image might have a different ID. A `data` source resolves the *current* correct value automatically every time, based on criteria (like "latest Ubuntu 22.04 image") rather than a specific ID that will eventually be wrong or deprecated.

**Q: What's the practical difference between `-var` and a `.tfvars` file?**
A: `-var` sets one value directly on the command line — quick for a single override or a CI pipeline injecting one or two values. A `.tfvars` file holds a complete, reusable set of values for a whole environment, checked into version control (when it doesn't contain secrets) so the same `dev.tfvars` reliably produces the same dev environment every time it's used.

**Q: Can I have both a `variable` and a `data` source with a similar-sounding purpose?**
A: Yes, and it's common — a `variable` might supply *which* environment/region to target, while a `data` source then *looks up* something specific within that environment/region (like an existing VPC ID) based on the variable's value.

## Practice / Exercise

**Core:**
1. Convert the previous lesson's hardcoded resource into one using variables: declare `instance_type` (with a default) and `environment` (required, with a `validation` block restricting it to a fixed set of values).
2. Create a `dev.tfvars` file supplying values for both variables, and run `terraform plan -var-file="dev.tfvars"`.
3. Deliberately supply an invalid `environment` value and confirm the `validation` block's error message appears.
4. Add a `locals` block computing a name prefix from `var.environment`, and use it in a resource's tags.
5. Add an `output` for at least one resource attribute, `apply`, then run `terraform output` and `terraform output -raw <name>`.
6. Add a second output marked `sensitive = true`, `apply`, and confirm the console shows `(sensitive value)` instead of the real value.
7. Add a `data` source appropriate to your provider (or use the `random` provider's `random_pet`/`random_id` as a dependency-free stand-in if you don't have cloud credentials), and reference its result in a resource argument.

**Stretch:**
1. Set `TF_VAR_environment` as an environment variable, then also pass a *different* value via `-var-file`, and confirm which one actually takes effect — explaining the precedence order that produced that result.
2. Deliberately create a stray `terraform.tfvars` file with a different `instance_type` than your explicit `-var-file`, and observe which value wins on `plan` — explain why, referencing the precedence table.
3. Write a short Bash script (Lesson 16/17) that runs `terraform output -json`, and uses a tool you haven't formally learned yet (worth a quick look: `jq`) to extract one specific field — note this is a common real pattern for gluing Terraform output into other automation.

## Further Reading

- [Terraform: Input Variables](https://developer.hashicorp.com/terraform/language/values/variables)
- [Terraform: Output Values](https://developer.hashicorp.com/terraform/language/values/outputs)
- [Terraform: Data Sources](https://developer.hashicorp.com/terraform/language/data-sources)
