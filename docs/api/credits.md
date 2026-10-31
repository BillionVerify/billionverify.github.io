# Credits

Canonical page: https://billionverify.github.io/docs/api/credits/

Get the current credit balance for the account that owns the API key.

```http
GET /credits
```

## Response

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "account_id": "abc123",
    "api_key_id": "key_xyz",
    "api_key_name": "Default API Key",
    "credits_balance": 9500,
    "credits_consumed": 500,
    "credits_added": 10000,
    "last_updated": "2026-02-04T10:30:00Z"
  }
}
```

| Field | Description |
| ----- | ----------- |
| `account_id` | Account that owns the key |
| `api_key_id`, `api_key_name` | The key used for this request |
| `credits_balance` | Credits available now |
| `credits_consumed` | Total credits used |
| `credits_added` | Total credits added |
| `last_updated` | Last update time of the balance record |