# AGENTS.md

## Project Purpose

This Infrastructure-as-Code repository deploys Azure resources for OpenShift bare-metal-on-Azure scenarios using the `azi` CLI workflow.

The deployment model is:
- `azi` CLI orchestration
- `go-task` Taskfile automation
- Azure CLI (`az`) deployment calls
- Azure Verified Modules (AVM) from Bicep public registry
- **Configuration only through `.bicepparam` files**

## Mandatory Deployment Conventions

1. Use `azi` as the primary interface for deployment lifecycle operations.
2. Do not introduce standalone `.bicep` templates in this repo.
3. Use only `.bicepparam` files for resource/module configuration.
4. Keep deployment definitions in `deploy.yaml` manifest format expected by `azi`.
5. Keep scope-specific resource params under existing folders:
   - `resources/**`
   - `network/**`
   - `compute/**`

## Current Repo Structure

- `taskfile.yaml` (repo root) — intended entrypoint for project tasks
- `deploy.yaml` — deployment manifest consumed by `azi`
- `tags.yaml` — optional tagging metadata
- `resources/resource-group/*.bicepparam`
- `network/virtual-network/*.bicepparam`
- `compute/virtual-machine/*.bicepparam`
- `docs/guide.md` — architecture and operations notes

## `azi` CLI Research Summary

From direct CLI help inspection:

- `azi --help`
  - Usage: `azi [task] [flags]` or `azi [command]`
  - Commands include: `bicep`, `resources` (`res` alias), `install`, `update`, `changelog`
- `azi bicep --help`
  - Commands include: `describe`, `validate`, `plan`, `apply`, `fix`, `destroy`
- `azi resources avm --help`
  - Commands include: `init`, `list`, `new`, `update`

Global flags available across commands include:
- `--dry`
- `--force`
- `--parallel`
- `--silent`
- `--summary`
- `-v, --verbose`
- `-y, --yes`

### Supported workflow commands for this repo

Use these commands as the supported operational path:
- `azi bicep describe`
- `azi bicep validate`
- `azi bicep plan`
- `azi bicep apply`

> `destroy` is intentionally excluded from recommended workflow in this repo because it only removes Azure deployment metadata at scope and does not reliably remove underlying Azure resources.

## How to Add New Resource Configuration

1. Initialize provider data when needed:
   - `azi resources avm init`
2. Discover available modules:
   - `azi resources avm list`
3. Create new AVM-backed resource config in target directory:
   - `azi resources avm new`
4. Ensure output is a `.bicepparam` file and keep it under an existing domain folder (`resources/`, `network/`, `compute/`) or a clearly scoped new folder.
5. Fill all required placeholder params in the generated `.bicepparam` file.
6. Add the new parameter file path into a `deploy.yaml` deployment entry under `resources`.
7. Run lifecycle checks in order:
   - `azi bicep describe`
   - `azi bicep validate`
   - `azi bicep plan`
   - `azi bicep apply`

## Manifest Expectations (`deploy.yaml` and `tags.yaml`)

`azi` evaluates manifests from the current working directory context.

### Working-directory behavior

- `azi bicep` defaults to `deploy.yaml` in the current directory.
- You can keep multiple `deploy.yaml` manifests across the repo and execute commands from the folder that owns the intended manifest.
- The same structural pattern applies to `tags.yaml` when using tag workflows.

This enables logical, layered layouts such as:
- `./deploy.yaml`
- `./tenants/Contoso/subscriptions/dev/deploy.yaml`
- `./tenants/Contoso/subscriptions/test/deploy.yaml`
- `./tenants/Contoso/subscriptions/prod/deploy.yaml`
- `./tenants/Contoso/subscriptions/dev/rg-aks/deploy.yaml`
- `./tenants/Contoso/subscriptions/dev/rg-openshift/deploy.yaml`
- `./environments/dev/rg-openshift/deploy.yaml`
- `./rg-openshift/deploy.yaml`

### `deploy.yaml` schema and conditional requirements

`deploy.yaml` is parsed as a YAML array of deployment objects.

Supported keys per deployment object:
- `deployment_type` (`group` or `subscription`)
- `scope`
- `desc`
- `location`
- `subscription`
- `resources` (array of `.bicepparam` paths)
- `parameters` (array of direct parameter fragments)
- `require_confirmation` (bool)
- `template_file` (currently unsupported; must be empty)
- `template_uri` (currently unsupported; must be empty)
- `template_spec` (currently unsupported; must be empty)

Conditional requirements from task preconditions:
- `deployment_type: group`
  - Required: `scope`
  - `desc` is required by validate/plan tasks and strongly recommended for apply naming.
- `deployment_type: subscription`
  - Required: `scope`, `location`
  - `desc` is required by validate/plan tasks and strongly recommended for apply naming.

Execution notes:
- Entries with empty `resources` are skipped by status guards.
- `resources` and `parameters` are emitted as repeated Azure CLI `--parameters` arguments.
- Resource config references should be explicit `.bicepparam` paths resolved from the active manifest context.

### `tags.yaml` schema and conditional requirements

`tags.yaml` is optional and processed by `azi bicep apply` via internal `apply:tags` task.

`tags.yaml` is parsed as a YAML array of tag-operation objects.

Supported keys per tag object:
- `resource_id` (string)
- `desc` (string)
- `tags` (array of `key=value` strings)
- `auto_tags` (bool, default `true`)
- `action` (string, default `update`)
- `operation` (string, default `merge`)
- `require_confirmation` (bool, default `true`)

Allowed values:
- `action`: `add-value`, `remove-value`, `create`, `delete`, `update`
- `operation`: `delete`, `merge`, `replace`

Conditional requirements from task preconditions:
- `resource_id` is required unless `action` is `create`, `add-value`, or `remove-value`.
- At least one of these must be provided:
  - non-empty `tags`, or
  - `auto_tags: true`
- For `action: add-value` and `action: remove-value`, exactly one tag is allowed.
- For `action: delete`, `resource_id` is required; `require_confirmation: false` adds `--yes`.

Tagging should reference Azure targets by `resource_id` for deterministic behavior.

### Behavior from upstream taskfiles

- `describe` renders manifest entries.
- `validate` calls `az deployment ... validate`.
- `plan` calls `az deployment ... what-if`.
- `apply` calls `az deployment ... create`.
- `resources` and `parameters` are passed as Azure CLI `--parameters` arguments.
- `template_file`, `template_uri`, and `template_spec` paths are currently precondition-blocked in bicep task libs.

## Implementation Rules for Agents

1. Preserve existing deployment workflow; do not bypass `azi` conventions.
2. Prefer minimal and reversible changes.
3. Keep edits scoped to requested behavior.
4. If introducing new infra resources, add AVM-backed `.bicepparam` files only.
5. Do not add new dependencies unless explicitly requested.

## Validation Expectations

When making infra changes:
1. Run `azi bicep describe` to verify manifest parsing.
2. Run `azi bicep validate` before apply.
3. Run `azi bicep plan` and inspect for unintended changes.
4. Run `azi bicep apply` only after validation/plan look correct.

## Security and Safety

- Do not hardcode secrets in `.bicepparam` files.
- Prefer secure parameterization patterns supported by Azure/Bicep parameters.
- Keep network exposure minimal and explicit.

## Notes for Future Work

- Root-level `taskfile.yaml`, `deploy.yaml`, and `tags.yaml` are currently empty placeholders and need project-specific content to make end-to-end `azi` execution operational.
- Existing `.bicepparam` files contain required-parameter placeholders and must be completed prior to successful validation/apply.
