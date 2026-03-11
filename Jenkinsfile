## Step Functions CI/CD (Folder-based bundles)

This repository deploys AWS Step Functions from source control using GitHub Actions.

Cert deployment is automatic on push/merge to `cert`.
Prod deployment is automatic on push/merge to `prod`.
Manual runs are also available via `workflow_dispatch`.
PRs run validation.

Each workflow is defined as a folder bundle:

1. `definition.asl.json` (**required**) state machine definition
2. `<env>-trigger.json` (optional, preferred) EventBridge trigger config
3. `<env>-input.json` (required only when trigger exists and EventBridge is enabled)
4. `<env>-s3copy.json` (optional, preferred, when S3 copy/update is needed)
5. `<env>-artifacts.json` (optional, preferred, when artifact tar/copy is needed)

---

## Directory structure

```text
.github/workflows/stepfunctions-cicd.yml
scripts/
  deploy_sfn.sh
  delete_sfn.sh
  update_sfn_state_checkpoint.sh
  detect_stepfunctions_changes.sh
  detect_changed_stepfunctions.sh
  render_asl.py
stepfunctions/
  env/
    cert.json
    prod.json
  workflows/
    <step-function-name>/
      definition.asl.json
      cert-trigger.json (optional, preferred)
      prod-trigger.json (optional, preferred)
      trigger.json      (optional fallback)
      cert-input.json   (required only when trigger is enabled)
      prod-input.json   (required only when trigger is enabled)
      input.json        (fallback for event input)
      cert-s3copy.json  (optional, preferred)
      prod-s3copy.json  (optional, preferred)
      s3copy.json       (optional fallback)
      cert-artifacts.json (optional, preferred)
      prod-artifacts.json (optional, preferred)
      artifacts.json      (optional fallback)
```

The workflow ID is the folder name under `stepfunctions/workflows/`.

---

## Workflow bundle format

### `definition.asl.json`

- Standard ASL definition.
- Supports `${KEY}` placeholders rendered from `stepfunctions/env/<environment>.json`.

### `<env>-trigger.json` / `trigger.json`

Controls EventBridge trigger behavior.

Example:

```json
{
  "comment": "Update this value to force trigger reapply",
  "stateMachine": {
    "type": "STANDARD",
    "tags": {
      "app": "orders",
      "managed-by": "github-actions"
    }
  },
  "eventBridge": {
    "enabled": true,
    "description": "Run orders workflow every 30 minutes",
    "scheduleExpression": "rate(30 minutes)",
    "state": "ENABLED",
    "targetId": "run-orders-workflow",
    "roleArn": "arn:aws:iam::123456789012:role/eventbridge-start-sfn"
  }
}
```

S3 copy notes:
- `eventBridge.enabled` defaults to `true`.
- `eventBridge.scheduleExpression` **or** `eventBridge.eventPattern` must be set (exactly one) when enabled.
- `eventBridge.roleArn` is required unless `EVENTBRIDGE_INVOKE_ROLE_ARN` env var is provided.
- `comment` is optional and included in trigger state hash; changing it forces trigger reapply.
- Trigger config file is only for EventBridge/state machine settings.
- Deploy/validate look up trigger config in this order:
  1. `<env>-trigger.json` (for example `cert-trigger.json`, `prod-trigger.json`)
  2. `trigger.json` (fallback)
- If neither trigger file exists, EventBridge setup is skipped.
- Event input is read only when EventBridge is enabled.
- S3 copy settings are read in this order:
  1. `<env>-s3copy.json` (for example `cert-s3copy.json`, `prod-s3copy.json`)
  2. `s3copy.json` (fallback)

### `<env>-s3copy.json` / `s3copy.json` (optional)

Runs an S3 copy step during deployment.

```json
{
  "comment": "Update this value to force S3 copy rerun",
  "enabled": true,
  "mode": "files",
  "sourceUri": "s3://source-bucket/prefix/",
  "destinationUri": "s3://destination-bucket/prefix/",
  "files": [
    "config/file1.json",
    "input/file2.csv"
  ],
  "backupExisting": true,
  "backupTimestampFormat": "%Y%m%d%H%M%S"
}
```

### `<env>-artifacts.json` / `artifacts.json` (optional)

Runs artifact build/copy steps during deployment.  
Supports multiple tasks using a `tasks` array.

```json
{
  "tasks": [
    {
      "comment": "Build tar from batch.sql and upload",
      "enabled": true,
      "mode": "tar",
      "sourceSqlPath": "artifacts/x/xa/xy/batch.sql",
      "destinationUri": "s3://my-bucket/path/",
      "backupExisting": true,
      "backupTimestampFormat": "%Y%m%d%H%M%S"
    },
    {
      "comment": "Copy batch.sql directly",
      "enabled": true,
      "mode": "copy",
      "sourceFilePath": "artifacts/x/xa/xy/batch.sql",
      "destinationUri": "s3://my-bucket/path/batch.sql",
      "backupExisting": true,
      "backupTimestampFormat": "%Y%m%d%H%M%S"
    }
  ]
}
```

Artifact notes:
- File lookup order is:
  1. `<env>-artifacts.json`
  2. `artifacts.json` (fallback)
- If no artifacts file exists, artifact flow is skipped.
- `mode=tar` creates `<parent-dir-of-sourceSqlPath>.tar.gz`.
  - Example: `artifacts/x/xa/xy/batch.sql` -> `xy.tar.gz`
  - Equivalent tar command run from source directory: `tar -czf xy.tar.gz batch.sql`
- `mode=copy` uploads the source file directly.
- `destinationUri` must be `s3://...`
  - If URI ends with `/`, filename is appended automatically.
- Existing destination objects are backed up first when `backupExisting=true`.

Dummy example included in this repository:
- Source file: `artifacts/x/xa/xy/batch.sql`
- Workflow config: `stepfunctions/workflows/dummy-ecs-run-task/cert-artifacts.json`
  and `stepfunctions/workflows/dummy-ecs-run-task/prod-artifacts.json`
- Task behavior:
  - `mode=tar` creates `xy.tar.gz` from `batch.sql` and uploads to S3
  - `mode=copy` uploads `batch.sql` directly to S3

Notes:
- Use per-workflow variable names in env JSON for clarity, for example:
  - `DUMMY_ECS_RUN_TASK_COPY_SOURCE_URI`
  - `DUMMY_ECS_RUN_TASK_COPY_DESTINATION_URI`
- `comment` is optional and included in S3 copy state hash; changing it forces S3 copy rerun.
- `mode=files` copies only the listed relative paths from source prefix to destination prefix.
- When `backupExisting=true`, if a destination object already exists, it is backed up first as:
  - `<original-key>.<UTC timestamp>` (timestamp format configurable by `backupTimestampFormat`)
- `backupExisting=true` is supported with `mode=files`.
- To run multiple copy operations in one workflow deploy, use `tasks`:

```json
{
  "tasks": [
    {
      "enabled": true,
      "mode": "files",
      "sourceUri": "s3://source-a/prefix/",
      "destinationUri": "s3://dest-a/prefix/",
      "files": ["a.txt", "b.txt"],
      "backupExisting": true
    },
    {
      "enabled": true,
      "mode": "files",
      "sourceUri": "s3://source-b/prefix/",
      "destinationUri": "s3://dest-b/prefix/",
      "files": ["c.txt"]
    }
  ]
}
```

### `<env>-input.json` / `input.json`

- JSON payload passed to EventBridge target as the `Input` for `StartExecution`.
- Supports `${KEY}` placeholders rendered from env JSON.
- Deploy/validate look up input file in this order:
  1. `<env>-input.json` (for example `cert-input.json`, `prod-input.json`)
  2. `input.json` (fallback)

---

## Environment files

`stepfunctions/env/cert.json` and `stepfunctions/env/prod.json` provide template values
for definition, trigger, and input rendering.

---

## Naming templates

Deploy/delete scripts use deterministic names so delete can work even when a workflow folder is removed.

- State machine name template:
  - `SFN_STATE_MACHINE_NAME_TEMPLATE` (default: `sfn-__ENV__-__WORKFLOW__`)
- EventBridge rule name template:
  - `EVENTBRIDGE_RULE_NAME_TEMPLATE` (default: `__ENV__-__WORKFLOW__-trigger`)
- Event bus:
  - `EVENTBRIDGE_EVENT_BUS_NAME` (default: `default`)

`__WORKFLOW__` is the workflow folder name; `__ENV__` is the deployment environment.

---

## CI/CD behavior

### Validation

The workflow validates:
- all env JSON files
- each workflow bundle JSON
- rendered output for definition/trigger/input/s3copy/artifacts (using both `cert` and `prod` env files)
- trigger constraints (schedule vs event pattern)
- S3 URI format when S3 copy is enabled
- artifacts task format and source file paths (when artifacts config exists)

### Change detection

`scripts/detect_stepfunctions_changes.sh` emits:
- `DEPLOY:<workflow-dir>`
- `DELETE:<workflow-id>`

Rules:
- deploy changed workflow folders only
- env-only changes do not trigger deployment selection
- detect deletes/renames via git diff status

### Deploy

`scripts/deploy_sfn.sh <cert|prod> <workflow-dir>`:
- create/update Step Function
- if trigger config exists and EventBridge is enabled: create/update EventBridge rule + target
- if EventBridge is enabled: pass rendered env-scoped input payload to EventBridge target
- optionally run S3 copy (`files`, `sync`, or `cp`)
- optionally run artifacts tasks (`tar` or `copy`)
- update a persistent state file in S3 with separate hashes for:
  - `stepFunction`
  - `trigger`
  - `s3Copy`
  - `artifacts`

### State file tracking

State is stored in S3 (one file per environment) and used to detect whether each sub-part
needs rerun. Configure these secrets:

- `SFN_STATE_FILE_S3_URI_CERT` (example: `s3://my-bucket/stepfunctions/state/cert.json`)
- `SFN_STATE_FILE_S3_URI_PROD` (example: `s3://my-bucket/stepfunctions/state/prod.json`)

During deploy:
- if Step Function hash unchanged, Step Function update is skipped
- if trigger hash unchanged, EventBridge upsert is skipped
- if S3 copy hash unchanged, S3 copy tasks are skipped
- if artifacts hash unchanged, artifacts tasks are skipped
- changing `comment` in trigger config (`<env>-trigger.json`/`trigger.json`), S3 copy config (`<env>-s3copy.json`/`s3copy.json`), or artifacts config (`<env>-artifacts.json`/`artifacts.json`) changes hash and forces rerun
- after successful job completion, `lastSuccessfulCommit` is updated in state file

Manual run (`workflow_dispatch`, `mode=changed`) uses `lastSuccessfulCommit` as diff base.
That allows one run to catch all changes merged since the last successful deployment.
If no checkpoint exists yet, it falls back to `HEAD^`.

### Delete

`scripts/delete_sfn.sh <cert|prod> <workflow-id>`:
- delete Step Function by deterministic name
- remove EventBridge targets
- delete EventBridge rule
- remove workflow entry from the environment state file in S3

---

## Required GitHub Secrets

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION`
- `SFN_EXECUTION_ROLE_ARN_CERT`
- `SFN_EXECUTION_ROLE_ARN_PROD`
- `SFN_STATE_FILE_S3_URI_CERT` (required for state tracking)
- `SFN_STATE_FILE_S3_URI_PROD` (required for state tracking)

---

## Adding a new Step Function job

Create a folder:

`stepfunctions/workflows/<step-function-name>/`

Add:
- `definition.asl.json`
- `<env>-trigger.json` (optional, preferred) or `trigger.json` fallback
- `<env>-input.json` (required only if trigger is enabled) or `input.json` fallback
- `<env>-s3copy.json` (optional, preferred) or `s3copy.json` fallback
- `<env>-artifacts.json` (optional, preferred) or `artifacts.json` fallback

Commit and raise a PR. After merge, run deployment based on your workflow trigger strategy.
