# Kuwenta – External APIs

| Field | Value |
|---|---|
| Project | Kuwenta – personal finance and expense tracker |
| Document | Third-party APIs: what is called, by which component, with what data, and how failures are handled |
| Version | 1.1 (Draft) – diagram links updated |
| Date | 11 October 2026 |
| Status | Draft – for review |
| Related | kuwenta-requirements.md, kuwenta-tech-stack.md, design/kuwenta-sequence-diagram.drawio.svg |

## 1. Overview

| API | Used for | Called by | Auth | Cost | Status |
|---|---|---|---|---|---|
| Google OAuth 2.0 | Getting and refreshing read-only Gmail access | Gmail connect Lambda, Reconcile Lambda | OAuth client ID + secret, PKCE | Free | MVP |
| Gmail API | Searching and reading receipt emails; sender discovery | Reconcile Lambda | User's OAuth access token (gmail.readonly) | Free (quota-limited) | MVP |
| Gemini API | Extracting transactions from emails; suggesting categories; classifying sender candidates | Reconcile Lambda, Import Lambda | API key | Free tier (local, staging); paid tier (prd) | MVP |
| FX rate API (Frankfurter) | Reference USD→PHP rate when a receipt has no PHP amount | Expenses Lambda | None | Free | Optional |

AWS services (Cognito, DynamoDB, Aurora Data API, KMS, SSM, Secrets Manager) are internal to the architecture and are not covered here.

Facts marked "verified" were checked against the provider's documentation on 11 October 2026. Re-check limits and prices before going live, since providers change them.

## 2. Rules for every external call

- **Secrets from SSM only.** Keys and client secrets are read from SSM Parameter Store at runtime and cached in memory (SEC-09); never in code, Terraform variables or plain environment variables.
- **Timeouts.** Every call has an explicit timeout, kept well under the API Gateway limit of ~30 s (REC-06).
- **Retries.** Retry only on 429 and 5xx, with exponential backoff and jitter, at most 3 attempts; never retry 4xx errors other than 429.
- **No sensitive logging.** Request and response bodies, tokens and email content are never logged (PRV-04); log only status codes, durations and counts.
- **Minimum data.** Send only what the call needs; email text is redacted before it reaches Gemini (REC-03).

| Parameter (per environment) | Holds |
|---|---|
| `/kuwenta/<env>/google/oauth-client-id` | Google OAuth client ID (String) |
| `/kuwenta/<env>/google/oauth-client-secret` | Google OAuth client secret (SecureString) |
| `/kuwenta/<env>/gemini/api-key` | Gemini API key (SecureString) |
| `/kuwenta/<env>/gemini/model` | Model ID, default gemini-3.5-flash-lite (String) |

## 3. Google OAuth 2.0

**Purpose.** Let the user grant Kuwenta read-only access to Gmail, and keep that access working with a refresh token (GML-01, GML-02).

**Scope.** `https://www.googleapis.com/auth/gmail.readonly` (read-only; restricted scope, see Constraints).

| Step | Endpoint | Notes |
|---|---|---|
| Consent | `GET https://accounts.google.com/o/oauth2/v2/auth` | The gmail.readonly scope, `access_type=offline`, `prompt=consent`, `state` (single use, ~10 min TTL in DynamoDB), `code_challenge` + `code_challenge_method=S256` (PKCE). |
| Code exchange | `POST https://oauth2.googleapis.com/token` | `grant_type=authorization_code` with the code and PKCE verifier; returns an access token (short-lived) and a refresh token. |
| Refresh | `POST https://oauth2.googleapis.com/token` | `grant_type=refresh_token`; done at the start of each reconcile. |
| Revoke | `POST https://oauth2.googleapis.com/revoke` | On disconnect (GML-05) and account deletion (PRV-05). |

**Data stored.** Only the refresh token, encrypted with KMS using the user's Cognito ID as encryption context (GML-02). Access tokens are kept in memory for the duration of one request and never stored.

| Failure | Handling |
|---|---|
| `invalid_grant` on refresh (revoked, expired, password changed) | Mark the connection as broken and show a reconnect prompt (GML-04). |
| Missing, expired or reused `state` | Reject the callback (GML-01). |
| User denies consent | Return to the app with a clear message; nothing is stored. |

**Constraints.** gmail.readonly is a restricted scope: an unverified app shows a warning and is capped at 100 users (C-04). Refresh tokens from a Testing-status app expire after 7 days, so prd must be published to In production (C-05). Setup steps are in Appendix C of the requirements.

## 4. Gmail API

**Purpose.** Find and read receipt emails from the user's sender list (REC-02, REC-10, REC-12), and scan sender and subject metadata for sender suggestions (GML-07, GML-08).

**Base URL.** `https://gmail.googleapis.com/gmail/v1`

| Method | Endpoint | Used for | Quota units (verified) |
|---|---|---|---|
| `users.messages.list` | `GET /users/me/messages` | Search with `q` (sender list + `after:` timestamp), paged with `pageToken` | 5 per call |
| `users.messages.get` (`format=full`) | `GET /users/me/messages/{id}` with `format=full` | Read one receipt: headers, text and HTML parts, attachment metadata | 20 per call |
| `users.messages.get` (`format=metadata`) | `GET /users/me/messages/{id}` with `format=metadata` | Sender discovery: only the From and Subject headers (`metadataHeaders`), never the body | 20 per call |

**Rate limits (verified).** 6,000 quota units per minute per user, 1,200,000 per minute per project. At 20 units per message read, one user can read about 300 messages per minute. A reconcile page of ~20 emails uses about 400 units, so normal use is far below the limit.

**Search query.** Built from the sender list (GML-03), for example:

```
(from:noreply-banktransfer@gcash.com OR from:ebanking@metrobank.com.ph OR (from:no_reply@email.apple.com subject:receipt)) after:1759708800
```

`after:` uses a Unix timestamp (exact, no time-zone ambiguity) set to the cursor minus a 2-day overlap (REC-02). Long sender lists are split into several queries.

**Sender discovery cost.** Reading metadata for every email in a large inbox would be slow at 20 units per message (5,000 emails ≈ 17 minutes of quota). The scan therefore narrows candidates first with a Gmail query such as `subject:(receipt OR invoice OR payment OR transaction OR transfer OR statement) after:<start>`, and reads metadata only for those.

**Reading a message.** Prefer the `text/plain` part; otherwise convert `text/html` to text. Bodies are base64url-encoded. PDF attachments are noted but not read in the MVP (REC-08; REC-13 later).

| Failure | Handling |
|---|---|
| 401 | Refresh the access token once and retry; if refresh fails, see OAuth `invalid_grant`. |
| 429 or 403 rate limit | Back off and stop the page early; the app requests the next page (REC-06). Progress is kept. |
| 404 on a message | The email was deleted; mark the ID as processed and skip it. |
| 5xx | Retry with backoff; if it still fails, end the page and let the user retry. |

## 5. Gemini API

**Purpose.** Turn redacted receipt text into structured transactions (REC-04), suggest categories from merchant names (IMP-04), and classify sender candidates (GML-07).

| Item | Value |
|---|---|
| Model | `gemini-3.5-flash-lite` in all environments, configurable per environment (AD-10) |
| Tier | Free in local and staging (fake or sanitized data only); paid in prd (PRV-01) |
| Endpoint | `POST /v1beta/models/{model}:generateContent` on `generativelanguage.googleapis.com`, called through the official `@google/genai` SDK |
| Auth | API key in the `x-goog-api-key` header (set by the SDK) |
| Output | JSON constrained by a JSON schema (structured output) |
| Thinking | Lowest level, "minimal", which is the default for gemini-3.5-flash-lite (verified); thinking tokens are billed as output |
| Price (prd, verified) | $0.30 per 1M input tokens, $2.50 per 1M output tokens; about $0.40 per month at expected volume |

Google also offers a newer Interactions API with the same JSON-schema output. Kuwenta uses whatever the official SDK recommends at build time; the request content and schema below stay the same.

**What is sent.** One request per email: a fixed instruction, the JSON schema, and the email's redacted text (account, card and mobile numbers cut to the last 4 digits, REC-03). Never attachments, Gmail tokens, user IDs or other emails. For statements and sender discovery, only merchant descriptors or sender and subject lines are sent.

**Extraction schema (per transaction, REC-04).**

| Field | Type | Notes |
|---|---|---|
| amount | number | Amount in the original currency |
| currency | string | ISO 4217 code, e.g. PHP, USD |
| php_amount | number or null | PHP actually charged, if shown |
| fee | number or null | Separate from amount |
| merchant | string | Merchant or counterparty |
| occurred_at | string | Date and time from the receipt, not the email date |
| source_account_hint | string or null | Name, alias or last 4 digits |
| destination_account_hint | string or null | For transfers |
| direction | enum | debit, credit, transfer, refund |
| external_ref | string or null | Transaction ID, InstaPay reference, invoice number |
| category | string | Suggested category |
| confidence | number | 0 to 1 |

**Validation in code (REC-11).** The amount must appear in the email text, the date must be within a set window of the email date, and the currency must be a valid ISO code. Items that fail stay pending and are marked for attention.

| Failure | Handling |
|---|---|
| 429 (rate limit; free-tier limits are lower) | Back off and retry; if still limited, end the page early (REC-06). |
| 5xx or timeout | Retry with backoff; then leave the email unprocessed so the next reconcile picks it up. |
| Output fails the schema or validation | Keep as pending, marked for attention; raw output kept 60 days for prompt tuning (PRV-06). |
| Safety block or empty response | Flag the email for manual entry (REC-08). |

## 6. FX rate API (optional)

**Purpose.** Provide a reference USD→PHP rate when a foreign-currency receipt does not show the PHP amount charged (EXP-07). Maya statements already include the rate, so this is a fallback.

| Item | Value |
|---|---|
| Provider | Frankfurter (open source, no API key; verified) |
| Request | `GET https://api.frankfurter.dev/v2/rates?date=YYYY-MM-DD&base=USD&quotes=PHP` |
| Source | By default a blend of central-bank sources; pin a single provider with `providers=` for consistent history (verified) |
| Limits | Rate-limited against abuse, no daily or monthly cap (verified) |
| Caching | Each date's rate is stored in the Aurora `fx_rates` table and fetched at most once |
| Weekends and holidays | Use the most recent earlier rate (EXP-07); the API's own weekend behavior is not documented, so Kuwenta handles it |

| Failure | Handling |
|---|---|
| API unavailable | Leave the PHP amount empty and ask the user to enter it during review. |

## 7. Open points

| ID | Question |
|---|---|
| API-01 | Which FX provider to pin: Bangko Sentral ng Pilipinas (if Frankfurter offers it) or ECB? |
| API-02 | Free-tier Gemini rate limits for local and staging: confirm in Google AI Studio before load-testing the backfill. |
| API-03 | Move to the Gemini Interactions API now or stay on generateContent until the SDK makes it the default? |
