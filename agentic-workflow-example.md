# Agentic Workflow Example and Lessons

This repository's `repo-assistant-triage` workflow is a manual, task-driven Copilot workflow. It accepts a natural-language `task`, checks out the repository, and can open at most one pull request with its changes.

## Workflow Configuration

The source is `.github/workflows/repo-assistant-triage.md`. Its key configuration is:

```yaml
on:
  workflow_dispatch:
    inputs:
      task:
        description: Task to implement in this repository
        required: true
        type: string

engine:
  id: copilot
  model: gpt-4.1

permissions:
  contents: read
  issues: read
  pull-requests: read
  copilot-requests: write

concurrency:
  job-discriminator: ${{ github.run_id }}

safe-outputs:
  create-pull-request:
    max: 1
```

The task goes in the prompt using `${{ github.event.inputs.task }}`. Keep gh-aw's `aw_context` input for its internal JSON routing data; it is not a free-text prompt field.

## Compile and Run

Compile the Markdown source to the generated GitHub Actions lock file:

```bash
gh aw compile repo-assistant-triage
```

Dispatch a task using the named `task` input:

```bash
gh aw run repo-assistant-triage --raw-field 'task=Create hello.py containing a simple Python program that prints "Hello, world!" to standard output, then open one pull request containing that change.'
```

Do not pass plain text as `aw_context`. To publish uncommitted workflow files before dispatch, `gh aw run` supports `--push`:

```bash
gh aw run repo-assistant-triage --push --raw-field 'task=Create hello.py containing a simple Python program that prints "Hello, world!" to standard output, then open one pull request containing that change.'
```

## Setup Requirements

- **Compiled workflow:** GitHub dispatches `.github/workflows/repo-assistant-triage.lock.yml`; compile and publish it to the repository's default branch.
- **Dispatch authentication:** the local `gh` credential needs permission to trigger Actions workflows. In Codespaces, the injected token may lack that permission; authenticate `gh` with a credential that can dispatch workflows.
- **Copilot authentication:** `copilot-requests: write` lets gh-aw use the built-in Actions token instead of a `COPILOT_GITHUB_TOKEN` secret when the account is eligible for Copilot Actions inference (organization Copilot subscription with centralized billing). Otherwise configure a fine-grained PAT with Copilot Requests access as `COPILOT_GITHUB_TOKEN`.
- **Model availability:** avoid `auto` if it resolves to a model unavailable to the account. This workflow pins `gpt-4.1`; check available catalog models with `gh aw models --refresh-observed=false` and choose one enabled for the Copilot subscription.
- **PR creation policy:** the generated safe-output job already has `contents: write` and `pull-requests: write`. A repository admin must also enable **Settings → Actions → General → Workflow permissions → Allow GitHub Actions to create and approve pull requests**. Workflow YAML permissions cannot override this repository setting.

## `aw_context` Parse Failures

gh-aw generates routing expressions that parse its internal `aw_context` as JSON. Passing text such as `you are an python expert` as `aw_context` can produce `Failed to parse aw_context input as JSON` or a GitHub Actions `fromJSON` template error. Pass that text as the `task` input and leave `aw_context` empty.

This workflow is manual and task-oriented, so its generated lock currently sets the four unused `GH_AW_EXPR_*` routing values to empty strings in both prompt steps. The same compiler version (`v0.88.7`) regenerates the JSON expressions on `gh aw compile`; if recompiling, reapply those four empty-string values in both steps, or upgrade to a compiler version that handles this case. This workaround intentionally does not support routing a specific issue, pull request, discussion, or comment through `aw_context`.

## Reading Failures

- **`workflow ... lock.yml not found`:** compile the Markdown source and publish the lock file to the default branch.
- **HTTP 403 while dispatching:** authenticate the local GitHub CLI with Actions workflow-dispatch permissions.
- **`COPILOT_GITHUB_TOKEN` missing:** enable eligible `copilot-requests: write` billing or configure the fine-grained PAT secret.
- **`fromJSON` / `aw_context` failure:** move natural-language instructions to `task`; keep internal context JSON-only and empty for a direct manual run.
- **PR creation denied:** enable the repository-level Actions PR setting above; the job's YAML permissions alone are insufficient.
- **`requested model is not supported`:** pin a model available to the account instead of using `auto`.
- **Ubuntu runner migration or missing `/tmp/gh-aw/mcp-logs` notices:** these are not the cause of the failures described above; investigate only if the associated step itself fails.
