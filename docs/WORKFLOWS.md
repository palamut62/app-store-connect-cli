# Workflows

`asc workflow` lets you define named, multi-step automation sequences in a repo-local file: `.asc/workflow.json`.

This is designed as a single, versioned workflow file that composes existing `asc` commands and normal shell commands.

## Quick Start

1. Create `.asc/workflow.json` in your repo.
2. Validate the file:

```bash
asc workflow validate
```

3. Run a workflow:

```bash
asc workflow run beta
asc workflow run beta BUILD_ID:123456789 GROUP_ID:abcdef
asc workflow run beta --resume beta-20260312T120000Z-deadbeef
```

## Security and Trust Model

Workflows intentionally execute arbitrary shell commands. This is by design: `asc workflow run` is effectively "run a repo-local script".

`asc` does not sandbox workflow execution. A workflow runs with the same permissions as the `asc` process: it can read files, make network requests, and access anything in the environment.

- Treat workflow files like code. Review changes to `.asc/workflow.json` the same way you'd review any code or CI config.
- Do not run workflow files from untrusted sources (e.g., copied from the internet, or from a PR/fork you haven't reviewed).
- In CI, avoid running `asc workflow run` for untrusted pull requests/forks if secrets or write-capable tokens are available in the environment. A safer pattern is to run `asc workflow validate` on PRs and run workflows only on trusted branches.
- Be careful with `--file`: it can point to any path, not just `.asc/workflow.json`.
- Step commands inherit your process environment (`os.Environ()`), so secrets present in the environment are visible to steps.
- Avoid printing secrets in commands; prefer passing secrets as env vars via your CI secret store.
- Treat params as untrusted input. Quote expansions in shell commands to avoid injection issues (e.g., `--app "$APP_ID"` not `--app $APP_ID`).
- Declared step outputs are persisted to a repo-local run-state file. Do not map secret plaintext into `outputs`.
- `asc workflow validate` checks structure and references, not safety of the commands.

## Example `.asc/workflow.json`

Notes:
- The file supports JSONC comments (`//` and `/* */`).
- Output is JSON on stdout; step and hook command output streams to stderr.
- On failures, stdout still remains JSON-only and includes a top-level `error` plus `hooks` results.

```json
{
  "env": {
    "APP_ID": "123456789",
    "VERSION": "1.0.0"
  },
  "before_all": "asc auth status",
  "after_all": "echo workflow_done",
  "error": "echo workflow_failed",
  "workflows": {
    "beta": {
      "description": "Distribute a build to a TestFlight group",
      "env": {
        "GROUP_ID": ""
      },
      "steps": [
        {
          "name": "resolve_build",
          "run": "asc builds latest --app $APP_ID --platform IOS --output json",
          "outputs": {
            "BUILD_ID": "$.id"
          }
        },
        {
          "name": "add_build_to_group",
          "run": "asc builds add-groups --build ${steps.resolve_build.BUILD_ID} --group $GROUP_ID"
        }
      ]
    },
    "release": {
      "description": "Run the full App Store release pipeline (validate, attach, submit)",
      "steps": [
        {
          "workflow": "preflight",
          "with": {
            "NOTE": "running private sub-workflow"
          }
        },
        {
          "name": "release",
          "run": "asc release run --app $APP_ID --version $VERSION --build $BUILD_ID --metadata-dir ./metadata/version/$VERSION --confirm"
        }
      ]
    },
    "preflight": {
      "private": true,
      "description": "Private helper workflow (callable only via workflow steps)",
      "steps": [
        {
          "name": "preflight",
          "run": "echo \"$NOTE\""
        }
      ]
    }
  }
}
```

After running the `release` workflow, monitor submission progress with:

```bash
asc status --app "APP_ID"
```

## Semantics

### Environment Merging

- Entry workflow env: `definition.env` -> `workflow.env` -> CLI params (`KEY:VALUE` / `KEY=VALUE`)
- Sub-workflow env: `sub_workflow.env` provides defaults, caller env overrides, and step `with` overrides win over everything.

### Conditionals

Add `"if": "VAR_NAME"` to a step to skip it when the variable is falsy.

Conditionals check the workflow env/params first, then fall back to `os.Getenv("VAR_NAME")`.
For deterministic behavior (especially in CI), prefer setting conditional variables in the workflow env or passing them as params.

Truthy values (case-insensitive): `1`, `true`, `yes`, `y`, `on`.

### Hooks

Hooks are definition-level commands:
- `before_all` runs once before any steps
- `after_all` runs once after all steps (only if steps succeeded)
- `error` runs on any failure (step failure, hook failure, max call depth, etc.)

Hooks are recorded in the structured JSON output as `hooks.before_all`, `hooks.after_all`, and `hooks.error`.

Successful `before_all` and `after_all` hooks are also persisted in the run-state file so resumed runs do not rerun them unnecessarily.

### Step Outputs

Run steps can declare named outputs extracted from JSON stdout:

```json
{
  "name": "upload",
  "run": "asc publish testflight --app $APP_ID --ipa $IPA --output json",
  "outputs": {
    "BUILD_ID": "$.buildId",
    "PROCESSING_STATE": "$.processingState"
  }
}
```

Rules:
- `outputs` is only valid on `run` steps.
- A step with `outputs` must have a unique, reference-safe `name`.
- Output expressions use a limited JSON path form like `$.field` or `$.nested.field`.
- The command must emit valid JSON on stdout. For `asc` commands, that usually means passing `--output json`.

Later steps can reference persisted outputs directly in shell commands or sub-workflow `with` values:

```json
{
  "name": "distribute",
  "run": "asc builds add-groups --build ${steps.upload.BUILD_ID} --group $GROUP_ID"
}
```

Interpolation is shell-escaped automatically in `run` commands.

### Resume and Run State

Non-dry-run executions persist run state under a repo-local `runs/` directory next to the workflow file.

Each run emits:
- `run_id`
- `run_file`
- `outputs` for declared step outputs

When a run fails after one or more successful persisted steps, the JSON result includes:
- `failed_step`
- `recoverable: true`
- `resume.command`

Resume the run with:

```bash
asc workflow run release --resume beta-20260312T120000Z-deadbeef
```

On resume:
- already-persisted successful steps are reported with `status: "resumed"`
- declared step outputs are restored before later steps run
- the workflow definition, workflow file path, and CLI params must still match the original run
- `--resume` cannot be combined with `--dry-run` or extra `KEY:VALUE` params

### Output Contract

- stdout: JSON-only (`asc workflow run` prints a structured result)
- stderr: step/hook command output, plus dry-run previews

This makes it safe to do:

```bash
asc workflow run beta BUILD_ID:123 GROUP_ID:xyz | jq -e '.status == "ok"'
```
