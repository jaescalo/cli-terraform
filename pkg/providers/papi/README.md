# PAPI provider notes: `--environment-layout`

This document explains how to use the `export-property` command with the `--environment-layout` feature.

## What `--environment-layout` does

When enabled, `export-property` generates a multi-environment Terraform scaffold:

- Root module files in the export directory
- Reusable property module under `modules/property`
- Environment-specific tfvars under `environments/dev` and `environments/prod`
- Split rules module under `modules/property/rules`

## Required flags

`--environment-layout` must be used together with:

- `--rules-as-hcl`
- `--split-depth <n>`
- `--rule-format <version>` where `<version>` is `v2026-01-09` or newer

## Export command example

```bash
akamai terraform export-property \
  --rules-as-hcl \
  --split-depth 1 \
  --rule-format v2026-01-09 \
  --environment-layout \
  --tfworkpath ./out \
  "my-property"
```

## Expected output layout

After export, the directory tree should look like this:

```text
out/
├── main.tf
├── variables.tf
├── import.sh
├── environments/
│   ├── dev/
│   │   └── terraform.tfvars
│   └── prod/
│       └── terraform.tfvars
└── modules/
    └── property/
        ├── property.tf
        ├── variables.tf
        └── rules/
            ├── module_config.tf
            ├── variables.tf
            └── *.tf
```

## Usage flow (dev/prod)

From the export directory (for example `./out`):

1. Initialize Terraform once:

```bash
terraform init
```

2. Plan/apply for dev:

```bash
terraform plan -var-file=environments/dev/terraform.tfvars
terraform apply -var-file=environments/dev/terraform.tfvars
```

3. Plan/apply for prod:

```bash
terraform plan -var-file=environments/prod/terraform.tfvars
terraform apply -var-file=environments/prod/terraform.tfvars
```

## Additional optimizations in `--environment-layout`

Beyond folder scaffolding, `--environment-layout` also parameterizes selected rules data so environment-specific values can be changed in tfvars without editing generated rules templates.

### 1) PMUSER variables (`rule_variables`)

- Root rule variables are exported into a `rule_variables` map.
- In scaffold mode, generated rules use `var.rule_variables` instead of hardcoded values for these rule variables.
- This allows per-environment overrides in:
  - `environments/dev/terraform.tfvars`
  - `environments/prod/terraform.tfvars`

### 2) Origin hostname parameterization

- For each rules behavior with `name == "origin"` and `options.hostname`, the exporter generates a scalar variable.
- Variable naming pattern:
  - `rule_<rule_name>_origin_hostname`
- When duplicate rule names appear, deterministic suffixes are added so names remain unique.
- Generated rules reference these variables directly, enabling environment-specific origins from tfvars.

### 3) CP code ID parameterization

- For each rules behavior with `name == "cpCode"` and `options.value.id`, the exporter generates a scalar variable.
- Variable naming pattern:
  - `rule_<rule_name>_cp_code`
- Values are typed as numbers in variable schemas and can differ by environment using tfvars.

### Where these variables are defined

- `variables.tf` (root): declares `rule_variables` and generated scalar variables.
- `modules/property/variables.tf`: mirrors variable definitions for module inputs.
- `modules/property/rules/variables.tf`: includes generated scalar rule variables with defaults from exported values.
- `environments/*/terraform.tfvars`: contains environment-specific assignments.

### Current scope and compatibility

- These parameterization behaviors are active in scaffold mode with split HCL rules.
- Rules template coverage is currently implemented for `--rule-format v2026-01-09` (and newer where equivalent logic exists).
- Legacy mode (without `--environment-layout`) keeps existing flat-file behavior and does not force this scaffold parameterization flow.

## Terraform patterns used in scaffold mode

The generated Terraform favors structured variables plus iterator-based resource generation. This keeps generated code stable while allowing per-environment values to vary through tfvars.

### `edge_hostnames`: map + `for_each`

- `edge_hostnames` is modeled as a map/object collection keyed by a stable hostname key.
- The property module creates `akamai_edge_hostname` resources using `for_each = var.edge_hostnames`.
- Result:
  - one Terraform resource instance per map key
  - stable addressing by key (better diffs than index-based lists)
  - easy environment overrides by replacing map values in `environments/*/terraform.tfvars`

Conceptually:

```hcl
resource "akamai_edge_hostname" "edge_hostnames" {
  for_each = var.edge_hostnames

  # fields read from each.value
}
```

### `property_hostnames`: dynamic nested blocks

- `property_hostnames` is modeled as structured input consumed by the property resource.
- The property resource emits nested `hostnames {}` blocks using a Terraform `dynamic` block.
- Result:
  - the number of nested hostname blocks automatically matches input data
  - optional nested attributes (for example `ccm_certificates`, `mtls`, `tls_configuration`) are included only when present

Conceptually:

```hcl
resource "akamai_property" "property" {
  # ...

  dynamic "hostnames" {
    for_each = var.property_hostnames
    content {
      # fields read from hostnames.value
    }
  }
}
```

### `rule_variables`: map-driven rule variable blocks

- `rule_variables` captures PMUSER-style rule variables as a map of objects.
- In scaffold mode, rule variable declarations in generated rules are driven from `var.rule_variables`.
- Result:
  - rule variable definitions remain generic in templates
  - environment-specific overrides live in tfvars, not in generated rule template files

Conceptually:

```hcl
variable "rule_variables" {
  # map(object(...))
}

# rules template logic consumes var.rule_variables to emit variable blocks
```

### Why this combination is used

- `for_each` on maps gives deterministic resource identity and safer plan output when items are added/removed.
- `dynamic` blocks avoid hardcoding a fixed number of nested blocks in resources like `akamai_property`.
- map/object input schemas keep root, module, and env tfvars aligned for multi-environment reuse.