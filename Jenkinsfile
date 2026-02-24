name: stepfunctions-cicd

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches: ["cert", "prod", "main"]
    paths:
      - "stepfunctions/**"
      - "scripts/**"
      - ".github/workflows/stepfunctions-cicd.yml"
  workflow_dispatch:
    inputs:
      deploy:
        description: "Target environment"
        required: true
        type: choice
        options:
          - cert
          - prod
      mode:
        description: "Deploy mode"
        required: true
        type: choice
        options:
          - changed
          - all

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Validate workflow bundle JSON files
        shell: bash
        run: |
          set -euo pipefail

          for f in stepfunctions/env/*.json; do
            jq -e . "$f" >/dev/null
          done

          WORKFLOW_DIRS="$(git ls-files 'stepfunctions/workflows/*/definition.asl.json' | sed 's#/definition.asl.json$##' | sort -u)"
          if [[ -z "${WORKFLOW_DIRS//[[:space:]]/}" ]]; then
            echo "No workflow bundles found under stepfunctions/workflows"
            exit 0
          fi

          while IFS= read -r d; do
            [[ -z "$d" ]] && continue
            req="$d/definition.asl.json"
            [[ -f "$req" ]] || { echo "Missing required file: $req" >&2; exit 1; }
            # definition.asl.json can contain unquoted ${...} placeholders.
            # It is validated after rendering in the next step.

            for optional in \
              "$d/cert-trigger.json" \
              "$d/prod-trigger.json" \
              "$d/trigger.json" \
              "$d/cert-input.json" \
              "$d/prod-input.json" \
              "$d/input.json" \
              "$d/cert-s3copy.json" \
              "$d/prod-s3copy.json" \
              "$d/s3copy.json"; do
              if [[ -f "$optional" ]]; then
                jq -e . "$optional" >/dev/null
              fi
            done
          done <<< "$WORKFLOW_DIRS"

      - name: Smoke render workflow bundles (cert + prod)
        shell: bash
        run: |
          set -euo pipefail
          WORKFLOW_DIRS="$(git ls-files 'stepfunctions/workflows/*/definition.asl.json' | sed 's#/definition.asl.json$##' | sort -u)"
          [[ -z "${WORKFLOW_DIRS//[[:space:]]/}" ]] && exit 0

          for env_name in cert prod; do
            env_file="stepfunctions/env/${env_name}.json"
            [[ -f "$env_file" ]] || { echo "Missing env file: $env_file" >&2; exit 1; }

            while IFS= read -r d; do
              [[ -z "$d" ]] && continue

              render_def="$(mktemp)"
              render_trigger="$(mktemp)"
              render_input="$(mktemp)"
              render_s3="$(mktemp)"

              python3 scripts/render_asl.py "$env_file" "$d/definition.asl.json" "$render_def"
              jq -e . "$render_def" >/dev/null

              trigger_path=""
              if [[ -f "$d/${env_name}-trigger.json" ]]; then
                trigger_path="$d/${env_name}-trigger.json"
              elif [[ -f "$d/trigger.json" ]]; then
                trigger_path="$d/trigger.json"
              fi

              if [[ -n "$trigger_path" ]]; then
                python3 scripts/render_asl.py "$env_file" "$trigger_path" "$render_trigger"
                jq -e . "$render_trigger" >/dev/null

                eb_enabled="$(jq -r '.eventBridge.enabled // true' "$render_trigger")"
                if [[ "$eb_enabled" == "true" ]]; then
                  schedule_expr="$(jq -r '.eventBridge.scheduleExpression // empty' "$render_trigger")"
                  event_pattern="$(jq -c '.eventBridge.eventPattern // empty' "$render_trigger")"
                  if [[ -n "$schedule_expr" && -n "$event_pattern" ]]; then
                    echo "Trigger config in $trigger_path specifies both scheduleExpression and eventPattern" >&2
                    exit 1
                  fi
                  if [[ -z "$schedule_expr" && -z "$event_pattern" ]]; then
                    echo "Trigger config in $trigger_path must specify scheduleExpression or eventPattern when eventBridge.enabled=true" >&2
                    exit 1
                  fi

                  input_path=""
                  if [[ -f "$d/${env_name}-input.json" ]]; then
                    input_path="$d/${env_name}-input.json"
                  elif [[ -f "$d/input.json" ]]; then
                    input_path="$d/input.json"
                  fi

                  [[ -n "$input_path" ]] || { echo "Missing required input file for EventBridge target in $d: expected ${env_name}-input.json or input.json" >&2; exit 1; }
                  jq -e . "$input_path" >/dev/null
                  python3 scripts/render_asl.py "$env_file" "$input_path" "$render_input"
                  jq -e . "$render_input" >/dev/null
                fi
              fi

              s3_file_path=""
              if [[ -f "$d/${env_name}-s3copy.json" ]]; then
                s3_file_path="$d/${env_name}-s3copy.json"
              elif [[ -f "$d/s3copy.json" ]]; then
                s3_file_path="$d/s3copy.json"
              fi
              if [[ -n "$s3_file_path" ]]; then
                jq -e . "$s3_file_path" >/dev/null
                python3 scripts/render_asl.py "$env_file" "$s3_file_path" "$render_s3"
              fi

              if [[ -s "$render_s3" ]]; then
                mapfile -t s3_tasks < <(jq -c 'if has("tasks") then .tasks[]? else . end' "$render_s3")
                for task_json in "${s3_tasks[@]}"; do
                  s3_enabled="$(jq -r '.enabled // true' <<< "$task_json")"
                  if [[ "$s3_enabled" != "true" ]]; then
                    continue
                  fi

                  source_uri="$(jq -r '.sourceUri // empty' <<< "$task_json")"
                  destination_uri="$(jq -r '.destinationUri // empty' <<< "$task_json")"
                  [[ "$source_uri" == s3://* ]] || { echo "Invalid s3copy.sourceUri in $d" >&2; exit 1; }
                  [[ "$destination_uri" == s3://* ]] || { echo "Invalid s3copy.destinationUri in $d" >&2; exit 1; }

                  files_count="$(jq -r '(.files // []) | length' <<< "$task_json")"
                  copy_mode="$(jq -r '.mode // empty' <<< "$task_json")"
                  backup_existing="$(jq -r '.backupExisting // false' <<< "$task_json")"
                  if [[ -z "$copy_mode" || "$copy_mode" == "null" ]]; then
                    if (( files_count > 0 )); then
                      copy_mode="files"
                    else
                      copy_mode="sync"
                    fi
                  fi

                  if (( files_count > 0 )) && [[ "$copy_mode" != "files" ]]; then
                    echo "s3copy.files in $d requires mode=files" >&2
                    exit 1
                  fi

                  if [[ "$copy_mode" == "files" ]]; then
                    (( files_count > 0 )) || { echo "s3copy.mode=files in $d requires files array" >&2; exit 1; }
                    while IFS= read -r rel_path; do
                      [[ -n "$rel_path" ]] || continue
                      [[ "$rel_path" != s3://* ]] || { echo "s3copy.files entries must be relative paths in $d" >&2; exit 1; }
                      [[ "$rel_path" != /* ]] || { echo "s3copy.files entries must not start with / in $d" >&2; exit 1; }
                    done < <(jq -r '.files[]? // empty' <<< "$task_json")
                  fi

                  if [[ "$backup_existing" == "true" && "$copy_mode" != "files" ]]; then
                    echo "s3copy.backupExisting=true in $d currently requires mode=files" >&2
                    exit 1
                  fi
                done
              fi

              rm -f "$render_trigger" "$render_def" "$render_input" "$render_s3"
            done <<< "$WORKFLOW_DIRS"
          done

  deploy-cert:
    # Manual-only deploy via workflow_dispatch.
    if: |
      github.event_name == 'workflow_dispatch' && inputs.deploy == 'cert'
    needs: [validate]
    runs-on: ubuntu-latest
    environment: cert
    concurrency:
      group: stepfunctions-cert
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure AWS credentials (cert)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Determine deploy + delete changes (cert)
        id: changes
        shell: bash
        run: |
          set -euo pipefail

          if [[ "${{ github.event_name }}" == "workflow_dispatch" && "${{ inputs.mode }}" == "all" ]]; then
            DEPLOY_LIST="$(git ls-files 'stepfunctions/workflows/*/definition.asl.json' | sed 's#/definition.asl.json$##' | sort -u)"
            DELETE_LIST=""
          else
            HEAD_SHA="${{ github.sha }}"
            if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
              BASE_SHA=""

              # For manual changed mode, diff from last successful deployment checkpoint.
              if [[ "${{ inputs.mode }}" == "changed" ]]; then
                STATE_TMP="$(mktemp)"
                LAST_DEPLOYED_SHA=""
                if aws s3 cp "${{ secrets.SFN_STATE_FILE_S3_URI_CERT }}" "$STATE_TMP" --region "${{ secrets.AWS_REGION }}" >/dev/null 2>&1; then
                  LAST_DEPLOYED_SHA="$(jq -r '.lastSuccessfulCommit // empty' "$STATE_TMP" 2>/dev/null || true)"
                fi
                rm -f "$STATE_TMP"

                if [[ -n "$LAST_DEPLOYED_SHA" ]] && git cat-file -e "${LAST_DEPLOYED_SHA}^{commit}" >/dev/null 2>&1; then
                  BASE_SHA="$LAST_DEPLOYED_SHA"
                fi
              fi

              # Fallback when no checkpoint exists yet.
              if [[ -z "$BASE_SHA" ]]; then
                if git rev-parse "${HEAD_SHA}^" >/dev/null 2>&1; then
                  BASE_SHA="$(git rev-parse "${HEAD_SHA}^")"
                else
                  # No parent commit (e.g. first commit on branch) => treat as full deploy.
                  BASE_SHA=""
                fi
              fi
            else
              BASE_SHA="${{ github.event.before }}"
            fi

            if [[ -z "${BASE_SHA}" ]]; then
              DEPLOY_LIST="$(git ls-files 'stepfunctions/workflows/*/definition.asl.json' | sed 's#/definition.asl.json$##' | sort -u)"
              DELETE_LIST=""
            else
              RAW="$(scripts/detect_stepfunctions_changes.sh "$BASE_SHA" "$HEAD_SHA" || true)"
              DEPLOY_LIST="$(echo "$RAW" | awk -F: '$1=="DEPLOY"{print $2}' | sed '/^\s*$/d' || true)"
              DELETE_LIST="$(echo "$RAW" | awk -F: '$1=="DELETE"{print $2}' | sed '/^\s*$/d' || true)"
            fi
          fi

          echo "Deploy definitions:"
          echo "$DEPLOY_LIST"
          echo
          echo "Delete workflow ids:"
          echo "$DELETE_LIST"

          {
            echo "deploy<<EOF"
            echo "$DEPLOY_LIST"
            echo "EOF"
            echo "delete<<EOF"
            echo "$DELETE_LIST"
            echo "EOF"
          } >> "$GITHUB_OUTPUT"

      - name: Deploy changed workflows to cert
        if: ${{ steps.changes.outputs.deploy != '' }}
        shell: bash
        env:
          AWS_REGION: ${{ secrets.AWS_REGION }}
          SFN_EXECUTION_ROLE_ARN: ${{ secrets.SFN_EXECUTION_ROLE_ARN_CERT }}
          SFN_STATE_FILE_S3_URI: ${{ secrets.SFN_STATE_FILE_S3_URI_CERT }}
          SFN_STATE_MACHINE_NAME_TEMPLATE: "dummy-__ENV__-__WORKFLOW__"
          EVENTBRIDGE_RULE_NAME_TEMPLATE: "dummy-__ENV__-__WORKFLOW__-trigger"
          EVENTBRIDGE_EVENT_BUS_NAME: "default"
          ENV_NAME: cert
        run: |
          set -euo pipefail
          chmod +x scripts/deploy_sfn.sh scripts/detect_stepfunctions_changes.sh scripts/delete_sfn.sh
          while IFS= read -r d; do
            [[ -z "$d" ]] && continue
            scripts/deploy_sfn.sh "$ENV_NAME" "$d"
          done <<< "${{ steps.changes.outputs.deploy }}"

      - name: Delete removed workflows in cert
        if: ${{ steps.changes.outputs.delete != '' }}
        shell: bash
        env:
          AWS_REGION: ${{ secrets.AWS_REGION }}
          SFN_STATE_FILE_S3_URI: ${{ secrets.SFN_STATE_FILE_S3_URI_CERT }}
          SFN_STATE_MACHINE_NAME_TEMPLATE: "dummy-__ENV__-__WORKFLOW__"
          EVENTBRIDGE_RULE_NAME_TEMPLATE: "dummy-__ENV__-__WORKFLOW__-trigger"
          EVENTBRIDGE_EVENT_BUS_NAME: "default"
          ENV_NAME: cert
        run: |
          set -euo pipefail
          chmod +x scripts/delete_sfn.sh
          while IFS= read -r id; do
            [[ -z "$id" ]] && continue
            scripts/delete_sfn.sh "$ENV_NAME" "$id"
          done <<< "${{ steps.changes.outputs.delete }}"

      - name: Update deploy checkpoint (cert)
        if: ${{ success() }}
        shell: bash
        env:
          AWS_REGION: ${{ secrets.AWS_REGION }}
          SFN_STATE_FILE_S3_URI: ${{ secrets.SFN_STATE_FILE_S3_URI_CERT }}
          ENV_NAME: cert
          DEPLOY_COMMIT: ${{ github.sha }}
        run: |
          set -euo pipefail
          chmod +x scripts/update_sfn_state_checkpoint.sh
          scripts/update_sfn_state_checkpoint.sh "$ENV_NAME" "$DEPLOY_COMMIT"

  deploy-prod:
    # Prod is gated by GitHub Environment approvals (configure in repo settings).
    # Manual-only deploy via workflow_dispatch.
    if: |
      github.event_name == 'workflow_dispatch' && inputs.deploy == 'prod'
    needs: [validate]
    runs-on: ubuntu-latest
    environment: prod
    concurrency:
      group: stepfunctions-prod
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure AWS credentials (prod)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Determine deploy + delete changes (prod)
        id: changes
        shell: bash
        run: |
          set -euo pipefail

          if [[ "${{ github.event_name }}" == "workflow_dispatch" && "${{ inputs.mode }}" == "all" ]]; then
            DEPLOY_LIST="$(git ls-files 'stepfunctions/workflows/*/definition.asl.json' | sed 's#/definition.asl.json$##' | sort -u)"
            DELETE_LIST=""
          else
            HEAD_SHA="${{ github.sha }}"
            if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
              BASE_SHA=""

              # For manual changed mode, diff from last successful deployment checkpoint.
              if [[ "${{ inputs.mode }}" == "changed" ]]; then
                STATE_TMP="$(mktemp)"
                LAST_DEPLOYED_SHA=""
                if aws s3 cp "${{ secrets.SFN_STATE_FILE_S3_URI_PROD }}" "$STATE_TMP" --region "${{ secrets.AWS_REGION }}" >/dev/null 2>&1; then
                  LAST_DEPLOYED_SHA="$(jq -r '.lastSuccessfulCommit // empty' "$STATE_TMP" 2>/dev/null || true)"
                fi
                rm -f "$STATE_TMP"

                if [[ -n "$LAST_DEPLOYED_SHA" ]] && git cat-file -e "${LAST_DEPLOYED_SHA}^{commit}" >/dev/null 2>&1; then
                  BASE_SHA="$LAST_DEPLOYED_SHA"
                fi
              fi

              # Fallback when no checkpoint exists yet.
              if [[ -z "$BASE_SHA" ]]; then
                if git rev-parse "${HEAD_SHA}^" >/dev/null 2>&1; then
                  BASE_SHA="$(git rev-parse "${HEAD_SHA}^")"
                else
                  # No parent commit (e.g. first commit on branch) => treat as full deploy.
                  BASE_SHA=""
                fi
              fi
            else
              BASE_SHA="${{ github.event.before }}"
            fi

            if [[ -z "${BASE_SHA}" ]]; then
              DEPLOY_LIST="$(git ls-files 'stepfunctions/workflows/*/definition.asl.json' | sed 's#/definition.asl.json$##' | sort -u)"
              DELETE_LIST=""
            else
              RAW="$(scripts/detect_stepfunctions_changes.sh "$BASE_SHA" "$HEAD_SHA" || true)"
              DEPLOY_LIST="$(echo "$RAW" | awk -F: '$1=="DEPLOY"{print $2}' | sed '/^\s*$/d' || true)"
              DELETE_LIST="$(echo "$RAW" | awk -F: '$1=="DELETE"{print $2}' | sed '/^\s*$/d' || true)"
            fi
          fi

          echo "Deploy definitions:"
          echo "$DEPLOY_LIST"
          echo
          echo "Delete workflow ids:"
          echo "$DELETE_LIST"

          {
            echo "deploy<<EOF"
            echo "$DEPLOY_LIST"
            echo "EOF"
            echo "delete<<EOF"
            echo "$DELETE_LIST"
            echo "EOF"
          } >> "$GITHUB_OUTPUT"

      - name: Deploy to prod
        if: ${{ steps.changes.outputs.deploy != '' }}
        shell: bash
        env:
          AWS_REGION: ${{ secrets.AWS_REGION }}
          SFN_EXECUTION_ROLE_ARN: ${{ secrets.SFN_EXECUTION_ROLE_ARN_PROD }}
          SFN_STATE_FILE_S3_URI: ${{ secrets.SFN_STATE_FILE_S3_URI_PROD }}
          SFN_STATE_MACHINE_NAME_TEMPLATE: "dummy-__ENV__-__WORKFLOW__"
          EVENTBRIDGE_RULE_NAME_TEMPLATE: "dummy-__ENV__-__WORKFLOW__-trigger"
          EVENTBRIDGE_EVENT_BUS_NAME: "default"
          ENV_NAME: prod
        run: |
          set -euo pipefail
          chmod +x scripts/deploy_sfn.sh scripts/detect_stepfunctions_changes.sh scripts/delete_sfn.sh
          while IFS= read -r d; do
            [[ -z "$d" ]] && continue
            scripts/deploy_sfn.sh "$ENV_NAME" "$d"
          done <<< "${{ steps.changes.outputs.deploy }}"

      - name: Delete removed workflows in prod
        if: ${{ steps.changes.outputs.delete != '' }}
        shell: bash
        env:
          AWS_REGION: ${{ secrets.AWS_REGION }}
          SFN_STATE_FILE_S3_URI: ${{ secrets.SFN_STATE_FILE_S3_URI_PROD }}
          SFN_STATE_MACHINE_NAME_TEMPLATE: "dummy-__ENV__-__WORKFLOW__"
          EVENTBRIDGE_RULE_NAME_TEMPLATE: "dummy-__ENV__-__WORKFLOW__-trigger"
          EVENTBRIDGE_EVENT_BUS_NAME: "default"
          ENV_NAME: prod
        run: |
          set -euo pipefail
          chmod +x scripts/delete_sfn.sh
          while IFS= read -r id; do
            [[ -z "$id" ]] && continue
            scripts/delete_sfn.sh "$ENV_NAME" "$id"
          done <<< "${{ steps.changes.outputs.delete }}"

      - name: Update deploy checkpoint (prod)
        if: ${{ success() }}
        shell: bash
        env:
          AWS_REGION: ${{ secrets.AWS_REGION }}
          SFN_STATE_FILE_S3_URI: ${{ secrets.SFN_STATE_FILE_S3_URI_PROD }}
          ENV_NAME: prod
          DEPLOY_COMMIT: ${{ github.sha }}
        run: |
          set -euo pipefail
          chmod +x scripts/update_sfn_state_checkpoint.sh
          scripts/update_sfn_state_checkpoint.sh "$ENV_NAME" "$DEPLOY_COMMIT"
