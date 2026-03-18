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