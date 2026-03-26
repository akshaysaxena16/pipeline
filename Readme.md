
## Developer guide: add a new Step Function

This walkthrough focuses on **state machine definition**, **environment placeholders** in `stepfunctions/env/cert.json` and `prod.json`, and **artifact build/upload** via `<env>-artifacts.json` during deploy. EventBridge triggers, EventBridge input payloads, and deploy-time S3 copy (`<env>-s3copy.json`) are covered under [Workflow bundle format](#workflow-bundle-format) above.

### 1. Create the workflow bundle folder

The **workflow ID** is the folder name under `stepfunctions/workflows/` (for example `my-batch-export`).

Minimum to deploy a state machine:

```text
stepfunctions/workflows/my-batch-export/
  definition.asl.json           # required
```

Add **`<env>-artifacts.json`** (preferred) or `artifacts.json` when you want the deploy job to build a tarball and/or copy files from this repo to S3. Example layout:

```text
stepfunctions/workflows/my-batch-export/
  definition.asl.json
  cert-artifacts.json
  prod-artifacts.json
```

Put files that should be archived or uploaded under a path such as `artifacts/...` in the repository (see **`<env>-artifacts.json` / `artifacts.json`** under [Workflow bundle format](#workflow-bundle-format)).

Cert deploy runs on push/merge to `cert`; prod runs on `prod`. Open a PR for validation on other branches.

### 2. Environment variables in `cert.json` / `prod.json`

`stepfunctions/env/cert.json` and `stepfunctions/env/prod.json` supply values for `${KEY}` substitution. `scripts/render_asl.py` replaces each token in:

- `definition.asl.json`
- and any optional bundle JSON that uses placeholders (including `<env>-artifacts.json` `destinationUri` values)

**Scalars** (strings, numbers) replace as text. **Arrays/objects** (for example subnet lists) replace as raw JSON so they embed correctly in ASL.

**Example** — execution settings plus an **artifacts upload prefix** used in `<env>-artifacts.json`:

`stepfunctions/env/cert.json`:

```json
{
  "ENV": "cert",
  "CLUSTER_ARN": "arn:aws:ecs:us-east-1:111111111111:cluster/my-cert-cluster",
  "TASK_DEFINITION_ARN": "arn:aws:ecs:us-east-1:111111111111:task-definition/my-batch:1",
  "SUBNETS_JSON": ["subnet-aaa", "subnet-bbb"],
  "SECURITY_GROUPS_JSON": ["sg-ccc"],
  "ASSIGN_PUBLIC_IP": "DISABLED",
  "MY_BATCH_EXPORT_ARTIFACTS_DESTINATION_URI": "s3://my-cert-bucket/applications/my-batch-export/artifacts/"
}
```

`stepfunctions/env/prod.json` — same keys with production values:

```json
{
  "ENV": "prod",
  "CLUSTER_ARN": "arn:aws:ecs:us-east-1:222222222222:cluster/my-prod-cluster",
  "TASK_DEFINITION_ARN": "arn:aws:ecs:us-east-1:222222222222:task-definition/my-batch:1",
  "SUBNETS_JSON": ["subnet-xxx", "subnet-yyy"],
  "SECURITY_GROUPS_JSON": ["sg-zzz"],
  "ASSIGN_PUBLIC_IP": "DISABLED",
  "MY_BATCH_EXPORT_ARTIFACTS_DESTINATION_URI": "s3://my-prod-bucket/applications/my-batch-export/artifacts/"
}
```

Use a **workflow-specific prefix** on keys (here `MY_BATCH_EXPORT_*`) so workflows do not share the same env names. Every `${...}` referenced in templates must exist in **both** env files for CI rendering to pass.

### 3. ASL definition and placeholders

Reference env keys in `definition.asl.json` as `"${KEY}"` or embed list/object tokens without quotes where JSON allows (see existing ECS examples in this repo).

Example fragment passing an artifact location into a container after deploy has uploaded it (your container reads `ARTIFACT_S3_URI` or a prefix you agree on):

```json
{
  "Overrides": {
    "ContainerOverrides": [
      {
        "Name": "app",
        "Environment": [
          { "Name": "ENV", "Value": "${ENV}" },
          { "Name": "ARTIFACT_S3_PREFIX", "Value": "${MY_BATCH_EXPORT_ARTIFACTS_DESTINATION_URI}" }
        ]
      }
    ]
  }
}
```

**Local render check:**

```bash
python3 scripts/render_asl.py stepfunctions/env/cert.json \
  stepfunctions/workflows/my-batch-export/definition.asl.json /tmp/rendered.json
jq . /tmp/rendered.json
```

### 4. Artifacts: `<env>-artifacts.json` (tar and copy to S3)

During `scripts/deploy_sfn.sh`, optional **artifacts tasks** can:

- **`mode=tar`** — from a source file path in the repo, build a `.tar.gz` named from the **parent folder** of that file (see **Artifact notes** under [Workflow bundle format](#workflow-bundle-format)), then `aws s3 cp` to `destinationUri`.
- **`mode=copy`** — upload a single repo file to `destinationUri`.

`destinationUri` must render to `s3://...`. It can be a **`${KEY}`** from env JSON (recommended), for example `"${MY_BATCH_EXPORT_ARTIFACTS_DESTINATION_URI}"`. If the URI ends with `/`, the object key (file name) is chosen automatically.

Use a **`tasks` array** for multiple steps. Top-level **`comment`** (and task comments) participate in deploy state hashing — change them when you want to force artifacts to run again without changing file content.

**Minimal example** (mirroring the idea of `stepfunctions/workflows/dummy-ecs-run-task/cert-artifacts.json`):

```json
{
  "comment": "My workflow artifacts",
  "tasks": [
    {
      "enabled": true,
      "mode": "tar",
      "sourceSqlPath": "artifacts/my-batch/export.sql",
      "destinationUri": "${MY_BATCH_EXPORT_ARTIFACTS_DESTINATION_URI}",
      "backupExisting": true,
      "backupTimestampFormat": "%Y%m%d%H%M%S"
    },
    {
      "enabled": true,
      "mode": "copy",
      "sourceFilePath": "artifacts/my-batch/export.sql",
      "destinationUri": "${MY_BATCH_EXPORT_ARTIFACTS_DESTINATION_URI}",
      "backupExisting": true,
      "backupTimestampFormat": "%Y%m%d%H%M%S"
    }
  ]
}
```

Check in **`sourceSqlPath` / `sourceFilePath`** files under the repo root; validation and deploy resolve them as local paths relative to the checkout.

### Dummy example: `dummy-ecs-run-task` (where files live, what gets uploaded)

This repository includes a small **dummy Step Functions** bundle plus a sample artifact file so you can see paths and **resulting object names** on S3.

| What | Location / value |
|------|------------------|
| **Workflow folder (Step Function ID)** | `stepfunctions/workflows/dummy-ecs-run-task/` |
| **State machine definition** | `stepfunctions/workflows/dummy-ecs-run-task/definition.asl.json` |
| **Artifact source file in git** | `artifacts/x/xa/xy/batch.sql` — SQL file checked into the repo (dummy content for demos / CI) |
| **Deploy artifact config** | `cert-artifacts.json` and `prod-artifacts.json` in that workflow folder |

The dummy **`cert-artifacts.json`** / **`prod-artifacts.json`** define two tasks against the same source file:

1. **`mode=tar`** with `"sourceSqlPath": "artifacts/x/xa/xy/batch.sql"`  
   - The deploy script `cd`s into the **parent directory** of that file (`artifacts/x/xa/xy/`) and runs `tar` on `batch.sql`.  
   - **Archive file name** = **parent folder name** + `.tar.gz` → **`xy.tar.gz`**.  
   - So the tarball is always named after the **last path segment** that contains the file (`xy`), not after `batch.sql`.

2. **`mode=copy`** with `"sourceFilePath": "artifacts/x/xa/xy/batch.sql"`  
   - Uploads the file as-is. **Object name** = **basename of the path** → **`batch.sql`**.

**Destination prefix** comes from env JSON after `${...}` render:

- Key: **`DUMMY_ECS_RUN_TASK_ARTIFACTS_DESTINATION_URI`**
- Example (cert): **`s3://amplify-amplifybackend-dev-15f2c-deployment/test/artifacts/`** (see `stepfunctions/env/cert.json`)

Because `destinationUri` ends with `/`, the deploy step **appends** the object name automatically. After a successful deploy you get two objects next to each other, for example:

```text
s3://<bucket>/test/artifacts/xy.tar.gz    # tar archive (contains batch.sql inside)
s3://<bucket>/test/artifacts/batch.sql    # plain file copy
```

Your ASL or ECS task can pass `DUMMY_ECS_RUN_TASK_ARTIFACTS_DESTINATION_URI` (or a full object URI you build in env) so the workload knows where **`xy.tar.gz`** / **`batch.sql`** were published.

**End-to-end flow:** add files under `artifacts/...`, add `cert-artifacts.json` and `prod-artifacts.json` with per-env `destinationUri` placeholders, add matching keys to `cert.json` / `prod.json`, reference the same S3 prefix or object in your ASL/task if the workload must read what was published.
