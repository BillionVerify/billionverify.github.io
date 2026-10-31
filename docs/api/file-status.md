# File job status

Canonical page: https://billionverify.github.io/docs/api/file-status/

Get the latest status for a file job.

```http
GET /verify/file/{job_id}
```

## Request

### Query parameters

| Parameter | Type    | Required | Default | Description                               |
| --------- | ------- | -------- | ------- | ----------------------------------------- |
| `timeout` | integer | No       | `0`     | Long-polling timeout in seconds (`0-300`) |

## Download links

- `download_url` is the stable API entrypoint for result downloads.
- `direct_download_url` is a signed storage URL for immediate download.
- `direct_download_expires_at` tells you when that signed storage URL expires.
- `result_file_url` and `result_file_path` are **not** part of the public API contract.
- `result_download_available` tells you whether the result file can be downloaded. `result_download_expires_at` is currently always `null`.

## Response

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "task_id": "7143874e-21c5-43c1-96f3-2b52ea41ae6a",
    "status": "completed",
    "progress": 100,
    "email_progress": 100,
    "chunk_progress": 100,
    "progress_source": "email",
    "total_emails": 1000,
    "processed_emails": 1000,
    "valid_emails": 850,
    "invalid_emails": 100,
    "unknown_emails": 30,
    "role_emails": 15,
    "catchall_emails": 5,
    "risky_emails": 0,
    "disposable_emails": 0,
    "credits_used": 970,
    "download_url": "https://api.billionverify.com/v1/verify/file/7143874e-21c5-43c1-96f3-2b52ea41ae6a/results",
    "direct_download_url": "https://download.example.com/results_7143874e.csv?signature=...",
    "direct_download_expires_at": "2026-02-04T11:32:05Z",
    "result_download_available": true,
    "started_at": "2026-02-04T10:30:00Z",
    "completed_at": "2026-02-04T10:32:05Z",
    "total_chunks": 10,
    "completed_chunks": 10,
    "failed_chunks": 0,
    "can_pause": false,
    "can_resume": false,
    "can_restart": false,
    "restart_attempts": 0,
    "current_attempt": 1,
    "deliverability_buckets": {
      "strict_deliverable": 865,
      "usable_with_catchall": 870,
      "not_deliverable": 100,
      "unknown": 30,
      "anti_verified": 12,
      "transient_failure": 4
    },
    "unique_emails": 950,
    "total_rows": 1000
  }
}
```

Some fields appear only in certain states. For example `download_url` and the download fields are set once the job is `completed`, `stage` and `pending_retry_emails` appear while the job finishes retries, and `error_message` and `pause_reason` appear on failed or paused jobs.

### Job status values

| Status                   | Description                      |
| ------------------------ | -------------------------------- |
| `preparing`              | File is being parsed and staged  |
| `pending`                | Job is queued for execution      |
| `running`                | Workflow has started             |
| `processing`             | Emails are actively being verified |
| `merging`                | Results are being finalized      |
| `paused`                 | Job is paused and can be resumed |
| `completed`              | Job finished successfully        |
| `completed_with_warning` | Job finished, but with warnings  |
| `failed`                 | Job failed                       |
| `cancelled`              | Job was cancelled                |