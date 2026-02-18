#!/usr/bin/env bash
set -euo pipefail

# Delete a Step Functions state machine and its EventBridge trigger.
#
# Requirements:
# - AWS credentials already configured (e.g. via GitHub OIDC).
# - Env vars:
#     AWS_REGION
# - Optional:
#     SFN_STATE_MACHINE_NAME_TEMPLATE   (default: sfn-__ENV__-__WORKFLOW__)
#     EVENTBRIDGE_RULE_NAME_TEMPLATE    (default: __ENV__-__WORKFLOW__-trigger)
#     EVENTBRIDGE_EVENT_BUS_NAME        (default: default)
# - Required for state tracking:
#     SFN_STATE_FILE_S3_URI             (s3://... state file path)
# - Usage:
#     scripts/delete_sfn.sh <cert|prod> <workflow-id>

ENV_NAME="${1:-}"
WF_ID="${2:-}"

if [[ -z "$ENV_NAME" || -z "$WF_ID" ]]; then
  echo "Usage: $0 <cert|prod> <workflow-id>" >&2
  exit 2
fi

if [[ -z "${AWS_REGION:-}" ]]; then
  echo "Missing AWS_REGION env var" >&2
  exit 2
fi
if [[ -z "${SFN_STATE_FILE_S3_URI:-}" ]]; then
  echo "Missing SFN_STATE_FILE_S3_URI env var (expected s3://bucket/path/state.json)" >&2
  exit 2
fi
if [[ "${SFN_STATE_FILE_S3_URI}" != s3://* ]]; then
  echo "Invalid SFN_STATE_FILE_S3_URI (must start with s3://): ${SFN_STATE_FILE_S3_URI}" >&2
  exit 2
fi

STATE_MACHINE_TEMPLATE="${SFN_STATE_MACHINE_NAME_TEMPLATE:-sfn-__ENV__-__WORKFLOW__}"
RULE_TEMPLATE="${EVENTBRIDGE_RULE_NAME_TEMPLATE:-__ENV__-__WORKFLOW__-trigger}"
EVENT_BUS_NAME="${EVENTBRIDGE_EVENT_BUS_NAME:-default}"

STATE_MACHINE_NAME="${STATE_MACHINE_TEMPLATE//__ENV__/$ENV_NAME}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME//__WORKFLOW__/$WF_ID}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME//\\\$\{ENV\}/$ENV_NAME}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME//\$\{ENV\}/$ENV_NAME}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME//\\\$\{WORKFLOW\}/$WF_ID}"
STATE_MACHINE_NAME="${STATE_MACHINE_NAME//\$\{WORKFLOW\}/$WF_ID}"

RULE_NAME="${RULE_TEMPLATE//__ENV__/$ENV_NAME}"
RULE_NAME="${RULE_NAME//__WORKFLOW__/$WF_ID}"
RULE_NAME="${RULE_NAME//\\\$\{ENV\}/$ENV_NAME}"
RULE_NAME="${RULE_NAME//\$\{ENV\}/$ENV_NAME}"
RULE_NAME="${RULE_NAME//\\\$\{WORKFLOW\}/$WF_ID}"
RULE_NAME="${RULE_NAME//\$\{WORKFLOW\}/$WF_ID}"

echo "==> Env: $ENV_NAME"
echo "==> Deleting workflow: $WF_ID"
echo "==> State machine name: $STATE_MACHINE_NAME"
echo "==> EventBridge rule name: $RULE_NAME"

# Resolve state machine ARN robustly from JSON output.
# Avoid --output text + "None" checks; paginated output can yield multiple "None" lines.
ARN="$(
  aws stepfunctions list-state-machines \
    --region "$AWS_REGION" \
    --output json \
  | jq -r --arg name "$STATE_MACHINE_NAME" '
      ([.stateMachines[]? | select(.name == $name) | .stateMachineArn] | first) // ""
    '
)"

if [[ -z "${ARN//[[:space:]]/}" || "$ARN" != arn:* ]]; then
  echo "==> State machine not found; skipping state machine delete."
else
  aws stepfunctions delete-state-machine \
    --region "$AWS_REGION" \
    --state-machine-arn "$ARN" \
    --output json >/dev/null
  echo "==> Deleted: $ARN"
fi

RULE_DESC_JSON="$(aws events describe-rule \
  --region "$AWS_REGION" \
  --name "$RULE_NAME" \
  --event-bus-name "$EVENT_BUS_NAME" \
  --output json 2>/dev/null || echo '{}')"
RULE_ARN="$(echo "$RULE_DESC_JSON" | jq -r '.Arn // empty')"

if [[ -z "${RULE_ARN//[[:space:]]/}" || "$RULE_ARN" != arn:* ]]; then
  echo "==> EventBridge rule not found; nothing to delete."
else
  TARGETS_JSON="$(aws events list-targets-by-rule \
    --region "$AWS_REGION" \
    --event-bus-name "$EVENT_BUS_NAME" \
    --rule "$RULE_NAME" \
    --output json 2>/dev/null || echo '{}')"

  mapfile -t TARGET_ID_ARRAY < <(echo "$TARGETS_JSON" | jq -r '.Targets[]?.Id // empty')
  if (( ${#TARGET_ID_ARRAY[@]} > 0 )); then
    aws events remove-targets \
      --region "$AWS_REGION" \
      --event-bus-name "$EVENT_BUS_NAME" \
      --rule "$RULE_NAME" \
      --ids "${TARGET_ID_ARRAY[@]}" \
      --force \
      --output json >/dev/null || true
  fi

  aws events delete-rule \
    --region "$AWS_REGION" \
    --event-bus-name "$EVENT_BUS_NAME" \
    --name "$RULE_NAME" \
    --output json >/dev/null

  echo "==> Deleted EventBridge rule: $RULE_NAME"
fi

TMP_DIR="$(mktemp -d)"
trap 'rm -rf "$TMP_DIR"' EXIT
STATE_FILE_LOCAL="${TMP_DIR}/deploy-state.json"
if aws s3 cp "${SFN_STATE_FILE_S3_URI}" "$STATE_FILE_LOCAL" --region "$AWS_REGION" >/dev/null 2>&1; then
  jq -e . "$STATE_FILE_LOCAL" >/dev/null
else
  printf '{"version":1,"environment":"%s","workflows":{}}\n' "$ENV_NAME" > "$STATE_FILE_LOCAL"
fi

STATE_FILE_UPDATED="${TMP_DIR}/deploy-state.updated.json"
jq -c --arg id "$WF_ID" '
  .workflows = (.workflows // {})
  | .workflows |= with_entries(select(.key != $id))
' "$STATE_FILE_LOCAL" > "$STATE_FILE_UPDATED"

aws s3 cp "$STATE_FILE_UPDATED" "${SFN_STATE_FILE_S3_URI}" --region "$AWS_REGION" >/dev/null
echo "==> Removed workflow from deploy state file: ${SFN_STATE_FILE_S3_URI}"
