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
              artifact_file_path=""
              if [[ -f "$d/${env_name}-artifacts.json" ]]; then
                artifact_file_path="$d/${env_name}-artifacts.json"
              elif [[ -f "$d/artifacts.json" ]]; then
                artifact_file_path="$d/artifacts.json"
              fi

              if [[ -n "$artifact_file_path" ]]; then
                jq -e . "$artifact_file_path" >/dev/null
                python3 scripts/render_asl.py "$env_file" "$artifact_file_path" "$render_artifacts"
              fi

              if [[ -s "$render_artifacts" ]]; then
                mapfile -t artifact_tasks < <(jq -c 'if has("tasks") then .tasks[]? else . end' "$render_artifacts")
                for task_json in "${artifact_tasks[@]}"; do
                  artifact_enabled="$(jq -r '.enabled // true' <<< "$task_json")"
                  if [[ "$artifact_enabled" != "true" ]]; then
                    continue
                  fi

                  artifact_mode="$(jq -r '.mode // empty' <<< "$task_json")"
                  source_sql_path="$(jq -r '.sourceSqlPath // empty' <<< "$task_json")"
                  source_file_path="$(jq -r '.sourceFilePath // empty' <<< "$task_json")"
                  destination_uri="$(jq -r '.destinationUri // empty' <<< "$task_json")"

                  if [[ -z "$artifact_mode" || "$artifact_mode" == "null" ]]; then
                    if [[ -n "$source_sql_path" ]]; then
                      artifact_mode="tar"
                    elif [[ -n "$source_file_path" ]]; then
                      artifact_mode="copy"
                    fi
                  fi

                  [[ "$destination_uri" == s3://* ]] || { echo "Invalid artifacts.destinationUri in $d" >&2; exit 1; }

                  case "$artifact_mode" in
                    tar)
                      [[ -n "$source_sql_path" ]] || { echo "artifacts.mode=tar in $d requires sourceSqlPath" >&2; exit 1; }
                      [[ "$source_sql_path" != s3://* ]] || { echo "artifacts.sourceSqlPath must be a local path in $d" >&2; exit 1; }
                      [[ -f "$source_sql_path" ]] || { echo "artifacts.sourceSqlPath file not found in repo: $source_sql_path ($d)" >&2; exit 1; }
                      ;;
                    copy)
                      [[ -n "$source_file_path" ]] || { echo "artifacts.mode=copy in $d requires sourceFilePath" >&2; exit 1; }
                      [[ "$source_file_path" != s3://* ]] || { echo "artifacts.sourceFilePath must be a local path in $d" >&2; exit 1; }
                      [[ -f "$source_file_path" ]] || { echo "artifacts.sourceFilePath file not found in repo: $source_file_path ($d)" >&2; exit 1; }
                      ;;
                    *)
                      echo "Unsupported artifacts.mode in $d: $artifact_mode (supported: tar, copy)" >&2
                      exit 1
                      ;;
                  esac
                done
              fi

              rm -f "$render_trigger" "$render_def" "$render_input" "$render_s3" "$render_artifacts"
            done <<< "$WORKFLOW_DIRS"
          done
