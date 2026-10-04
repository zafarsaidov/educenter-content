# Lesson 23: Terraform Modules, Functions & Dynamic Blocks

**Module:** Terraform – Infrastructure as Code
**Duration:** 120-150 min
**Prerequisites:** Lessons 20-22 (Terraform Basics, Variables/Outputs/Data Sources, State Management)

## Learning Objectives

By end of lesson student can:
- Explain the root/child module relationship, and write a reusable local module
- Call a module with the `module` block, referencing local, Git, and registry sources with version pinning
- Use Terraform's common built-in functions for string/collection manipulation
- Write `for` expressions to transform and filter lists/maps, and conditional expressions
- Use `dynamic` blocks to generate repeated nested blocks from a list, and splat expressions to reference multiple resource instances

## Topics

- Modules: root/child module concepts; creating a local module (`variables.tf`, `outputs.tf`, `main.tf`)
- Calling modules: `module` block, `source` (local path, git, registry), version pinning
- Built-in functions: `format`, `join`, `split`, `merge`, `flatten`, `concat`, `lookup`
- For expressions: `[for item in list : expr]`, `{for k,v in map : k => v}`, filtering with `if`
- Conditional expressions: `condition ? true_val : false_val`; `try()` for error handling
- Dynamic blocks: `dynamic "ingress" { for_each, content {} }`; splat expressions `resource.*.attr`

## Concepts

### Root and child modules

Every Terraform configuration has at least one module: the **root module** — the directory you actually run `terraform apply` from. A **child module** is any self-contained, reusable set of `.tf` files called *from* another module (root or another child) via a `module` block. This is Terraform's answer to "don't repeat yourself": instead of copy-pasting the same 40 lines of resource blocks for every environment or every team that needs "a standard web server setup," that logic lives once in a module, and each caller supplies only the handful of inputs that actually differ.

A well-formed local module directory looks just like a small root module of its own:

```
modules/web-server/
├── main.tf        # resource blocks
├── variables.tf   # input variables this module accepts
└── outputs.tf     # values this module exposes back to its caller
```

A module's `variables.tf` defines its *public interface* — what the caller must/may supply; its `outputs.tf` defines what the caller can read back afterward. Everything else inside the module (its internal resource blocks, locals) is effectively private — invisible to, and not directly referenceable by, whoever calls it.

### Calling a module

```hcl
module "web" {
    source  = "./modules/web-server"
    version = "~> 1.0"                    # only meaningful for registry sources, not local paths

    instance_type = var.instance_type
    environment   = var.environment
}
```

`source` tells Terraform where the module's code lives:

| Source type | Example |
|---|---|
| Local path | `source = "./modules/web-server"` |
| Git repository | `source = "git::https://github.com/org/repo.git//modules/web-server?ref=v1.2.0"` |
| Terraform Registry | `source = "terraform-aws-modules/vpc/aws"` |

Version pinning (`version = "~> 1.0"`) applies to registry and some Git sources (via the `ref` query parameter for Git specifically) — pinning a module's version matters for exactly the same reason as pinning a provider's version (Lesson 20): an unpinned module reference can silently start behaving differently after a routine `init` if the module's source has moved on. A local path module has no separate version to pin — it's exactly whatever's in that directory right now, versioned implicitly through Git itself (Lesson 14-15) since it typically lives in the same repository.

A module's outputs are referenced from the caller as `module.<name>.<output>`, e.g. `module.web.instance_id`.

### Built-in functions

Terraform ships a substantial standard library of functions, usable anywhere an expression is valid (not just inside specific blocks). A handful come up constantly:

| Function | Purpose | Example |
|---|---|---|
| `format` | `printf`-style string formatting | `format("%s-%03d", "web", 7)` → `"web-007"` |
| `join` | Combine a list into a string | `join(", ", ["a", "b", "c"])` → `"a, b, c"` |
| `split` | Break a string into a list | `split(",", "a,b,c")` → `["a","b","c"]` |
| `merge` | Combine two or more maps | `merge({a=1}, {b=2})` → `{a=1, b=2}` |
| `flatten` | Flatten a nested list into one level | `flatten([["a","b"],["c"]])` → `["a","b","c"]` |
| `concat` | Combine multiple lists | `concat(["a"], ["b","c"])` → `["a","b","c"]` |
| `lookup` | Safe map access with a default | `lookup({a=1}, "b", 0)` → `0` (key missing, returns default instead of erroring) |

These parallel ideas already familiar from both Bash string manipulation (Lesson 16) and Python's string/list methods (Lesson 19) — the same underlying operations, expressed in HCL's own function syntax.

### `for` expressions

A **for expression** builds a new list or map by transforming (and optionally filtering) an existing one — the HCL equivalent of Python's list comprehensions (Lesson 19):

```hcl
# list -> list, transforming each element
[for name in ["web", "api", "db"] : upper(name)]
# -> ["WEB", "API", "DB"]

# list -> list, with filtering
[for n in [10, 55, 90, 20] : n if n > 50]
# -> [55, 90]

# map -> map, transforming both keys and values
{for k, v in {a = 1, b = 2} : k => v * 10}
# -> {a = 10, b = 20}
```

### Conditional expressions and `try()`

A **conditional expression** is HCL's ternary operator — the one-line equivalent of a simple `if`/`else`:

```hcl
instance_type = var.environment == "production" ? "t3.large" : "t3.micro"
```

`try()` attempts each given expression in order and returns the first one that doesn't error — commonly used to gracefully fall back when a value *might* not exist (e.g. an optional field deep in a data structure) rather than letting the whole `plan` fail outright:

```hcl
value = try(var.config.region, "eu-west-1")
```

### Dynamic blocks

Some resource arguments aren't simple key-value pairs — they're repeated **nested blocks** (a classic example: a security group's multiple `ingress` rules, each its own block). Writing one out manually for every rule doesn't scale when the actual list of rules comes from a variable. A `dynamic` block generates one nested block per item in a given collection:

```hcl
variable "ingress_rules" {
    type = list(object({
        port        = number
        protocol    = string
        cidr_blocks = list(string)
    }))
}

resource "aws_security_group" "web" {
    name = "web-sg"

    dynamic "ingress" {
        for_each = var.ingress_rules
        content {
            from_port   = ingress.value.port
            to_port     = ingress.value.port
            protocol    = ingress.value.protocol
            cidr_blocks = ingress.value.cidr_blocks
        }
    }
}
```

Inside `content { }`, `ingress.value` refers to the current item being iterated (named after the dynamic block's own label, `"ingress"`) — this generates exactly as many `ingress { }` blocks as there are items in `var.ingress_rules`, without writing each one by hand.

### Splat expressions

A **splat expression** (`resource.*.attribute`) collects one attribute from *every* instance of a resource created with `count` or `for_each`, into a single list — shorthand for a `for` expression that would otherwise need to be written out explicitly:

```hcl
resource "aws_instance" "web" {
    count = 3
    # ...
}

output "all_ips" {
    value = aws_instance.web[*].public_ip     # splat: one IP per instance, as a list
}
```

`aws_instance.web[*].public_ip` is exactly equivalent to `[for instance in aws_instance.web : instance.public_ip]` — the splat form is simply more concise for this very common case.

## Commands / Syntax Reference

| Syntax | Purpose | Example |
|---|---|---|
| `module "name" { source = ... }` | Call a module | see walkthrough |
| `module.name.output` | Reference a module's output | `module.web.instance_id` |
| `format(...)` | printf-style formatting | `format("%s-%d", "x", 1)` |
| `merge(...)` | Combine maps | `merge(a, b)` |
| `[for x in list : expr]` | For expression (list) | `[for n in nums : n * 2]` |
| `{for k,v in map : k => v}` | For expression (map) | `{for k,v in m : k => upper(v)}` |
| `cond ? a : b` | Conditional expression | `var.env == "prod" ? "large" : "small"` |
| `try(a, b)` | Fallback on error | `try(var.x.y, "default")` |
| `dynamic "block_name" { }` | Generate repeated nested blocks | see walkthrough |
| `resource.*.attr` | Splat: collect one attribute from all instances | `aws_instance.web[*].id` |

## Examples / Walkthrough

```
# --- modules/web-server/ : a reusable local module ---
modules/web-server/
├── main.tf
├── variables.tf
└── outputs.tf
```

```hcl
# --- modules/web-server/variables.tf ---
variable "instance_type" {
    type    = string
    default = "t3.micro"
}

variable "environment" {
    type = string
}
```

```hcl
# --- modules/web-server/main.tf ---
resource "aws_instance" "this" {
    ami           = "ami-0123456789abcdef0"
    instance_type = var.instance_type

    tags = {
        Name        = "${var.environment}-web"
        Environment = var.environment
    }
}
```

```hcl
# --- modules/web-server/outputs.tf ---
output "instance_id" {
    value = aws_instance.this.id
}

output "public_ip" {
    value = aws_instance.this.public_ip
}
```

```hcl
# --- root main.tf: calling the module twice, for two environments ---
module "web_dev" {
    source        = "./modules/web-server"
    instance_type = "t3.micro"
    environment   = "dev"
}

module "web_prod" {
    source        = "./modules/web-server"
    instance_type = "t3.large"
    environment   = "production"
}

output "dev_ip" {
    value = module.web_dev.public_ip
}
```

```hcl
# --- built-in functions ---
locals {
    instance_name = format("%s-%s-%03d", var.environment, "web", 7)   # "dev-web-007"

    all_tags = merge(
        { ManagedBy = "terraform" },
        { Environment = var.environment },
    )

    cidr_list = split(",", "10.0.1.0/24,10.0.2.0/24")                  # -> ["10.0.1.0/24", "10.0.2.0/24"]

    az_value = lookup(var.az_map, var.environment, "eu-west-1a")        # safe default if key missing
}
```

```hcl
# --- for expressions and conditionals ---
variable "server_names" {
    type    = list(string)
    default = ["web-01", "web-02", "api-01"]
}

locals {
    upper_names   = [for n in var.server_names : upper(n)]
    web_only      = [for n in var.server_names : n if startswith(n, "web")]
    name_to_index = {for i, n in var.server_names : n => i}

    instance_type = var.environment == "production" ? "t3.large" : "t3.micro"
    region        = try(var.override_region, "eu-west-1")
}
```

```hcl
# --- dynamic block: security group with a variable number of ingress rules ---
variable "ingress_rules" {
    type = list(object({
        port        = number
        protocol    = string
        cidr_blocks = list(string)
    }))
    default = [
        { port = 22, protocol = "tcp", cidr_blocks = ["10.0.0.0/16"] },
        { port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
        { port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    ]
}

resource "aws_security_group" "web" {
    name = "web-sg"

    dynamic "ingress" {
        for_each = var.ingress_rules
        content {
            from_port   = ingress.value.port
            to_port     = ingress.value.port
            protocol    = ingress.value.protocol
            cidr_blocks = ingress.value.cidr_blocks
        }
    }
}
```

```hcl
# --- splat expression: collecting an attribute across multiple instances ---
resource "aws_instance" "web" {
    count         = 3
    ami           = "ami-0123456789abcdef0"
    instance_type = "t3.micro"
}

output "all_ids" {
    value = aws_instance.web[*].id            # splat: list of all 3 instance IDs
}

output "all_ips" {
    value = aws_instance.web[*].public_ip      # equivalent to: [for i in aws_instance.web : i.public_ip]
}
```

```bash
terraform init          # resolves module sources, including downloading any Git/registry modules
terraform plan
terraform apply
terraform state list     # module resources appear as module.web_dev.aws_instance.this, etc.
```

## Common Pitfalls

- **Forgetting `terraform init` after adding/changing a `module` block** — just like providers (Lesson 20), a new or changed module source isn't fetched until `init` runs again; `plan` errors out referencing a module Terraform hasn't downloaded yet.
- **Hardcoding values inside a module instead of exposing them as variables** — defeats the purpose of reusability; if every caller needs the exact same `instance_type`, that's fine to hardcode, but anything that genuinely varies between callers belongs in `variables.tf`.
- **Not pinning a Git or registry module's version** — identical risk to an unpinned provider: a module's `main` branch (or an un-pinned registry version) can change underneath you on the next `init`, silently altering behavior.
- **Confusing a `for` expression's list form and map form syntax** — `[for x in list : expr]` (square brackets, single result per item) versus `{for k, v in map : k => v}` (curly braces, explicit key => value pairs) look similar but aren't interchangeable; using the wrong brackets is an immediate syntax error.
- **Using `dynamic` for a fixed, small, known set of blocks** — if a resource always needs exactly 2 specific nested blocks that never vary, writing them directly is clearer than wrapping them in `dynamic` machinery; `dynamic` earns its complexity specifically when the *number* of blocks needs to vary based on input.
- **Relying on splat syntax (`resource.*.attr`) without understanding what it expands to** — it silently returns an empty list if the resource doesn't use `count`/`for_each` at all (a single, non-counted resource), which can be a confusing surprise if a resource block's `count` is later removed without updating code that references it with `[*]`.

## FAQ

**Q: When is it worth extracting something into a module versus just writing it directly?**
A: Once the same group of resources needs to be created more than once (multiple environments, multiple teams, multiple similar apps) with only a few differing inputs — if it's truly a one-off, a module just adds indirection without a real reuse payoff.

**Q: Can a module call another module?**
A: Yes — a child module can itself contain `module` blocks calling further modules, forming a nested hierarchy. This is common for breaking a large module into smaller, focused pieces (e.g. a "full environment" module composed of a "networking" module plus a "compute" module).

**Q: What's the practical difference between `for` expressions and splat expressions?**
A: A splat expression is a concise shorthand specifically for "collect one attribute from every instance of a `count`/`for_each` resource" — exactly the single most common `for`-expression use case. Anything more involved (transforming, filtering with a condition, building a map instead of a list) needs the full `for` expression syntax; splat can't do those.

**Q: Why would I use `try()` instead of just referencing the value directly?**
A: When a value might legitimately not exist — an optional nested field in a complex input object, a data source result that might be empty — `try()` lets the configuration gracefully fall back to a default instead of the entire `plan`/`apply` erroring out on that one missing reference.

**Q: Does a `dynamic` block work with a map as well as a list?**
A: Yes — `for_each` accepts a map too (iterating its key-value pairs, accessible inside `content { }` as `each.key`/`each.value` in that case, since the block's own name — `ingress.value` in the earlier example — is really just `dynamic`'s automatic default name for the iteration variable); maps are particularly useful when each generated block also needs a stable identity tied to a specific key, not just position in a list.

## Practice / Exercise

**Core:**
1. Create a local module (`modules/web-server/` with `main.tf`, `variables.tf`, `outputs.tf`) wrapping a single resource (or the `local_file`/`random_pet` providers if you don't have cloud credentials, to practice the module mechanics without real infrastructure), accepting at least 2 input variables and exposing at least 1 output.
2. Call that module twice from a root configuration with different variable values, and confirm (via `terraform state list`) both instances are tracked separately under `module.<name>.*`.
3. Write a `locals` block using at least 3 different built-in functions (`format`, `merge`, `join`, `split`, or `lookup`) and print their results via `output` blocks.
4. Write a `for` expression that filters a list of names down to only ones matching some condition, and a second `for` expression building a map from a list.
5. Write a conditional expression that picks between two values based on a variable, and a `try()` expression with a fallback default.
6. Write a resource using a `dynamic` block driven by a `list(object(...))` variable, with at least 2 items in the default value, and confirm via `plan` that the correct number of nested blocks get generated.
7. Create 3 instances of a resource using `count`, and write an output using a splat expression to collect one attribute from all 3.

**Stretch:**
1. Pin a module source to a specific Git tag (`?ref=v1.0.0`) against any public Terraform module repository you can find, run `init`, and confirm the pinned version is what gets downloaded.
2. Refactor a `dynamic` block's `for_each` to iterate a `map` instead of a `list`, adjusting `content { }` to use `each.key`/`each.value`, and explain when a map's extra "stable key" property would matter in a real scenario (e.g. adding/removing one rule without disturbing others' generated block identity).
3. Rewrite a splat expression (`resource.*.attr`) as an explicit `for` expression, and confirm both produce identical output.

## Further Reading

- [Terraform: Modules](https://developer.hashicorp.com/terraform/language/modules)
- [Terraform: Built-in Functions](https://developer.hashicorp.com/terraform/language/functions)
- [Terraform: For Expressions](https://developer.hashicorp.com/terraform/language/expressions/for)
- [Terraform: Dynamic Blocks](https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks)
- [Terraform Registry](https://registry.terraform.io/) — browse reusable community/official modules
