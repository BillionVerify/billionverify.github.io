# Verification Reasons

Canonical page: https://billionverify.github.io/docs/verification-reasons/

Every verification result includes a `reason` field that explains why the email received its status. Use this reference to build precise filtering logic, diagnose unexpected results, and tune your sending strategy.

## How reasons work

The `reason` field appears in verification responses alongside the `status` field. It can be absent on some results:

```json
{
  "email": "user@example.com",
  "status": "valid",
  "reason": "smtp_deliverable"
}
```

Reasons are stable string identifiers — you can use them directly in filtering rules or downstream conditional logic.

## How each mailbox type is verified

The reason you get depends on how the recipient's provider can be checked:

- **Business domains** (Google Workspace, Microsoft 365, self-hosted mail servers) get a live SMTP RCPT check against the exact mailbox. A random address on the same domain is probed first, so catch-all domains, Microsoft 365 internal relay, and security gateways in accept-all mode are reported as `catchall` rather than `valid`.
- **Gmail** gets a live SMTP RCPT check.
- **Yahoo, AOL, Outlook.com, Hotmail, Live, and MSN** block SMTP verification, so they are checked through provider-specific account lookups. These return `yahoo_api_*` or `microsoft_api_*` reasons and confirm whether the account exists.
- **GMX, web.de, Mail.ru, Yandex, and other regional providers** are verified with provider-specific pacing. Rate limits, greylisting, and refused checks lead to more `unknown` results.
- **Apple iCloud, Proton, and Tuta** allow no reliable check, so their addresses always return `unknown` with `smtp_unverifiable`.

Results with `status: "unknown"` are never billed, whatever the reason. Addresses with `invalid_syntax` are never billed either.

## Reasons by status

### Valid

Emails with `status: "valid"` are confirmed deliverable.

| Reason | Description |
| ------ | ----------- |
| `smtp_deliverable` | SMTP RCPT returned 250 OK — mailbox confirmed deliverable. Also produced when Microsoft 365 DBEB mode confirms delivery. |
| `yahoo_api_verified` | Yahoo or AOL account lookup confirmed the account exists. |
| `microsoft_api_verified` | Microsoft consumer account lookup (Outlook.com, Hotmail, Live, MSN) confirmed the account exists. |
| `unitedinternet_api_verified` | GMX.com / mail.com account lookup confirmed the account exists. |

### Invalid

Emails with `status: "invalid"` should be removed from your list — delivery is not possible.

| Reason | Description |
| ------ | ----------- |
| `invalid_syntax` | Email address fails syntax parsing — malformed local part, domain, or missing `@`. Never billed. |
| `no_mx_records` | Domain has no MX records (NXDOMAIN or empty MX lookup result). |
| `null_mx` | Domain publishes a null MX record (RFC 7505) — it explicitly accepts no mail. |
| `spf_reject_all` | Domain has no MX records and an SPF record with `-all` — the domain explicitly rejects all inbound mail. |
| `mailbox_not_found` | SMTP 550/5xx confirmed the mailbox does not exist on the remote server. |
| `mailbox_disabled` | Mailbox exists but has been disabled or deactivated. |
| `host_not_found` | The MX hostname did not respond — host unreachable or non-existent. |
| `yahoo_api_not_deliverable` | Yahoo or AOL account lookup confirmed the account does not exist. |
| `microsoft_api_not_deliverable` | Microsoft consumer account lookup confirmed the account does not exist. |
| `unitedinternet_api_not_deliverable` | GMX.com / mail.com account lookup confirmed the account does not exist. |

### Unknown

Emails with `status: "unknown"` could not be conclusively verified. Unknown results are never billed. Some are temporary (retryable); others represent structural limitations.

| Reason | Retryable | Description |
| ------ | --------- | ----------- |
| `smtp_unverifiable` | No | The provider allows no reliable SMTP verification (Apple iCloud, Proton, Tuta). |
| `smtp_blocked` | No | The remote SMTP server explicitly blocked or blacklisted the connection. |
| `smtp_eof_blocked` | Yes | The MX closed the connection before sending a banner — typically a per-IP rate limiter or fail2ban rule. |
| `smtp_timeout` | Yes | SMTP operation timed out (context deadline or cancellation). |
| `smtp_connection_failed` | Yes | Generic SMTP connection failure — no more specific class matched. |
| `smtp_rate_limited` | Yes | Remote SMTP server returned a rate-limit response (421, 450, 451, or messaging limit exceeded). |
| `smtp_greylisted` | Yes | Generic greylisting response detected — retry after a delay. |
| `smtp_550_unclassified` | Yes | The server returned a 550 that does not clearly mean "user unknown", so it is not treated as invalid. |
| `smtp_egress_unavailable` | Yes | No verification egress was available for this provider at the time of the check. |
| `smtp_egress_unselectable` | No | No verification egress is allowed to reach this provider. |
| `mimecast_greylist` | Yes | Mimecast-specific greylist / 451 temporary failure. |
| `proofpoint_ad_lookup` | Yes | Proofpoint AD async lookup in progress (421 4.1.1 temp-fail during recipient validation). |
| `proofpoint_egress_listed` | No | Proofpoint-protected domain where our checks are currently not answered. |
| `policy_temp_fail` | Yes | Generic 4xx temporary failure from remote policy enforcement. |
| `google_rate_limit` | Yes | Google MX returned a rate-limit or temporary rejection response. |
| `google_dmarc_misalignment` | No | Domain DMARC configuration is misaligned — Google rejects based on DMARC policy. |
| `m365_ip_rep_block` | Yes | Microsoft 365 rejected the connection due to IP reputation (4.7.650 / 4.7.651 codes). |
| `tonline_egress_not_whitelisted` | No | T-Online only accepts checks from allowlisted senders, so the mailbox cannot be verified. |
| `mailbox_full` | No | Mailbox exists but is over quota (SMTP 452 or full inbox response). |
| `dns_timeout` | Yes | DNS or MX lookup timed out — transient resolver issue. |
| `dns_lookup_error` | Yes | DNS lookup failed with a non-NXDOMAIN error (SERVFAIL, UDP timeout, or resolver error). |
| `dns_resolution_failed` | Yes | DNS resolution failed for a transient reason. |
| `yahoo_api_error` | Yes | Yahoo / AOL account lookup failed — general error. |
| `yahoo_api_unavailable` | Yes | Yahoo / AOL account lookup was unavailable (batch-level health failure). |
| `yahoo_api_eof` | Yes | Yahoo / AOL account lookup returned early EOF — transient TCP/TLS drop. |
| `yahoo_api_rate_limited` | Yes | Yahoo / AOL account lookup was rate-limited. |
| `microsoft_api_error` | Yes | Microsoft consumer account lookup failed or was throttled. |
| `microsoft_api_unavailable` | Yes | Microsoft consumer account lookup was unavailable. |
| `unitedinternet_api_error` | Yes | GMX.com / mail.com account lookup failed. |
| `unitedinternet_api_unavailable` | No | GMX.com / mail.com account lookup was unavailable for this run. |
| `capacity_deferred` | Yes | The provider's verification capacity was full, so the check was deferred. |
| `provider_capacity_exhausted` | No | The provider's capacity stayed full for the whole retry window (about two hours), so the job finished without a conclusion for these addresses. |
| `carrier_blocked` | No | Japanese carrier domain (docomo.ne.jp, au.com, etc.) — SMTP is blocked by carrier policy. |
| `internal_verifier_error` | Yes | Unclassified internal verifier error — transient failure. |
| `rate_limited` | Yes | Email was rate-limited during batch processing (alias for `smtp_rate_limited` in file jobs). |
| `bucket_default_skip_smtp_unknown` | No | Account-level verification mode is set to skip SMTP — result is based on domain checks only. |
| `mx_only_verification_semaphore_timeout` | Yes | SMTP concurrency semaphore timed out — MX-only fallback confirmed valid MX records but the mailbox could not be individually verified. |

### Catch-all

Emails with `status: "catchall"` belong to domains that accept mail for all addresses. Individual deliverability cannot be confirmed.

| Reason | Description |
| ------ | ----------- |
| `catch_all_domain` | Domain confirmed as catch-all — random RCPT probe accepted by the mail server. |
| `catch_all_deliverable` | Catch-all domain, and the specific address also returned 250 OK on RCPT. |
| `catch_all_inferred` | Catch-all status inferred from cached evidence (indirect, not a live probe). |
| `m365_internal_relay` | Microsoft 365 domain configured as internal relay — all RCPT accepted but not individually verified. |
| `gateway_accept_all` | Security gateway (Mimecast, Proofpoint, etc.) is in accept-all mode. |
| `forwarding_alias` | Email forwarding alias service (SimpleLogin, Firefox Relay, Duck.com). |
| `yahoo_isp_accept_all` | ISP domain hosted on Yahoo mail infrastructure (for example att.net, sbcglobal.net, verizon.net) — the provider accepts all recipients, so no mailbox-level check is possible. |
| `region_consumer_accept_all` | Regional consumer domain known to accept all inbound mail (older results). |
| `bucket_default_skip_smtp_catchall` | Account-level verification mode is set to skip SMTP catch-all detection. |

### Role

Emails with `status: "role"` belong to shared inboxes (e.g. `info@`, `support@`, `noreply@`). These addresses are typically deliverable but have low engagement.

Role addresses **inherit the `reason` from their underlying check** — no dedicated reason codes exist for this status. For example, a role address that passes SMTP will show `reason: "smtp_deliverable"`.

A role address is reported as `role` even when the mailbox itself could not be checked, so its `reason` can be one of the `unknown` reasons above. It is charged like any other `role` result.

### Disposable

Emails with `status: "disposable"` belong to temporary email services. Accepting these usually leads to low-quality signups.

| Reason | Description |
| ------ | ----------- |
| `disposable_domain` | Domain is listed as a known disposable / temporary email provider. |

### Risky (older results only)

New verifications no longer return `status: "risky"`. Its only source, the Spamhaus Domain Blocklist check, was removed on 2026-09-11. You may still see `risky` on results created before that date.

On those older results, `risky` meant the address was most likely deliverable but the recipient domain was listed by Spamhaus for spam, phishing, malware, or botnet C&C activity. The `risk_reasons` array explains why:

| `risk_reasons` value | Description |
| -------------------- | ----------- |
| `spamhaus_dbl_spam` | The domain is listed as a spam source. |
| `spamhaus_dbl_phish` | The domain is listed as a phishing domain. |
| `spamhaus_dbl_malware` | The domain is listed as a malware distribution domain. |
| `spamhaus_dbl_botnet_cc` | The domain is listed as botnet command-and-control infrastructure. |

## Using reasons in filtering

You can combine `status` and `reason` for precise list segmentation. For example, to keep `unknown` emails that are only unknown due to temporary issues:

```javascript
function isRetryableUnknown(result) {
  const retryableReasons = new Set([
    'capacity_deferred', 'context_cancelled', 'dns_lookup_error',
    'dns_resolution_failed', 'dns_timeout', 'google_rate_limit',
    'internal_verifier_error', 'm365_ip_rep_block', 'microsoft_api_error',
    'microsoft_api_unavailable', 'mimecast_greylist',
    'mx_only_verification_semaphore_timeout', 'policy_temp_fail',
    'proofpoint_ad_lookup', 'rate_limited', 'smtp_550_unclassified',
    'smtp_connection_failed', 'smtp_egress_unavailable', 'smtp_eof_blocked',
    'smtp_greylisted', 'smtp_rate_check_failed', 'smtp_rate_limited',
    'smtp_rate_wait_failed', 'smtp_timeout', 'unitedinternet_api_error',
    'yahoo_api_eof', 'yahoo_api_error', 'yahoo_api_rate_limited',
    'yahoo_api_unavailable',
  ]);
  return result.status === 'unknown' && retryableReasons.has(result.reason);
}
```