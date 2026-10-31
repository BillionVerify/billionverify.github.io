# Upload a file

Canonical page: https://billionverify.github.io/docs/api/upload-file/

Upload a CSV, TXT, or XLSX file for asynchronous verification.

```http
POST /verify/file
```

## Request

### Multipart fields

| Field               | Type    | Required | Default     | Description                              |
| ------------------- | ------- | -------- | ----------- | ---------------------------------------- |
| `file`              | file    | Yes      | -           | CSV, TXT, or XLSX file                   |
| `check_smtp`        | boolean | No       | server default | `true` runs a live SMTP check, `false` skips it. When omitted, the server default applies (SMTP check on in production) |
| `email_column`      | string  | No       | auto-detect | Column containing email addresses        |
| `preserve_original` | boolean | No       | `true`      | Keep original columns in the result file |

### Limits

- Maximum file size: 20 GB. A larger file is rejected with `400` and code `3201`.
- No row-count limit.
- Credits are reserved for the unique email addresses in the file. If your balance is too low, the upload returns `402` with code `3000`.
- Small files are parsed during the request and the response includes the email counts. Large files are accepted with `status` set to `preparing` and no counts. Poll the status URL for the counts.

## Response

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "task_id": "7143874e-21c5-43c1-96f3-2b52ea41ae6a",
    "file_name": "emails.csv",
    "file_size": 45678,
    "status": "pending",
    "message": "File uploaded successfully, queued on River workflow",
    "status_url": "https://api.billionverify.com/v1/verify/file/7143874e-21c5-43c1-96f3-2b52ea41ae6a",
    "created_at": "2026-02-04T10:30:00Z",
    "estimated_count": 1000,
    "unique_emails": 950,
    "total_rows": 1000,
    "email_column": "email"
  }
}
```