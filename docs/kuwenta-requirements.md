# Kuwenta – Requirements

| Field | Value |
|---|---|
| Project | Kuwenta – personal finance and expense tracker (Gmail + Gemini, on AWS) |
| Document | Product & technical requirements |
| Version | 1.6 (Draft) – links to tech stack and external API docs |
| Date | 11 October 2026 |
| Status | Draft – for review |

## 1. Purpose

There is no open-banking aggregator covering Philippine banks and e-wallets, so Kuwenta builds a single view of spending across every account from two sources:

- **Email receipts in Gmail.** Banks and e-wallets mostly email only fund transfers. Purchase receipts come from merchants (e.g. Apple, Move It, Cebu Pacific) and name the e-wallet or bank used to pay.
- **Monthly statements.** Most e-wallet purchases (QR and in-store payments through Maya or GCash) produce no email at all, so the user uploads the monthly statement to capture them.

Gemini extracts structured transaction data from both sources, and the user reviews it before anything is saved.

The first release serves one user (the owner). The design keeps the door open for other users later without re-architecting.

## 2. Goals

- See total spending per month, per category and per account in one place.
- Capture transactions from Gmail receipts with minimal manual typing.
- Keep a human in the loop: nothing is saved until the user confirms it.
- Run at roughly $5–10 per month, with a hard ceiling of $20 per month.
- Treat financial emails and Gmail access as highly sensitive data.

## 3. Scope

### 3.1 In scope

- Responsive web app (React + Vite), usable from a mobile browser
- Sign-up, sign-in and password reset (Amazon Cognito)
- Connecting a Gmail account with read-only access
- On-demand reconcile: read new receipts, parse with Gemini, review, confirm
- Monthly statement upload (Maya first), parsed in the same review flow
- Accounts, transactions, categories, transfer detection and a spending dashboard
- Three environments (local, staging, prd) managed with Terraform

### 3.2 Out of scope

- Native mobile app (React Native / Expo) – revisit after the web app proves out
- Scheduled or background polling of Gmail
- Public sign-up for other users (needs Google app verification first)
- Budgets, forecasting, bill reminders and investment tracking
- Custom domain (the default CloudFront domain is used for now)

## Diagrams

| Diagram | Shows |
|---|---|
| [kuwenta-aws-prd.drawio.svg](kuwenta-aws-prd.drawio.svg) | Prd architecture: every AWS and external component and how they connect |
| [kuwenta-sequence.drawio.svg](kuwenta-sequence.drawio.svg) | Who calls whom, in order: sign in, connect Gmail, reconcile, statement upload, review and confirm, dashboard |
| [kuwenta-flow.drawio.svg](kuwenta-flow.drawio.svg) | The user-facing process as a flowchart, split into You (browser), Kuwenta (AWS) and external services |

Edit the `.drawio` files in draw.io; the `.svg` files are exports with the diagram embedded, so they open for editing in draw.io too.

## Related documents

| Document | Covers |
|---|---|
| [kuwenta-tech-stack.md](kuwenta-tech-stack.md) | Languages, frameworks, libraries, infrastructure, tooling and repository layout |
| [kuwenta-external-apis.md](kuwenta-external-apis.md) | Google OAuth, Gmail API, Gemini API and FX rate API: endpoints, data sent, limits, costs and failure handling |

## 4. How to read this document

Priorities use MoSCoW. Each requirement is also assigned to the phase in which it must be done.

| Term | Meaning |
|---|---|
| Must | Required for the phase it is assigned to. |
| Should | Important; included unless time runs out. |
| Could | Nice to have. |
| Won't | Explicitly not in this release. |
| MVP | First usable version for the owner. |
| Pre-launch | Required before anyone other than the owner uses the app. |
| Later | After pre-launch, when there is a real need. |

## 5. Functional requirements

### 5.1 Authentication and users (AUTH)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| AUTH-01 | Users sign up with email and password through Amazon Cognito. | A verification email is sent; an unverified user cannot sign in. | Must | MVP |
| AUTH-02 | Users can sign in, sign out and reset a forgotten password. | All three flows work end to end through Cognito with no custom auth Lambda. | Must | MVP |
| AUTH-03 | A user record is created in DynamoDB when sign-up is confirmed. | Post-confirmation trigger writes the record with a conditional put; it ignores ConfirmForgotPassword events and never overwrites an existing record. | Must | MVP |
| AUTH-04 | Sign-up can be limited to an invite allowlist. | Pre-sign-up trigger rejects emails not on the allowlist. | Could | Pre-launch |
| AUTH-05 | Users can turn on multi-factor authentication. | MFA can be enabled in the user pool and enforced per user. | Could | Later |
| AUTH-06 | MVP uses Cognito managed login pages; custom React sign-in forms replace them later. | MVP: sign-up, sign-in and reset go through the managed login (Hosted UI) with the authorization code flow + PKCE. Later: custom forms in the React app call Cognito directly with the same user pool. | Must (managed) / Could (custom) | MVP / Later |

### 5.2 Gmail connection (GML)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| GML-01 | A signed-in user can connect their Gmail account with read-only access. | OAuth flow uses the gmail.readonly scope, a single-use state value stored in DynamoDB with a ~10-minute TTL, and PKCE. A missing, expired or reused state is rejected. | Must | MVP |
| GML-02 | The Gmail refresh token is stored encrypted. | Token is encrypted with the environment's KMS key using the user's Cognito ID as encryption context, and only ciphertext is stored in DynamoDB. | Must | MVP |
| GML-03 | The user keeps a list of senders to read: banks, e-wallets and merchants. | Reconcile only fetches emails from senders on the list, using a Gmail from: query. Each entry has an address or domain, a type (bank, e-wallet, merchant), an optional subject keyword filter (e.g. "receipt" for senders that also send marketing) and an optional default account. The list starts with presets for common Philippine banks, e-wallets and merchants that the user can switch on. Long lists are split across several queries. | Must | MVP |
| GML-04 | The app shows Gmail connection status and asks the user to reconnect when access stops working. | An expired or revoked token produces a clear reconnect prompt, not a silent failure. | Should | MVP |
| GML-05 | The user can disconnect Gmail. | Token is revoked at Google and deleted from DynamoDB. | Should | Pre-launch |
| GML-06 | Supported email sources: GCash bank-transfer emails, Metrobank InstaPay emails, and merchant receipts that name the payment method (e.g. Apple, Move It, Cebu Pacific). BDO, BPI, GoTyme and other senders can be added. | The MVP sources parse correctly against sanitized sample emails. Adding a sender needs only a sender entry and test fixtures, no code change. | Must (MVP sources) / Should (others) | MVP |
| GML-07 | The app suggests senders that look like receipts but are not on the list. | The scan reads only sender and subject (Gmail metadata), never bodies. Keyword rules (receipt, invoice, payment, transfer, transaction, statement) shortlist candidates; Gemini may classify the shortlist. Suggestions appear as "add this sender?" and nothing is added without the user's approval. | Should | MVP |
| GML-08 | The first sender list is built automatically when Gmail is connected; typing senders is optional. | Right after connecting, the app scans sender and subject metadata for the chosen period (REC-10), first narrowing candidates with a keyword Gmail query (receipt, invoice, payment, transaction, transfer, statement) so large inboxes stay within Gmail API quotas. Presets whose senders appear in the inbox are pre-ticked, and other likely receipt senders (GML-07) are listed with an example subject. The user confirms with checkboxes. Adding an address by hand remains available for anything missed. | Must | MVP |

### 5.3 Reconcile (REC)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| REC-01 | Reconcile runs only when the user clicks Reconcile. | No scheduler, cron or polling job exists in any environment. | Must | MVP |
| REC-02 | Each reconcile continues automatically from where the last one stopped, with no gaps. | The user picks a start date only once (REC-10); after that the cursor moves forward by itself and the user never enters a date range. Each run searches from the cursor minus a 2-day overlap, using exact timestamps, so late-arriving emails and time-zone edges are covered; the overlap never creates duplicates because processed message IDs are skipped (REC-09). The cursor is stored per user in DynamoDB. | Must | MVP |
| REC-03 | Account, card and mobile numbers are redacted before any content (email or statement) is sent to Gemini. | Only the last 4 digits of any account, card or mobile number reach Gemini. E-wallet account numbers are often mobile numbers (+63 9xx or 09xx), so those are covered too. Redaction targets labelled account fields and number patterns, not every long digit run, so transaction reference IDs (e.g. Maya's 12-digit card references) stay intact for duplicate matching. | Must | MVP |
| REC-04 | Gemini extracts zero or more transactions from each email. | Uses a structured-output JSON schema returning a list. Each item has: amount, currency, PHP amount charged (if shown), fee (separate from amount), merchant or counterparty, transaction date/time taken from the receipt rather than the email date, source account hint (name, alias or last 4), destination account hint for transfers, direction (debit, credit, transfer, refund), external reference (transaction ID, InstaPay reference), suggested category and a confidence value. An email with no transaction returns an empty list. | Must | MVP |
| REC-05 | Parsed results are stored as pending items in DynamoDB; nothing is written to Aurora yet. | Pending items survive closing the tab and expire after 7 days (PRV-06). Aurora contains no new rows until the user confirms. | Must | MVP |
| REC-06 | Large backlogs are processed in pages. | Each reconcile request finishes within the API Gateway timeout (~30 s); the UI keeps requesting the next page until caught up. | Must | MVP |
| REC-07 | The Gmail cursor advances only after pending items are confirmed or dismissed. | Processed Gmail message IDs are tracked per user, so paging and partial reviews never skip or repeat an email. Abandoning a review does not lose those emails. | Must | MVP |
| REC-08 | Emails that cannot be parsed are flagged for manual entry. | PDF-only or password-protected receipts show as 'needs manual entry' with a link to the email. | Should | MVP |
| REC-09 | Running reconcile again never creates duplicate pending items. | Each pending item is keyed by user + Gmail message ID + line index, so re-processing an email overwrites its existing pending item instead of adding another. Emails whose message ID is already in the processed list (DynamoDB) are skipped before any Gemini call, so Reconcile never needs to wake Aurora. | Must | MVP |
| REC-10 | The user chooses how far back the first reconcile goes. | When connecting Gmail, the user picks a start date (default: start of the current month); it becomes the initial cursor. Only emails from allowlisted senders after that date are searched, using a Gmail query so non-matching emails are never downloaded. The backfill runs in pages (REC-06) and keeps its progress if stopped. Older history is better loaded from statements (IMP-01). | Must | MVP |
| REC-11 | Gemini output is validated in code before it becomes a pending item. | Each extracted amount must appear in the email text, the transaction date must be within a set window of the email date, and the currency must be a valid ISO code. Items that fail are kept as pending but marked for attention, never silently dropped or saved. | Must | MVP |
| REC-12 | A sender added later is backfilled from the original start date. | Adding a sender (or switching on a preset) runs a one-time search for that sender from the account's start date to now, so its earlier emails are not missed. Moving the start date earlier runs the same backfill for the new gap. | Must | MVP |
| REC-13 | Receipt details inside unprotected PDF attachments are read by code. | Text-based, unencrypted PDF attachments are extracted with pdf.js on the server and parsed like email text; protected or image-only attachments stay as "needs manual entry" (REC-08). | Could | Later |

### 5.4 Review and expenses (EXP)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| EXP-01 | The user reviews each pending item and can confirm, edit or dismiss it. | Edited values are saved as edited; dismissed items are not stored as transactions. | Must | MVP |
| EXP-02 | Confirmed items are saved to Aurora without duplicates. | Each row carries user_id, source type (email, statement, manual), source ID (Gmail message ID or statement upload ID) and line index. One email can produce several rows. Uniqueness: (user_id, source_type, source_id, line_index), plus (user_id, account_id, external_ref) when an external reference exists, so the same payment seen in two emails is stored once. | Must | MVP |
| EXP-03 | The user manages their accounts: type (bank, e-wallet, credit card, cash), nickname, optional last 4 digits, and aliases. | Aliases are the names an account appears under in emails and statements (e.g. "GCash", "Maya Wallet", a bank account nickname). Transactions are matched by alias, last 4 digits and sender. When no account matches, the user picks one during review and can save the text as a new alias. | Must | MVP |
| EXP-04 | Transfers between the user's own accounts are detected from a single email. | When the destination matches one of the user's own accounts (by alias, last 4 or the user's own name as recipient), the item becomes a transfer: debit on the source account, credit on the destination account, excluded from spending totals. Any fee is saved as a separate expense (category: Fees). Transfers to other people count as spending unless the user re-categorizes them. If the other side arrives later (e.g. a statement line), it is linked to the existing transfer, not stored again. | Must | MVP |
| EXP-05 | Transactions have categories the user can change. | A default category list exists; re-categorizing updates the dashboard. | Should | MVP |
| EXP-06 | The user can add, edit and delete transactions manually. | Manual entries are marked as manual and behave like parsed ones in reports. | Should | MVP |
| EXP-07 | Foreign-currency transactions (e.g. USD for Claude, Starlink) are converted to PHP. | Each transaction stores the original amount and currency and a PHP amount. If the receipt shows the PHP amount actually charged, that is used. Otherwise the PHP amount is calculated from the reference rate for the transaction date (the most recent earlier rate on weekends and holidays). The rate, its date and its source are stored with the transaction, and the user can override the PHP amount. All reports total in PHP. | Must | MVP |
| EXP-08 | Items from different sources that describe the same payment are merged, not duplicated. | A new item is flagged as a likely duplicate when it has the same external reference as an existing transaction, or the same account and the same amount within ±1 day. Amounts are compared in PHP and also in the original currency when both items have one (e.g. a USD 22.40 Anthropic receipt and a Maya line showing ₱1,398.43 for 22.4 USD). Review offers to merge, keeping the merchant detail from the email and the reference from the statement. | Must | MVP |
| EXP-09 | Manual entries are checked for duplicates too. | Before saving, a manual entry is matched against saved transactions (same account and amount within ±1 day) and the user can open the match, save anyway or cancel. An optional reference number field enables the exact uniqueness rule (EXP-02). A later email or statement item that matches a manual entry is offered as a replacement, keeping the user's category and notes. Each form submission carries a one-time ID so a double-click or retry cannot save it twice. | Should | MVP |
| EXP-10 | Refunds and credit notes reduce spending instead of counting as income. | A refund or credit note (e.g. "Credit note from Anthropic, PBC for invoice …") is saved as a refund in the original category and linked to the original transaction when its invoice or reference matches. Spending totals net out refunds. | Must | MVP |

### 5.5 Dashboard and reporting (DSH)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| DSH-01 | Show monthly spending totals by category and by account. | Totals exclude transfers and match the sum of confirmed debit transactions. | Must | MVP |
| DSH-02 | Show a transaction list with filters. | Filter by date range, account and category; search by merchant. | Should | MVP |
| DSH-03 | Show a 'waking up the database' state while Aurora resumes. | The UI shows status and retries for up to ~30 s instead of showing an error. | Must | MVP |
| DSH-04 | Export transactions to CSV. | Export respects the current filters. | Could | Later |
| DSH-05 | Show what period each source covers. | A coverage view shows the Gmail range reconciled so far and, per account, which statement months are uploaded, highlighting any missing months. | Should | MVP |

### 5.6 Clients (CLI)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| CLI-01 | Responsive web app built with React + Vite. | All MVP screens are usable on a phone-width browser. | Must | MVP |
| CLI-02 | Installable as a progressive web app (PWA). | App can be added to the home screen and opens full screen. | Could | Later |
| CLI-03 | Native mobile app with React Native / Expo. | Not built in this release. | Won't | Later |

### 5.7 Statement import (IMP)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| IMP-01 | The user uploads a monthly statement PDF for an account. | Maya statements are supported in the MVP; GCash transaction history and bank statements follow, each once a sanitized sample exists. | Must (Maya) / Should (others) | MVP |
| IMP-02 | The PDF is opened and its text extracted in the browser. | Uses pdf.js. A password-protected PDF asks for the password in the browser. The password and the raw PDF are never sent to the backend or stored. A PDF with no text layer (e.g. re-saved with "Print to PDF") is detected and the user is told to upload the original; image-only handling depends on Q-08. | Must | MVP |
| IMP-03 | Statement content is redacted before it leaves the browser and again on the server before any Gemini call. | Same rule as REC-03. With a deterministic parser the header (holder name, account number) is never sent at all. | Must | MVP |
| IMP-04 | Statement lines become pending items in the same review flow as email receipts. | Known formats (Maya Wallet first) are parsed in the browser by a deterministic parser, and only the parsed rows (date, type, details, reference, amounts) are sent to the backend. Unknown formats fall back to redacted text parsed by Gemini, in pages that each finish within ~30 s (REC-06). Review, confirm, edit and dismiss work the same as for email items (EXP-01). | Must | MVP |
| IMP-05 | Uploading the same statement twice, or statements with overlapping periods, creates no duplicates. | Each upload stores a hash of the extracted text; a repeat upload is rejected, and overlapping lines are matched by EXP-08. | Must | MVP |
| IMP-06 | The parsed lines are checked against the statement's balances. | If opening balance plus parsed lines does not equal the closing balance, the review shows a warning with the difference. Where the statement has a running balance per line (Maya does), each line is checked too and the first line that breaks the chain is highlighted. | Must | MVP |
| IMP-07 | Statement lines with no counterparty are completed during review. | Maya shows no merchant for QR payments and no sender or recipient for InstaPay lines. These lines are matched to email receipts first (EXP-08); otherwise the user fills in the merchant or picks the own account it went to or came from, which turns the line into a transfer (EXP-04). | Must | MVP |
| IMP-08 | The user is warned when a statement's period overlaps one already uploaded for the same account. | Each upload stores its account and coverage period. Before review, an overlapping upload shows which dates overlap and how many lines already exist as transactions; those lines are pre-marked as duplicates (EXP-02, EXP-08). | Should | MVP |

## 6. Non-functional requirements

### 6.1 Security (SEC)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| SEC-01 | Every API route except the Gmail OAuth callback requires a valid Cognito JWT. | API Gateway JWT authorizer rejects missing, expired or wrongly signed tokens. | Must | MVP |
| SEC-02 | Users can only ever read or change their own data. | Every query filters by user_id, and Postgres row-level security enforces it using SET LOCAL inside a Data API transaction. Tables use FORCE ROW LEVEL SECURITY so the policies also apply to the table owner. | Must | MVP |
| SEC-03 | Only the Reconcile Lambda can decrypt Gmail tokens. | kms:Decrypt is granted to that Lambda's role only; decrypts are visible in CloudTrail. | Must | MVP |
| SEC-04 | Database credentials live in Secrets Manager. | No password appears in code, Terraform variables or Lambda environment variables; the master password is managed by Aurora. | Must | MVP |
| SEC-05 | The app connects to Aurora as a least-privilege database user. | The app user cannot run DDL (CREATE, ALTER, DROP) and is not the table owner; it is separate from the master user, which is used only for migrations. Without this, row-level security (SEC-02) can be bypassed. | Must | MVP |
| SEC-06 | All SQL uses parameterized queries. | No query is built by string concatenation of user input. | Must | MVP |
| SEC-07 | Prd traffic passes through AWS WAF on CloudFront. | Rules: SQL injection managed group (Count first, then Block), Amazon IP reputation list, and a Bearer-header check on /api/* that excludes the public Gmail OAuth callback route. | Must | MVP |
| SEC-08 | API Gateway only accepts traffic that came through CloudFront. | Requests without the secret origin header get 403; the secret is in SSM Parameter Store and can be rotated. | Must | MVP |
| SEC-09 | Third-party secrets live in SSM Parameter Store. | The Gemini API key and the Google OAuth client secret are SecureString parameters, readable only by the Lambdas that need them; none appear in code, Terraform variables or plain environment variables. | Must | MVP |

### 6.2 Privacy (PRV)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| PRV-01 | Gemini uses the free tier in local and staging, and the paid tier in prd. | Local and staging only ever send fake or sanitized receipts to Gemini (see PRV-02). Billing is enabled on the prd Gemini project, so prompts containing real financial data are not used to improve Google's products. | Must | MVP |
| PRV-02 | Staging never sees real financial emails. | Staging uses a dummy Gmail account with fake receipts. | Must | MVP |
| PRV-03 | Full email bodies are not stored. | Only parsed fields and the Gmail message ID are kept in Aurora; raw Gemini output is kept in DynamoDB for 60 days (PRV-06). | Should | MVP |
| PRV-04 | Logs contain no secrets or sensitive content. | Tokens, full account numbers and email bodies never appear in CloudWatch logs. | Must | MVP |
| PRV-05 | A user can delete their account and all their data. | Deletion removes the Cognito user, DynamoDB items, Aurora rows and revokes Gmail access. | Must | Pre-launch |
| PRV-06 | Data is kept only as long as the retention policy allows. | Confirmed transactions: kept until the user deletes them or their account. Raw Gemini output: deleted after 60 days. Pending (unreviewed) items: deleted after 7 days via DynamoDB TTL. Email bodies: never stored. CloudWatch logs: 30 days (MNT-06). Account deletion: purged immediately (PRV-05). | Must | MVP |

### 6.3 Cost (CST)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| CST-01 | Total monthly cost stays within budget. | AWS plus Gemini stays under $20 per month; current estimate is about $13. | Must | MVP |
| CST-02 | No always-on paid components. | No NAT Gateway, RDS Proxy, VPC interface endpoints or always-on database. | Must | MVP |
| CST-03 | Cost alerts are in place. | An AWS Budgets alert at $15 exists on both the staging and prd accounts. | Must | MVP |
| CST-04 | Aurora pauses quickly when idle. | Auto-pause timeout is set near the minimum (~5 minutes) in staging and prd. | Should | MVP |
| CST-05 | DynamoDB stays inside the always-free tier. | Tables use provisioned capacity with low RCU/WCU, not on-demand. | Should | MVP |

### 6.4 Performance and availability (PRF)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| PRF-01 | The dashboard loads quickly when the database is awake. | Under 3 s when warm; under ~20 s from a paused database, with status shown. | Should | MVP |
| PRF-02 | Reconcile requests stay within API Gateway limits. | No reconcile request runs longer than ~30 s (see REC-06). | Must | MVP |
| PRF-03 | Single-region deployment with no high-availability requirement. | Runs in ap-southeast-1 (Singapore) with one Aurora writer. | Must | MVP |

### 6.5 Maintainability and operations (MNT)

| ID | Requirement | Acceptance criteria | Priority | Phase |
|---|---|---|---|---|
| MNT-01 | Backend is written in Node.js with TypeScript. | All Lambdas share types and a data-access layer. | Must | MVP |
| MNT-02 | Infrastructure is defined in Terraform. | One module set, separate remote state per environment; no manual console changes in staging or prd. | Must | MVP |
| MNT-03 | Three environments: local, staging, prd. | Staging and prd are separate AWS accounts with their own Cognito pool, KMS key, Gemini key and Google OAuth client. | Must | MVP |
| MNT-04 | Local development needs no AWS account. | Docker Compose runs Postgres 16 and DynamoDB Local; the query builder uses the pg driver locally and the Data API in AWS. | Must | MVP |
| MNT-05 | Integration tests run against staging. | Tests cover Data API behavior (types, transactions) before changes reach prd. | Should | MVP |
| MNT-06 | Logs have a retention period. | CloudWatch log groups keep logs for a set period (e.g. 30 days). | Should | MVP |

## 7. Constraints and assumptions

### 7.1 Constraints

| ID | Constraint |
|---|---|
| C-01 | Budget: target $5–10 per month, ceiling $20 per month across all environments. |
| C-02 | Region: AWS ap-southeast-1 (Singapore). The CloudFront WAF web ACL must be created in us-east-1. |
| C-03 | No custom domain for now, so the API Gateway default endpoint stays enabled and is protected by the secret origin header. |
| C-04 | gmail.readonly is a restricted Google scope. An unverified OAuth app is capped at 100 users and shows an "unverified app" warning on the consent screen; public use needs Google verification and a yearly security assessment (CASA). The Google Cloud project and OAuth client are set up once per environment by the app owner (Appendix C); users only grant consent. |
| C-05 | OAuth clients in Testing status issue refresh tokens that expire after 7 days. Prd must be published to In Production; staging re-consents weekly. |
| C-06 | AWS WAF cannot attach to API Gateway HTTP APIs, so WAF sits on CloudFront in front of the API. |
| C-07 | Aurora takes about 15 seconds to resume from auto-pause. |

### 7.2 Assumptions

| ID | Assumption |
|---|---|
| A-01 | One user (the owner) at first, with about 200 receipt emails per month. |
| A-02 | Banks and e-wallets mostly email only fund transfers (GCash bank transfers, Metrobank InstaPay). Purchase receipts come from merchants and name the payment method in the body (e.g. "Paid by GCash", "Payment Method: Maya"). Maya does not email purchase receipts, and GCash emails only some. QR and in-store e-wallet purchases appear only in monthly statements. |
| A-03 | A ~15-second wait when the database is asleep is acceptable for personal use. |
| A-04 | Transactions are mostly in Philippine pesos (PHP), with some card purchases in USD (e.g. Claude, G2A). Starlink bills in PHP through a local processor. PHP is the only reporting currency. |
| A-05 | Most day-to-day spending goes through Maya and GCash. |

## 8. Architecture decisions

| ID | Decision | Rationale | Alternatives rejected |
|---|---|---|---|
| AD-01 | Reconcile on demand, triggered by the user. | No scheduler or polling; cuts Lambda and Gemini cost and lets the database sleep. | Scheduled polling every 5–15 minutes. |
| AD-02 | Aurora PostgreSQL Serverless v2 (16.3+), min 0 ACU, for expenses. | Full Postgres for reporting queries; scales to zero when idle. | Aurora DSQL (no foreign keys, sequences or extensions); RDS started per click (slow, auto-restarts after 7 days); always-on Serverless v2 (~$44/month). |
| AD-03 | All Lambdas outside the VPC; Aurora accessed through the RDS Data API. | Avoids NAT Gateway and VPC endpoint costs; Data API wakes a paused database. | Lambda in VPC with NAT (~$30+/month); split VPC/non-VPC Lambdas (needs a paid Secrets Manager endpoint). |
| AD-04 | No SQS for now. | SQS is a queue, not storage (14-day max retention); not needed at single-user volume. | SQS as data store; SQS fan-out (revisit if reconcile goes async). |
| AD-05 | DynamoDB for users, settings, sender allowlist, Gmail cursor, encrypted tokens, OAuth state, pending items and raw Gemini output. | Always-on and in the free tier, so login, reconcile and imports never wake Aurora. DynamoDB TTL enforces retention (PRV-06) without any scheduled job (REC-01). | Storing everything in Aurora; a cleanup cron job. |
| AD-06 | Amazon Cognito for identity. | Sign-up, sign-in, reset and MFA built in; free up to 10,000 monthly active users. | Custom auth Lambdas. |
| AD-07 | CloudFront in front of S3 and the HTTP API, with WAF on prd only. | Keeps the cheaper HTTP API while adding SQLi, IP reputation and header rules. | REST API with WAF attached directly; CloudFront flat-rate plans. |
| AD-08 | One KMS key per environment for Gmail tokens, with user ID as encryption context. | Application-level encryption, tight decrypt permissions and an audit trail for $1/month. | Relying only on DynamoDB at-rest encryption; one key per user. |
| AD-09 | Secrets Manager for Aurora credentials (master + app user); SSM Parameter Store for the Gemini key and Google OAuth client secret. | Required by the Data API; Aurora can manage and rotate the master password. SSM SecureString is free for the other secrets. | Passwords in environment variables; Secrets Manager for every secret ($0.40 each). |
| AD-10 | Gemini 3.5 Flash-Lite everywhere: paid tier in prd, free tier in local and staging. | Free-tier prompts may be used to improve Google's products, so only fake or sanitized data goes to it. Paid-tier prompts are not used that way, so real receipts only go to prd. | Gemini free tier everywhere; paid tier everywhere. |
| AD-11 | Staging and prd in separate AWS accounts, managed by Terraform. | Isolation of real financial data and separate billing and permissions. | Single account with environment tags. |
| AD-12 | Web app first; PWA and native app later. | Responsive web covers most needs at the lowest cost. | Building React Native from day one. |
| AD-13 | Cognito managed login pages for the MVP; custom React forms later. | No auth UI to build or secure for the MVP; the custom UI can come later without changing the user pool. | Custom React forms from day one. |
| AD-14 | Store original currency plus a PHP amount; report in PHP only. | Covers USD subscriptions now and leaves room for multi-currency reporting later. The PHP amount actually charged (from the receipt) beats a reference rate because it includes bank FX markup. | PHP-only amounts; full multi-currency reporting now. |
| AD-15 | Two input sources (email receipts and statement uploads) feed one review flow, with cross-source duplicate matching. | Email alone misses most e-wallet purchases; statements alone lack merchant detail. Matching lets each source fill the other's gaps. | Email only; statements only. |
| AD-16 | Statement PDFs are read in the browser, not uploaded. | The PDF password and the raw statement never leave the device, and no S3 upload path or PDF parsing Lambda is needed. | Upload to S3 and parse in a Lambda; send the PDF directly to Gemini (skips redaction). |
| AD-17 | Drizzle ORM for database access and migrations. | Official drivers for both the RDS Data API (AWS) and node-postgres (local); table definitions double as shared TypeScript types (MNT-01); migrations are plain SQL. | Kysely and Knex (community Data API adapters; Knex has weak typing); Prisma (community adapter only, Prisma Migrate cannot run over the Data API). |
| AD-18 | Deterministic in-browser parsers for known statement formats; Gemini only as a fallback and for categorizing merchant descriptors. | The Maya statement is regular enough to parse exactly, and the running-balance check proves every line. This costs nothing, keeps statement content on the device, and avoids Gemini misreading amounts. Gemini still suggests categories from merchant descriptors such as "DLO*StarlinkPhilippi", which contain no personal data. | Gemini for every statement. |
| AD-19 | A separate Import Lambda for statement rows. | Keeps kms:Decrypt confined to the Reconcile Lambda (SEC-03) and keeps each function small. | Adding imports to the Reconcile or Expenses Lambda. |

## 9. Cost estimate (monthly, USD)

| Item | Env | Monthly | Basis |
|---|---|---|---|
| AWS WAF | prd | $8.00 | $5 web ACL + 3 rules × $1 |
| KMS | prd | $1.00 | One customer-managed key |
| Aurora Serverless v2 | prd | $0.60 | ~2.5 ACU-hours, <1 GB storage, I/O in cents |
| Secrets Manager | prd | $0.80 | Two secrets: Aurora master and least-privilege app user (SEC-05) |
| Gemini API | prd | $0.30 | ~400k input / 60k output tokens; Flash-Lite to Flash range |
| API Gateway, Data API, SES, S3 | prd | $0.10 | Per-request pricing at a few thousand requests |
| Lambda, DynamoDB, CloudFront, Cognito, CloudWatch, SSM | prd | $0.00 | Always-free tiers at this volume |
| KMS, Secrets Manager, Aurora, Gemini | staging | $2.00 | Same as prd without WAF; test usage only |
| Docker Postgres + DynamoDB Local | local | $0.00 | Runs on the developer machine |
| Total – prd |  | $10.80 |  |
| Total – staging |  | $2.00 |  |
| Total – all environments |  | $12.80 | Budget ceiling: $20.00 |

Estimate assumes one user, ~200 receipt emails per month and Aurora awake ~5 hours per month. Prices mix AWS published rates and figures recalled from memory; treat as accurate to about $1. Aurora always-on (multiple active users) would add roughly $44/month or more.

## 10. Risks

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R-01 | Google verification and CASA for gmail.readonly take weeks and cost money before others can use the app. | High | Stay under 100 test users; research forwarding receipts to an Amazon SES inbox instead of Gmail OAuth. |
| R-02 | Bank and e-wallet email formats vary or change without notice. | Medium | Per-sender handling, confidence scores, mandatory user review, manual entry fallback. |
| R-03 | Gemini extracts wrong amounts or directions. | Medium | Nothing saved without user confirmation; log low-confidence results for prompt tuning. |
| R-04 | WAF SQL injection rules block legitimate receipt text. | Medium | Run the SQLi group in Count mode first; exclude specific rules that misfire. |
| R-05 | Costs jump when several users keep Aurora awake. | Medium | Budget alerts; revisit database sizing before opening to others. |
| R-06 | A Gmail refresh token is revoked or expires. | Low | Detect on reconcile and prompt the user to reconnect (GML-04). |
| R-07 | Cold-start wait annoys the user. | Low | Clear 'waking up' state; warm the database when the dashboard loads. |
| R-08 | "Kuwenta" is a common Tagalog word, so it may be hard to register as an exclusive trademark. No existing app or product named Kuwenta was found in a web search (October 2026). The earlier name "Kwenta" was dropped because a crypto derivatives trading platform already uses it. | Low | Fine for personal use; search the IPOPHL trademark database before any public launch. |
| R-09 | The same payment arrives from two sources (email and statement, or two emails) and is counted twice. | Medium | External-reference uniqueness (EXP-02) and duplicate matching in review (EXP-08). |
| R-10 | Statement layouts change, or a statement is a scanned image with no extractable text. | Medium | Balance check (IMP-06); fall back to manual entry; keep sanitized sample statements as test fixtures. |
| R-11 | Sample receipts and statements contain real personal data. | Medium | Originals stay out of git (docs/sample-receipts is in .gitignore) and are never sent to the free Gemini tier; tests use sanitized copies only. |

## 11. Open questions

| ID | Question | Status |
|---|---|---|
| Q-01 | Which bank or e-wallet is supported first (e.g. BDO, BPI, GCash, Maya)? | Resolved: Maya and GCash first, then BDO, BPI, Metrobank, GoTyme and others (GML-06). |
| Q-02 | Cognito managed login pages or custom React forms? | Resolved: managed login for the MVP, custom React forms later (AUTH-06, AD-13). |
| Q-03 | Which Gemini model gives acceptable accuracy at low cost? Can the free tier be used? | Resolved: gemini-3.5-flash-lite in every environment (free tier in local and staging, paid tier in prd), so prompts are tuned on the same model that runs in prd. The model ID stays configurable per environment (AD-10). |
| Q-04 | Kysely, Drizzle, Prisma or Knex as the query builder / ORM, and does its Data API driver cover our needs? | Resolved: Drizzle (AD-17). |
| Q-05 | Is multi-currency needed, or PHP only? | Resolved: PHP reporting only; USD transactions converted at the transaction-date rate or the actual charged amount (EXP-07, AD-14). |
| Q-06 | What is the data retention policy for transactions and stored Gemini output? | Resolved: see PRV-06 (Gemini output kept 60 days). |
| Q-07 | Confirm Aurora, Data API and API Gateway prices for ap-southeast-1. | Deferred: not needed for now. |
| Q-08 | Is the original Maya statement PDF text-based, and is it password-protected? | Resolved: password-protected with a clean text layer. pdf.js opens it with the password and a deterministic parser read all 24 lines of the July 2026 sample with every running balance and summary total matching (AD-18). |


## Appendix A. Sample receipt findings

Based on the sample emails in `docs/sample-receipts/` (kept out of git). Personal details are redacted here: account and mobile numbers show only the last 4 digits, and names, addresses and reference numbers are omitted.

| Sample | Sender | Contents | Paid from | What it shows |
|---|---|---|---|---|
| GCash bank transfer | GCash | ₱30,000 transfer + ₱10 fee to the user's own Maya wallet (account ••••5988) | GCash | Own-account transfer seen in one email; fee is a separate expense (EXP-04); account number is a mobile number (REC-03). |
| Metrobank InstaPay | Metrobank | ₱46,900 from the user's payroll account to the user's Maya wallet | Metrobank | Accounts appear by nickname, not number, so aliases are needed (EXP-03). |
| Apple receipt | Apple (merchant) | ₱29 monthly app subscription, VAT inclusive | GCash (shown only as "Invoice.GCASH") | Merchant sender (GML-03); invoice date (7 Oct) differs from email date (8 Oct) (REC-04). |
| Move It receipt | Move It (merchant) | ₱84 ride (fare ₱77 + ride cover ₱7) | GCash ("Paid by GCash") | Payment method named in the body; only the total is needed. |
| Cebu Pacific itinerary | Cebu Pacific (merchant) | Two payments in one email: ₱7,327.24 on 18 Sep and ₱2,447.20 on 2 Oct, each with its own transaction ID | Maya | One email can hold several transactions on different dates; an earlier email may repeat the first payment, so external references must be unique (EXP-02). |

## Appendix B. Sample Maya Wallet statement findings

Based on a July 2026 Maya Wallet statement (kept out of git). Account number shown as last 4 only; names and reference IDs omitted.

**Layout.** Header: account holder, account number (the user's mobile number, ••••5988), coverage period. Account summary: starting balance ₱481.85 + incoming ₱69,040.00 − outgoing ₱48,558.32 = ending balance ₱20,963.53 (adds up exactly). Transaction table: date and time, transaction type and details, reference ID, signed amount, running balance. 4 pages, 24 lines for the month.

| Transaction type | Example details | What it means for Kuwenta |
|---|---|---|
| Purchased using card | APPLE.COM/BILL; DLO*StarlinkPhilippi; GOOGLE *YouTubePremium; Netflix.com | Card merchant descriptors need cleaning into merchant names (e.g. "DLO*StarlinkPhilippi" → Starlink). Most of these are recurring subscriptions. |
| Purchased using card (foreign) | ANTHROPIC* CLAUDE SUB — 22.4 USD, rate 62.42991, ₱1,398.43; G2A.COM — 20.02 USD, rate 62.7003, ₱1,255.26 | The statement gives the USD amount, the rate and the PHP actually charged, which is exactly what EXP-07 needs. |
| Purchased | "Pay with PayMaya Wallet QR Off Us"; "PWM Authorization" | No merchant name; the user fills it in (IMP-07). |
| Cashed in | "Receive Bank Transfer fr Maya Bank" | Money in from Maya Bank; likely the user's own savings account, so a transfer. |
| Received from bank | "Credit via InstaPay" | No sender shown; match to an email (e.g. a Metrobank InstaPay email) or the user picks the source account. |
| Sent to bank | "Instapay Sending" | No recipient shown. Amounts such as ₱32,015 and ₱2,010 suggest the InstaPay fee is included in the amount, not listed separately. |

**Reference IDs.** Card purchases have 12-digit numeric references; other types have 12-character hex references. Both are stored as the external reference (EXP-02) and must survive redaction (REC-03).

**File format.** The original statement is password-protected and has a clean text layer (a copy re-saved with "Print to PDF" had turned every page into an image, so the original must be used). Opened with pdf.js, a deterministic parser read all 24 lines; every running balance matched, and the totals matched the summary exactly (incoming ₱69,040.00, outgoing ₱48,558.32, ending ₱20,963.53).

## Appendix C. Google Cloud setup (one-time, by the app owner)

Done once per environment (one Google Cloud project for staging, one for prd). Users never touch Google Cloud: they only click Allow on the consent screen when connecting Gmail, and can revoke access at any time in their Google Account under Security → Third-party connections.

| Step | What to do | Notes |
|---|---|---|
| 1 | Create a Google Cloud project and enable the Gmail API. | Separate projects keep staging and prd credentials apart (MNT-03). |
| 2 | Configure the OAuth consent screen: app name Kuwenta, user type External, scope gmail.readonly, add test users. | While in Testing, only listed test users can connect. |
| 3 | Create an OAuth client (Web application) with redirect URI https://<cloudfront-domain>/api/gmail/callback, plus a localhost URI for local development. | Store the client secret in SSM Parameter Store (SEC-09). |
| 4 | Prd only: publish the app to In production. | Testing-mode refresh tokens expire after 7 days (C-05). Unverified apps show a warning and are capped at 100 users (C-04). |
| 5 | Create a Gemini API key in Google AI Studio; enable billing on the prd project. | Free tier in local and staging, paid tier in prd (PRV-01). Store the key in SSM (SEC-09). |
