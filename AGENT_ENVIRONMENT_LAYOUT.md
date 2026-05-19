# Agent Instructions: `environment-layout` for `export-property`

This document captures the full implementation requirements and behavior for the `environment-layout` feature added to `export-property`.

Use this as the source of truth when reproducing or porting this feature to future CLI versions.

---

## Goal

When exporting PAPI properties with HCL rules splitting, optionally generate a reusable multi-environment Terraform structure instead of a single flat output.

### Primary outcomes

1. Keep current behavior unchanged by default.
2. Enable a new optional layout via flag.
3. Make environment-specific values live in per-environment `terraform.tfvars`.
4. Keep property/rules logic in module(s).
5. Generate import script compatible with module addresses.

---

## Activation conditions

Feature is enabled only when:

- `--environment-layout=true`
- `--rules-as-hcl=true`
- `--split-depth` is set
- `--rule-format <version>` where `<version>` is `v2026-01-09` or newer

If these prerequisites are not met, command must fail validation.

---

## CLI and validation changes

### Command flag

In `export-property` command (`pkg/commands/commands.go`), add:

- `--environment-layout` (bool)
  - Usage: create multi-env scaffold (`root + modules/property + environments/*`) and requires `--rules-as-hcl` + `--split-depth`.

### Validators

In `pkg/commands/validation.go`:

- Existing `validateSplitDepth` remains.
- Add `validateEnvironmentLayout`:
  - if `environment-layout` is set and true:
    - require `rules-as-hcl` set and true
    - require `split-depth` set

Wire validator in `export-property` action chain.

### Validation tests

In `pkg/commands/validation_test.go` add/maintain tests for:

- flag not set => success
- missing `rules-as-hcl` => error
- missing `split-depth` => error
- valid combo => success

---

## Execution path changes (`CmdCreateProperty`)

File: `pkg/providers/papi/create_property.go`

### New option fields

`propertyOptions` includes:

- `useEnvironmentScaffold bool`
- `rulesOutputPath string`

### Decision logic

`useEnvironmentScaffold` comes **only** from `--environment-layout` (not implicitly from split-depth).

### Template target mapping

If scaffold mode ON, target files are:

- Root:
  - `main.tf`
  - `variables.tf`
  - `import.sh`
- Module:
  - `modules/property/property.tf`
  - `modules/property/variables.tf`
  - `modules/property/rules/*` (split files)
  - `modules/property/rules/module_config.tf`
  - `modules/property/rules/variables.tf`
- Environments:
  - `environments/dev/terraform.tfvars`
  - `environments/prod/terraform.tfvars`

If scaffold mode OFF, keep legacy targets:

- `property.tf`, `variables.tf`, `import.sh`, optional `rules.tf`, `rules/` as existing behavior dictates.

### Directory creation

For scaffold mode, ensure these exist before writing:

- `modules/property`
- `environments/dev`
- `environments/prod`

`createSplitRulesDir` must run against `rulesOutputPath` (`modules/property` in scaffold mode).

### Split rules output path

`prepareRulesForSplitRule(...)` should write to:

- `rulesOutputPath/rules/<rule_file>.tf`

not always `tfWorkPath/rules`.

---

## Template set for scaffold mode

### Added templates

Under `pkg/providers/papi/templates/`:

- `root_main.tmpl`
- `root_variables.tmpl`
- `module_property.tmpl`
- `module_variables.tmpl`
- `imports_module.tmpl`
- `env_dev_tfvars.tmpl`
- `env_prod_tfvars.tmpl`
- `rules_variables.tmpl`

### Existing templates still used

- `split-depth-rules.tmpl`
- `rules_v<date>.tmpl` (selected by rule format)
- `rules_module.tmpl` (outputs `rules` and `rule_format`)

---

## Required Terraform behavior

### Root module

`root_main.tmpl`:

- defines provider and required providers
- calls `module "property"` from `./modules/property`
- passes all variableized values including rule parameter maps

### Property module

`module_property.tmpl`:

- includes its own `terraform { required_providers ... }` block
- uses `for_each` for `akamai_edge_hostname` via `var.edge_hostnames`
- uses dynamic `hostnames` blocks via `var.property_hostnames`
- supports optional nested hostname fields:
  - `ccm_certificates`
  - `mtls`
  - `tls_configuration`
- for split depth: consumes `module "rules"`
- passes rule parameter maps to `module "rules"` **only when** scaffold mode is true

### Activation defaults

For property and include activations, default:

- `auto_acknowledge_rule_warnings = true`

This applies to:

- `property.tmpl`
- `module_property.tmpl`
- `includes.tmpl`

---

## Environment variableization (phase 2 state)

### In env tfvars

`env_dev_tfvars.tmpl` and `env_prod_tfvars.tmpl` include:

- `contract_id`, `group_id`
- `property_config`
- `edge_hostnames`
- `property_hostnames`
- `activation_config`
  - staging contacts/note
  - production contacts/note
- `version_notes`
- `activate_latest_on_staging`
- `activate_latest_on_production`
- `rule_variables`
- scalar `rule_<rule_name>_origin_hostname` entries
- scalar `rule_<rule_name>_cp_code` entries

### In root/module variable schemas

Both `root_variables.tmpl` and `module_variables.tmpl` define types for:

- `rule_variables` map(object)
- generated scalar string variables for each extracted origin hostname
- generated scalar number variables for each extracted cp code ID

`rules_variables.tmpl` mirrors the same generated scalar variables and sets defaults to exported values.

---

## Rules-module parameterization (current scope)

### Data extraction

In `create_property.go`, gather from exported rules:

- root rule `Variables` => `rule_variables`
- all `behavior.name == "origin"` + `options.hostname` => generated scalar variables (for example `rule_default_origin_hostname`)
- all `behavior.name == "cpCode"` + `options.value.id` => generated scalar variables (for example `rule_default_cp_code`)

Functions used:

- `collectRuleModuleParameters`
- `collectBehaviorParameters`
- `extractCPCodeID` (handles both map- and struct-shaped `cpCode.value` payloads)
- `asInt64`
- `derefString`

### Current template coverage

Parameterized in `rules_v2026-01-09.tmpl`:

1. Root `variable {}` blocks become dynamic when scaffold mode is on:
   - driven by `var.rule_variables`
2. `origin.hostname` uses direct scalar variable reference when scaffold mode is on:
  - `var.rule_<rule_name>_origin_hostname` (with deterministic suffixes for duplicates)
3. `cp_code.value.id` uses direct scalar variable reference when scaffold mode is on:
  - `var.rule_<rule_name>_cp_code` (with deterministic suffixes for duplicates)

This is implemented in the primary `origin` and `cpCode` behavior template blocks in `rules_v2026-01-09.tmpl`.

### Template scoping guardrail

When scaffold conditionals are used inside nested rule-behavior templates, do **not** rely on `$.UseEnvironmentScaffold`.

- Nested contexts (for example behavior-level data like `RuleBehavior`) may not expose the root object expected by `$.`.
- Use the registered helper function `UseEnvironmentScaffold` in templates instead.
- This avoids runtime execution errors such as: `can't evaluate field UseEnvironmentScaffold in type papi.RuleBehavior`.

### Important limitation

This parameterization is currently implemented for `rules_v2026-01-09.tmpl`.
If older rule formats need parity, repeat equivalent changes in each relevant `rules_v*.tmpl`.

---

## Import script behavior

`imports_module.tmpl` is used in scaffold mode and must import module addresses, for example:

- `module.property.akamai_edge_hostname.edge_hostnames["key"]`
- `module.property.akamai_property.<name>`
- `module.property.akamai_property_activation.<name>-staging`
- `module.property.akamai_property_activation.<name>-production`

---

## Backward compatibility requirements

1. Without `--environment-layout`, legacy output must remain unchanged.
2. Include export (`export-property-include`) should not be forced into scaffold flow.
3. Split-depth behavior remains available without scaffold mode.

---

## Test/update checklist when porting

### Required tests to run

- `go test ./pkg/commands`
- `go test ./pkg/providers/papi`

### Golden fixtures

If activation default flips or templates are modified, update expected fixtures under:

- `pkg/providers/papi/testdata/**`

Notably, expected strings for `auto_acknowledge_rule_warnings` must match defaults.

### Regression checks

- Export-property legacy mode still writes `property.tf`, `variables.tf`, `import.sh`.
- Export-property scaffold mode writes root/module/environments tree.
- Rules split files are placed under `modules/property/rules` in scaffold mode.
- Module property file includes required providers block.

---

## Quick reproduce command

Example scaffold export:

```bash
akamai terraform export-property \
  --rules-as-hcl \
  --split-depth 1 \
  --environment-layout \
  --tfworkpath ./out \
  "<PROPERTY_NAME>"
```

Expected key outputs:

- `out/main.tf`
- `out/variables.tf`
- `out/import.sh`
- `out/modules/property/property.tf`
- `out/modules/property/variables.tf`
- `out/modules/property/rules/*.tf`
- `out/modules/property/rules/module_config.tf`
- `out/modules/property/rules/variables.tf`
- `out/environments/dev/terraform.tfvars`
- `out/environments/prod/terraform.tfvars`

---

## Future extension ideas

1. Apply rules parameterization to all supported `rules_v*.tmpl` versions.
2. Expand parameter extraction beyond root variables/origin hostname/cp_code IDs.
3. Add explicit docs section in `README.md` for scaffold mode usage patterns.
4. Add integration-style test that asserts generated scaffold tree from command path.
