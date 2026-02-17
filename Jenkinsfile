echo "==> Env: $ENV_NAME"
echo "==> Workflow: $WF_ID"
echo "==> State machine: $STATE_MACHINE_NAME"

# Resolve existing state machine ARN robustly.
# Avoid --output text + None handling because paginated output can produce multiple "None" lines.
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

    # Safety guard: never call update with invalid ARN.
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
