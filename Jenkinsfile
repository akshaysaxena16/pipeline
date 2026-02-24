#!/usr/bin/env bash
set -euo pipefail

# Deploy a Step Functions workflow bundle and its EventBridge trigger.
#
# Requirements:
# - AWS credentials already configured (e.g. via GitHub OIDC).
# - Env vars:
#     AWS_REGION
#     SFN_EXECUTION_ROLE_ARN   (role for the state machine to assume)
# - Optional:
#     EVENTBRIDGE_INVOKE_ROLE_ARN (fallback role for EventBridge target)
#     SFN_STATE_MACHINE_NAME_TEMPLATE (default: sfn-__ENV__-__WORKFLOW__)
#     EVENTBRIDGE_RULE_NAME_TEMPLATE  (default: __ENV__-__WORKFLOW__-trigger)
#     EVENTBRIDGE_EVENT_BUS_NAME      (default: default)
#     SFN_STATE_FILE_S3_URI           (required; s3://... state file path)
#
# Usage:
#   scripts/deploy_sfn.sh <cert|prod> <workflow-directory>
#
# Workflow bundle layout:
#   stepfunctions/workflows/<workflow-id>/
#     definition.asl.json
#     <env>-trigger.json (optional, preferred)
#     trigger.json       (optional, fallback)
#     <env>-input.json   (required when trigger exists and EventBridge is enabled)
#     input.json         (fallback for EventBridge input)
#     <env>-s3copy.json  (optional, preferred)
#     s3copy.json        (optional, fallback)

ENV_NAME="${1:-}"
WORKFLOW_DIR="${2:-}"

if [[ -z "$ENV_NAME" || -z "$WORKFLOW_DIR" ]]; then
  echo "Usage: $0 <cert|prod> <workflow-directory>" >&2
  exit 2
fi

if [[ -z "${AWS_REGION:-}" ]]; then
  echo "Missing AWS_REGION env var" >&2
  exit 2
fi

if [[ -z "${SFN_EXECUTION_ROLE_ARN:-}" ]]; then
  echo "Missing SFN_EXECUTION_ROLE_ARN env var" >&2
  exit 2
fi

resolve_env_scoped_json() {
  local workflow_dir="$1"
  local env_name="$2"
  local base_name="$3"

  local env_scoped_path="${workflow_dir}/${env_name}-${base_name}.json"
  local shared_path="${workflow_dir}/${base_name}.json"

  if [[ -f "$env_scoped_path" ]]; then
    printf '%s\n' "$env_scoped_path"
    return 0
  fi

  if [[ -f "$shared_path" ]]; then
    printf '%s\n' "$shared_path"
    return 0
  fi

  printf '\n'
}

ENV_FILE="stepfunctions/env/${ENV_NAME}.json"
DEF_PATH="${WORKFLOW_DIR}/definition.asl.json"
TRIGGER_PATH="$(resolve_env_scoped_json "$WORKFLOW_DIR" "$ENV_NAME" "trigger")"
INPUT_PATH="$(resolve_env_scoped_json "$WORKFLOW_DIR" "$ENV_NAME" "input")"
S3_COPY_FILE_PATH="$(resolve_env_scoped_json "$WORKFLOW_DIR" "$ENV_NAME" "s3copy")"

if [[ ! -f "$ENV_FILE" ]]; then
  echo "Missing env file: $ENV_FILE" >&2
  exit 2
fi
if [[ ! -f "$DEF_PATH" ]]; then
  echo "Missing definition file: $DEF_PATH" >&2
  exit 2
fi

WF_ID="$(basename "$WORKFLOW_DIR")"

TMP_DIR="$(mktemp -d)"
trap 'rm -rf "$TMP_DIR"' EXIT

RENDERED_DEF="${TMP_DIR}/definition.rendered.json"
RENDERED_TRIGGER="${TMP_DIR}/trigger.rendered.json"
RENDERED_INPUT="${TMP_DIR}/input.rendered.json"
RENDERED_S3_COPY="${TMP_DIR}/s3copy.rendered.json"

python3 scripts/render_asl.py "$ENV_FILE" "$DEF_PATH" "$RENDERED_DEF"

TRIGGER_EXISTS="false"
if [[ -n "$TRIGGER_PATH" ]]; then
  TRIGGER_EXISTS="true"
  python3 scripts/render_asl.py "$ENV_FILE" "$TRIGGER_PATH" "$RENDERED_TRIGGER"
fi

if [[ -n "$S3_COPY_FILE_PATH" ]]; then
  python3 scripts/render_asl.py "$ENV_FILE" "$S3_COPY_FILE_PATH" "$RENDERED_S3_COPY"
fi

parse_s3_uri() {
  local uri="$1"
  local without_scheme bucket key
  if [[ "$uri" != s3://* ]]; then
    return 1
  fi
  without_scheme="${uri#s3://}"
  bucket="${without_scheme%%/*}"
  key=""
  if [[ "$without_scheme" == */* ]]; then
    key="${without_scheme#*/}"
  fi
  printf '%s\n%s\n' "$bucket" "$key"
}

join_s3_key() {
  local prefix="$1"
  local file="$2"
  prefix="${prefix#/}"
  prefix="${prefix%/}"
  file="${file#/}"
  if [[ -z "$prefix" ]]; then
    printf '%s\n' "$file"
  elif [[ -z "$file" ]]; then
    printf '%s\n' "$prefix"
  else
    printf '%s/%s\n' "$prefix" "$file"
  fi
}

sha256_of_file() {
  local path="$1"
  sha256sum "$path" | awk '{print $1}'
}

sha256_of_payload() {
  sha256sum | awk '{print $1}'
}

delete_eventbridge_rule_if_exists() {
  local rule_name="$1"
  local event_bus="$2"

  local rule_desc_json rule_arn
  rule_desc_json="$(aws events describe-rule \
    --region "$AWS_REGION" \
    --name "$rule_name" \
    --event-bus-name "$event_bus" \
    --output json 2>/dev/null || echo '{}')"
  rule_arn="$(echo "$rule_desc_json" | jq -r '.Arn // empty')"

  if [[ -z "${rule_arn//[[:space:]]/}" || "$rule_arn" != arn:* ]]; then
    return 0
  fi

  local targets_json
  targets_json="$(aws events list-targets-by-rule \
    --region "$AWS_REGION" \
    --event-bus-name "$event_bus" \
    --rule "$rule_name" \
    --output json 2>/dev/null || echo '{}')"

  mapfile -t target_ids < <(echo "$targets_json" | jq -r '.Targets[]?.Id // empty')
  if (( ${#target_ids[@]} > 0 )); then
    aws events remove-targets \
      --region "$AWS_REGION" \
      --event-bus-name "$event_bus" \
      --rule "$rule_name" \
      --ids "${target_ids[@]}" \
      --force \
      --output json >/dev/null || true
  fi

  aws events delete-rule \
    --region "$AWS_REGION" \
    --event-bus-name "$event_bus" \
    --name "$rule_name" \
    --output json >/dev/null
}

STATE_FILE_S3_URI="${SFN_STATE_FILE_S3_URI:-}"
if [[ -z "$STATE_FILE_S3_URI" ]]; then
  echo "Missing SFN_STATE_FILE_S3_URI env var (expected s3://bucket/path/state.json)" >&2
  exit 2
fi
if [[ "$STATE_FILE_S3_URI" != s3://* ]]; then
  echo "Invalid SFN_STATE_FILE_S3_URI (must start with s3://): $STATE_FILE_S3_URI" >&2
  exit 2
fi

STATE_FILE_LOCAL="${TMP_DIR}/deploy-state.json"
if aws s3 cp "$STATE_FILE_S3_URI" "$STATE_FILE_LOCAL" --region "$AWS_REGION" >/dev/null 2>&1; then
  jq -e . "$STATE_FILE_LOCAL" >/dev/null
else
  printf '{"version":1,"environment":"%s","workflows":{}}\n' "$ENV_NAME" > "$STATE_FILE_LOCAL"
fi

PREV_STEPFN_HASH="$(jq -r --arg id "$WF_ID" '.workflows[$id].stepFunction.hash // empty' "$STATE_FILE_LOCAL")"
PREV_TRIGGER_HASH="$(jq -r --arg id "$WF_ID" '.workflows[$id].trigger.hash // empty' "$STATE_FILE_LOCAL")"
PREV_S3_HASH="$(jq -r --arg id "$WF_ID" '.workflows[$id].s3Copy.hash // empty' "$STATE_FILE_LOCAL")"

STATE_MACHINE_NAME_TEMPLATE="${SFN_STATE_MACHINE_NAME_TEMPLATE:-sfn-__ENV__-__WORKFLOW__}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME_TEMPLATE//__ENV__/$ENV_NAME}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME//__WORKFLOW__/$WF_ID}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME//\\\$\{ENV\}/$ENV_NAME}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME//\$\{ENV\}/$ENV_NAME}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME//\\\$\{WORKFLOW\}/$WF_ID}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME//\$\{WORKFLOW\}/$WF_ID}"

TYPE="STANDARD"
STATE_MACHINE_TAGS='{}'
if [[ "$TRIGGER_EXISTS" == "true" ]]; then
  TYPE="$(jq -r '.stateMachine.type // "STANDARD"' "$RENDERED_TRIGGER")"
  STATE_MACHINE_TAGS="$(jq -c '.stateMachine.tags // {}' "$RENDERED_TRIGGER")"
fi

TAGS_JSON="$(jq -cn --arg env "$ENV_NAME" --arg id "$WF_ID" --argjson state_tags "$STATE_MACHINE_TAGS" '
  ($state_tags + {"managed-by": "github-actions", env: $env, workflow: $id})
')"

echo "==> Env: $ENV_NAME"
echo "==> Workflow: $WF_ID"
echo "==> State machine: $STATE_MACHINE_NAME"

# Resolve existing state machine ARN robustly.
# Avoid --output text + "None" checks; paginated output can yield multiple "None" lines.
EXISTING_ARN="$(
  aws stepfunctions list-state-machines \
    --region "$AWS_REGION" \
    --output json \
  | jq -r --arg name "$STATE_MACHINE_NAME" '
      ([.stateMachines[]? | select(.name == $name) | .stateMachineArn] | first) // ""
    '
)"

HAS_EXISTING_ARN="false"
if [[ -n "${EXISTING_ARN//[[:space:]]/}" && "$EXISTING_ARN" == arn:* ]]; then
  HAS_EXISTING_ARN="true"
fi

TRACING_ARGS=(--tracing-configuration "enabled=true")

TAGS_ARG="$(echo "$TAGS_JSON" | jq -c 'to_entries | map({key: .key, value: (.value|tostring)})')"

STEPFN_HASH="$(
  {
    sha256_of_file "$RENDERED_DEF"
    printf '%s\n' "$TYPE"
    printf '%s\n' "$SFN_EXECUTION_ROLE_ARN"
    printf '%s\n' "$TAGS_JSON"
  } | sha256_of_payload
)"

RUN_STEPFN="true"
if [[ -n "$PREV_STEPFN_HASH" && "$PREV_STEPFN_HASH" == "$STEPFN_HASH" && "$HAS_EXISTING_ARN" == "true" ]]; then
  RUN_STEPFN="false"
fi

if [[ "$RUN_STEPFN" == "true" ]]; then
  if [[ "$HAS_EXISTING_ARN" != "true" ]]; then
    echo "==> Creating state machine..."
    CREATE_OUT="$(aws stepfunctions create-state-machine \
      --region "$AWS_REGION" \
      --name "$STATE_MACHINE_NAME" \
      --type "$TYPE" \
      --role-arn "$SFN_EXECUTION_ROLE_ARN" \
      --definition "file://${RENDERED_DEF}" \
      "${TRACING_ARGS[@]}" \
      --tags "$TAGS_ARG" \
      --output json)"

    STATE_MACHINE_ARN="$(echo "$CREATE_OUT" | jq -r '.stateMachineArn')"
  else
    STATE_MACHINE_ARN="$EXISTING_ARN"
    if [[ "$STATE_MACHINE_ARN" != arn:* ]]; then
      echo "Resolved existing state machine ARN is invalid: '$STATE_MACHINE_ARN'" >&2
      exit 2
    fi
    echo "==> Updating state machine: $STATE_MACHINE_ARN"
    aws stepfunctions update-state-machine \
      --region "$AWS_REGION" \
      --state-machine-arn "$STATE_MACHINE_ARN" \
      --role-arn "$SFN_EXECUTION_ROLE_ARN" \
      --definition "file://${RENDERED_DEF}" \
      "${TRACING_ARGS[@]}" \
      --output json >/dev/null

    # Tags are not updated by update-state-machine; enforce them.
    aws stepfunctions tag-resource \
      --region "$AWS_REGION" \
      --resource-arn "$STATE_MACHINE_ARN" \
      --tags "$TAGS_ARG" \
      --output json >/dev/null
  fi
else
  STATE_MACHINE_ARN="$EXISTING_ARN"
  echo "==> Step Function unchanged by state hash; skipping state machine update."
fi

if [[ "$TRIGGER_EXISTS" == "true" ]]; then
  EVENTBRIDGE_ENABLED="$(jq -r '.eventBridge.enabled // true' "$RENDERED_TRIGGER")"
else
  EVENTBRIDGE_ENABLED="false"
fi

RULE_TEMPLATE="${EVENTBRIDGE_RULE_NAME_TEMPLATE:-__ENV__-__WORKFLOW__-trigger}"
RULE_NAME="${RULE_TEMPLATE//__ENV__/$ENV_NAME}"
RULE_NAME="${RULE_NAME//__WORKFLOW__/$WF_ID}"
RULE_NAME="${RULE_NAME//\\\$\{ENV\}/$ENV_NAME}"
RULE_NAME="${RULE_NAME//\$\{ENV\}/$ENV_NAME}"
RULE_NAME="${RULE_NAME//\\\$\{WORKFLOW\}/$WF_ID}"
RULE_NAME="${RULE_NAME//\$\{WORKFLOW\}/$WF_ID}"
EVENT_BUS="${EVENTBRIDGE_EVENT_BUS_NAME:-default}"
TRIGGER_HASH=""
TRIGGER_COMMENT=""

if [[ "$EVENTBRIDGE_ENABLED" == "true" ]]; then
  if [[ -z "$INPUT_PATH" ]]; then
    echo "Missing input file required for EventBridge target: expected ${WORKFLOW_DIR}/${ENV_NAME}-input.json or ${WORKFLOW_DIR}/input.json" >&2
    exit 2
  fi
  python3 scripts/render_asl.py "$ENV_FILE" "$INPUT_PATH" "$RENDERED_INPUT"

  TRIGGER_COMMENT="$(jq -r '.comment // empty' "$RENDERED_TRIGGER")"
  RULE_STATE="$(jq -r '.eventBridge.state // "ENABLED"' "$RENDERED_TRIGGER")"
  RULE_DESCRIPTION="$(jq -r '.eventBridge.description // empty' "$RENDERED_TRIGGER")"
  SCHEDULE_EXPRESSION="$(jq -r '.eventBridge.scheduleExpression // empty' "$RENDERED_TRIGGER")"
  EVENT_PATTERN_JSON="$(jq -c '.eventBridge.eventPattern // empty' "$RENDERED_TRIGGER")"
  TARGET_ID="$(jq -r '.eventBridge.targetId // "stepfunction-target"' "$RENDERED_TRIGGER")"
  TARGET_ROLE_ARN="$(jq -r '.eventBridge.roleArn // empty' "$RENDERED_TRIGGER")"

  if [[ -z "$TARGET_ROLE_ARN" && -n "${EVENTBRIDGE_INVOKE_ROLE_ARN:-}" ]]; then
    TARGET_ROLE_ARN="$EVENTBRIDGE_INVOKE_ROLE_ARN"
  fi
  if [[ -z "$TARGET_ROLE_ARN" ]]; then
    echo "Missing eventBridge.roleArn in selected trigger file and EVENTBRIDGE_INVOKE_ROLE_ARN env var" >&2
    exit 2
  fi

  if [[ -n "$SCHEDULE_EXPRESSION" && -n "$EVENT_PATTERN_JSON" ]]; then
    echo "Trigger config must specify only one of eventBridge.scheduleExpression or eventBridge.eventPattern" >&2
    exit 2
  fi
  if [[ -z "$SCHEDULE_EXPRESSION" && -z "$EVENT_PATTERN_JSON" ]]; then
    echo "Trigger config must specify eventBridge.scheduleExpression or eventBridge.eventPattern" >&2
    exit 2
  fi

  TRIGGER_HASH="$(
    {
      sha256_of_file "$RENDERED_TRIGGER"
      sha256_of_file "$RENDERED_INPUT"
      printf '%s\n' "$STATE_MACHINE_ARN"
      printf '%s\n' "$RULE_NAME"
      printf '%s\n' "$EVENT_BUS"
      printf '%s\n' "$TARGET_ID"
      printf '%s\n' "$TARGET_ROLE_ARN"
    } | sha256_of_payload
  )"

  RULE_DESC_JSON="$(aws events describe-rule \
    --region "$AWS_REGION" \
    --name "$RULE_NAME" \
    --event-bus-name "$EVENT_BUS" \
    --output json 2>/dev/null || echo '{}')"
  RULE_ARN="$(echo "$RULE_DESC_JSON" | jq -r '.Arn // empty')"
  RULE_EXISTS="true"
  if [[ -z "${RULE_ARN//[[:space:]]/}" || "$RULE_ARN" != arn:* ]]; then
    RULE_EXISTS="false"
  fi

  RUN_TRIGGER="true"
  if [[ -n "$PREV_TRIGGER_HASH" && "$PREV_TRIGGER_HASH" == "$TRIGGER_HASH" && "$RUN_STEPFN" == "false" && "$RULE_EXISTS" == "true" ]]; then
    RUN_TRIGGER="false"
  fi

  if [[ "$RUN_TRIGGER" == "false" ]]; then
    echo "==> EventBridge trigger unchanged by state hash; skipping trigger update."
  else
    RULE_ARGS=(
      --region "$AWS_REGION"
      --name "$RULE_NAME"
      --event-bus-name "$EVENT_BUS"
      --state "$RULE_STATE"
    )
    if [[ -n "$RULE_DESCRIPTION" ]]; then
      RULE_ARGS+=(--description "$RULE_DESCRIPTION")
    fi
    if [[ -n "$SCHEDULE_EXPRESSION" ]]; then
      RULE_ARGS+=(--schedule-expression "$SCHEDULE_EXPRESSION")
    else
      RULE_ARGS+=(--event-pattern "$EVENT_PATTERN_JSON")
    fi

    echo "==> Upserting EventBridge rule: $RULE_NAME"
    aws events put-rule "${RULE_ARGS[@]}" --output json >/dev/null

    INPUT_JSON="$(jq -c . "$RENDERED_INPUT")"
    TARGETS_JSON="$(jq -cn \
      --arg id "$TARGET_ID" \
      --arg arn "$STATE_MACHINE_ARN" \
      --arg role "$TARGET_ROLE_ARN" \
      --arg input "$INPUT_JSON" \
      '[{Id: $id, Arn: $arn, RoleArn: $role, Input: $input}]'
    )"

    PUT_TARGETS_OUT="$(aws events put-targets \
      --region "$AWS_REGION" \
      --event-bus-name "$EVENT_BUS" \
      --rule "$RULE_NAME" \
      --targets "$TARGETS_JSON" \
      --output json)"

    FAILED_COUNT="$(echo "$PUT_TARGETS_OUT" | jq -r '.FailedEntryCount // 0')"
    if [[ "$FAILED_COUNT" != "0" ]]; then
      echo "Failed to attach one or more EventBridge targets: $PUT_TARGETS_OUT" >&2
      exit 1
    fi
  fi
elif [[ "$TRIGGER_EXISTS" == "false" ]]; then
  if [[ -n "$PREV_TRIGGER_HASH" ]]; then
    echo "==> No trigger config found; removing existing EventBridge rule if present."
    delete_eventbridge_rule_if_exists "$RULE_NAME" "$EVENT_BUS"
  else
    echo "==> No trigger config found; skipping EventBridge setup."
  fi
else
  if [[ -n "$PREV_TRIGGER_HASH" ]]; then
    echo "==> Trigger config present but EventBridge disabled; removing existing EventBridge rule if present."
    delete_eventbridge_rule_if_exists "$RULE_NAME" "$EVENT_BUS"
  else
    echo "==> Trigger config present but EventBridge disabled; skipping EventBridge setup."
  fi
fi

S3_HASH=""
S3_COMMENT=""
if [[ -f "$RENDERED_S3_COPY" ]]; then
  S3_HASH="$(sha256_of_file "$RENDERED_S3_COPY")"
  S3_COMMENT="$(jq -r '.comment // empty' "$RENDERED_S3_COPY")"
  RUN_S3="true"
  if [[ -n "$PREV_S3_HASH" && "$PREV_S3_HASH" == "$S3_HASH" ]]; then
    RUN_S3="false"
  fi

  if [[ "$RUN_S3" == "false" ]]; then
    echo "==> S3 copy unchanged by state hash; skipping S3 copy tasks."
  else
  mapfile -t S3_TASKS < <(jq -c 'if has("tasks") then .tasks[]? else . end' "$RENDERED_S3_COPY")
  if (( ${#S3_TASKS[@]} > 0 )); then
    for task_index in "${!S3_TASKS[@]}"; do
      task_json="${S3_TASKS[$task_index]}"
      S3_COPY_ENABLED="$(jq -r '.enabled // true' <<< "$task_json")"
      if [[ "$S3_COPY_ENABLED" != "true" ]]; then
        echo "==> Skipping S3 copy task #$((task_index + 1)) (enabled=false)"
        continue
      fi

      SOURCE_URI="$(jq -r '.sourceUri // empty' <<< "$task_json")"
      DESTINATION_URI="$(jq -r '.destinationUri // empty' <<< "$task_json")"
      COPY_MODE="$(jq -r '.mode // empty' <<< "$task_json")"
      DELETE_EXTRA_FILES="$(jq -r '.deleteExtraFiles // false' <<< "$task_json")"
      BACKUP_EXISTING="$(jq -r '.backupExisting // false' <<< "$task_json")"
      BACKUP_FORMAT="$(jq -r '.backupTimestampFormat // "%Y%m%d%H%M%S"' <<< "$task_json")"
      FILE_COUNT="$(jq -r '(.files // []) | length' <<< "$task_json")"

      if [[ -z "$SOURCE_URI" || -z "$DESTINATION_URI" ]]; then
        echo "s3copy task #$((task_index + 1)) requires sourceUri and destinationUri when enabled=true" >&2
        exit 2
      fi
      if [[ "$SOURCE_URI" != s3://* || "$DESTINATION_URI" != s3://* ]]; then
        echo "S3 copy URIs must start with s3:// (sourceUri=$SOURCE_URI, destinationUri=$DESTINATION_URI)" >&2
        exit 2
      fi

      if [[ -z "$COPY_MODE" || "$COPY_MODE" == "null" ]]; then
        if (( FILE_COUNT > 0 )); then
          COPY_MODE="files"
        else
          COPY_MODE="sync"
        fi
      fi

      echo "==> Running S3 copy task #$((task_index + 1)) (${COPY_MODE}) from ${SOURCE_URI} to ${DESTINATION_URI}"
      case "$COPY_MODE" in
        files)
          if (( FILE_COUNT <= 0 )); then
            echo "s3copy.mode=files requires a non-empty files array (task #$((task_index + 1)))" >&2
            exit 2
          fi
          mapfile -t S3_URI_PARTS < <(parse_s3_uri "$SOURCE_URI")
          SOURCE_BUCKET="${S3_URI_PARTS[0]}"
          SOURCE_PREFIX="${S3_URI_PARTS[1]}"

          mapfile -t S3_URI_PARTS < <(parse_s3_uri "$DESTINATION_URI")
          DEST_BUCKET="${S3_URI_PARTS[0]}"
          DEST_PREFIX="${S3_URI_PARTS[1]}"

          if [[ -z "$SOURCE_BUCKET" || -z "$DEST_BUCKET" ]]; then
            echo "Failed to parse s3copy source/destination bucket names" >&2
            exit 2
          fi

          BACKUP_SUFFIX="$(date -u +"$BACKUP_FORMAT")"
          mapfile -t FILES_TO_COPY < <(jq -r '.files[]? // empty' <<< "$task_json")
          for rel_path in "${FILES_TO_COPY[@]}"; do
            [[ -z "$rel_path" ]] && continue
            if [[ "$rel_path" == s3://* || "$rel_path" == /* ]]; then
              echo "s3copy.files entries must be relative paths: $rel_path" >&2
              exit 2
            fi

            src_key="$(join_s3_key "$SOURCE_PREFIX" "$rel_path")"
            dst_key="$(join_s3_key "$DEST_PREFIX" "$rel_path")"

            if [[ -z "$src_key" || -z "$dst_key" ]]; then
              echo "Invalid resolved S3 key for rel_path=$rel_path" >&2
              exit 2
            fi

            if [[ "$BACKUP_EXISTING" == "true" ]]; then
              if aws s3api head-object --region "$AWS_REGION" --bucket "$DEST_BUCKET" --key "$dst_key" >/dev/null 2>&1; then
                backup_key="${dst_key}.${BACKUP_SUFFIX}"
                echo "==> Backing up existing object: s3://${DEST_BUCKET}/${dst_key} -> s3://${DEST_BUCKET}/${backup_key}"
                aws s3 cp "s3://${DEST_BUCKET}/${dst_key}" "s3://${DEST_BUCKET}/${backup_key}"
              fi
            fi

            echo "==> Copying file: s3://${SOURCE_BUCKET}/${src_key} -> s3://${DEST_BUCKET}/${dst_key}"
            aws s3 cp "s3://${SOURCE_BUCKET}/${src_key}" "s3://${DEST_BUCKET}/${dst_key}"
          done
          ;;
        sync)
          if (( FILE_COUNT > 0 )); then
            echo "s3copy.files is only supported with mode=files" >&2
            exit 2
          fi
          if [[ "$BACKUP_EXISTING" == "true" ]]; then
            echo "s3copy.backupExisting=true currently requires mode=files" >&2
            exit 2
          fi
          SYNC_ARGS=(s3 sync "$SOURCE_URI" "$DESTINATION_URI")
          if [[ "$DELETE_EXTRA_FILES" == "true" ]]; then
            SYNC_ARGS+=(--delete)
          fi
          aws "${SYNC_ARGS[@]}"
          ;;
        cp)
          if (( FILE_COUNT > 0 )); then
            echo "s3copy.files is only supported with mode=files" >&2
            exit 2
          fi
          if [[ "$BACKUP_EXISTING" == "true" ]]; then
            echo "s3copy.backupExisting=true currently requires mode=files" >&2
            exit 2
          fi
          aws s3 cp "$SOURCE_URI" "$DESTINATION_URI" --recursive
          ;;
        *)
          echo "Unsupported s3copy.mode: $COPY_MODE (supported: files, sync, cp)" >&2
          exit 2
          ;;
      esac
    done
  fi
  fi
fi

UPDATED_AT="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
STATE_UPDATED="${TMP_DIR}/deploy-state.updated.json"

jq -c \
  --arg env "$ENV_NAME" \
  --arg id "$WF_ID" \
  --arg hash "$STEPFN_HASH" \
  --arg sm_name "$STATE_MACHINE_NAME" \
  --arg sm_arn "$STATE_MACHINE_ARN" \
  --arg updated_at "$UPDATED_AT" \
  '
  .version = (.version // 1)
  | .environment = $env
  | .workflows = (.workflows // {})
  | .workflows[$id] = ((.workflows[$id] // {}) + {
      stepFunction: {
        hash: $hash,
        stateMachineName: $sm_name,
        stateMachineArn: $sm_arn,
        updatedAt: $updated_at
      }
    })
  ' "$STATE_FILE_LOCAL" > "$STATE_UPDATED"

if [[ -n "$TRIGGER_HASH" ]]; then
  STATE_UPDATED_NEXT="${TMP_DIR}/deploy-state.updated.trigger.json"
  jq -c \
    --arg id "$WF_ID" \
    --arg hash "$TRIGGER_HASH" \
    --arg rule_name "$RULE_NAME" \
    --arg event_bus "$EVENT_BUS" \
    --arg comment "$TRIGGER_COMMENT" \
    --arg updated_at "$UPDATED_AT" \
    '
    .workflows[$id].trigger = {
      hash: $hash,
      ruleName: $rule_name,
      eventBusName: $event_bus,
      comment: $comment,
      updatedAt: $updated_at
    }
    ' "$STATE_UPDATED" > "$STATE_UPDATED_NEXT"
  mv "$STATE_UPDATED_NEXT" "$STATE_UPDATED"
else
  STATE_UPDATED_NEXT="${TMP_DIR}/deploy-state.updated.trigger-removed.json"
  jq -c --arg id "$WF_ID" '
    if (.workflows[$id] // null) == null then
      .
    else
      .workflows[$id] |= del(.trigger)
    end
  ' "$STATE_UPDATED" > "$STATE_UPDATED_NEXT"
  mv "$STATE_UPDATED_NEXT" "$STATE_UPDATED"
fi

if [[ -n "$S3_HASH" ]]; then
  S3_TASK_COUNT="$(jq -r 'if has("tasks") then (.tasks | length) else 1 end' "$RENDERED_S3_COPY")"
  STATE_UPDATED_NEXT="${TMP_DIR}/deploy-state.updated.s3.json"
  jq -c \
    --arg id "$WF_ID" \
    --arg hash "$S3_HASH" \
    --arg comment "$S3_COMMENT" \
    --arg updated_at "$UPDATED_AT" \
    --argjson task_count "$S3_TASK_COUNT" \
    '
    .workflows[$id].s3Copy = {
      hash: $hash,
      comment: $comment,
      taskCount: $task_count,
      updatedAt: $updated_at
    }
    ' "$STATE_UPDATED" > "$STATE_UPDATED_NEXT"
  mv "$STATE_UPDATED_NEXT" "$STATE_UPDATED"
else
  STATE_UPDATED_NEXT="${TMP_DIR}/deploy-state.updated.s3-removed.json"
  jq -c --arg id "$WF_ID" '
    if (.workflows[$id] // null) == null then
      .
    else
      .workflows[$id] |= del(.s3Copy)
    end
  ' "$STATE_UPDATED" > "$STATE_UPDATED_NEXT"
  mv "$STATE_UPDATED_NEXT" "$STATE_UPDATED"
fi

aws s3 cp "$STATE_UPDATED" "$STATE_FILE_S3_URI" --region "$AWS_REGION" >/dev/null
echo "==> Updated deploy state file: $STATE_FILE_S3_URI"

echo "==> Done: $STATE_MACHINE_ARN"
