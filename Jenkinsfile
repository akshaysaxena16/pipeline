stepfunctions/workflows/dummy-ecs-run-task/cert-artifacts.json
{
  "comment": "Dummy artifacts for cert flow demonstration",
  "tasks": [
    {
      "enabled": true,
      "mode": "tar",
      "sourceSqlPath": "artifacts/x/xa/xy/batch.sql",
      "destinationUri": "${DUMMY_ECS_RUN_TASK_ARTIFACTS_DESTINATION_URI}",
      "backupExisting": true,
      "backupTimestampFormat": "%Y%m%d%H%M%S"
    },
    {
      "enabled": true,
      "mode": "copy",
      "sourceFilePath": "artifacts/x/xa/xy/batch.sql",
      "destinationUri": "${DUMMY_ECS_RUN_TASK_ARTIFACTS_DESTINATION_URI}",
      "backupExisting": true,
      "backupTimestampFormat": "%Y%m%d%H%M%S"
    }
  ]
}
