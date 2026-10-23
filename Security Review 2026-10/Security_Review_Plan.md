# Plan: LH / ERPNext Security Posture Review (TJ Slack thread, 2026-10-22)

## Context
TJ asked Viral (thread C0A1Z6XQE7M / p1792672088431809, cc Arth and Srishti) to own security across six areas:
infra, supply chain, integrations, credentials, deployment and monitoring/recovery. The Frappe Cloud firewall alone is not enough.
**Hard deadline: reply in the thread by 2026-10-23 19:00 IST** with:
1. The real exposure, including this week's incidents, and what has been checked.
2. A plan covering all six areas, with the report date.
3. How the reviews will keep running.

Dates committed are binding. Critical findings must be flagged straight away.

TJ's reply 1 includes an image (IMG_0302.jpg, "Run this too") that cannot be read from here. **Viral has to open it and add whatever it asks for to the plan before sending.**

## Deliverables (implementation = writing these files)
New folder: `/home/frappe/frappe-bench/LH Documentation/Security Review 2026-10/`

| File | Purpose |
|---|---|
| `00_Slack_Reply_2026-10-23.md` | Paste-ready reply to TJ covering points 1–3. Short and direct, with dates. |
| `01_Initial_Exposure_Findings.md` | What was checked on 2026-10-23: evidence (file:line), severity and the immediate action for each finding. |
| `02_Security_Review_Plan.md` | All six areas plus extras: what gets checked, how, the exit criteria, and the date for each phase. |
| `03_Ongoing_Security_Process.md` | Recurring cadence, vulnerability monitoring, incident runbook and ownership. |

The docs never contain secret values. Masked prefixes only (e.g. `shpss_****`).

## Content of 00 – Slack reply (outline)
- **This week's incidents vs us:**
  - LiteLLM 1.107.0 PyPI/Docker compromise (Oct 21): not installed in our bench, so not affected.
  - ERPNext CVE batch (Oct 13: SQLi and access-control bugs, fixed in 15.121.3): local dev runs frappe 15.121.1, so it is affected. Production's version is being confirmed on Frappe Cloud, with an update scheduled if it is affected.
  - MCP SDK advisories: our `mcp` is 2.2.0, which is the patched version. The OpenCode/MCP OAuth RCE does not apply because we don't use OpenCode.
  - GhostAction GitHub campaign: our repos still need checking.
- **Already checked:** a code and secrets scan of all custom apps, a webhook auth review, guest endpoints, dependencies, CI, data on the dev box, and the MCP connector (audited earlier, 28/28 fixed in Aug).
- **Plan and dates:** see the phases below.
- **Ongoing process:** see 03.

## Content of 01 – Initial findings (already verified today)
**Critical**
- C1: Shopify client secret hardcoded in `apps/lh/lh/patches/add_shopify_client_secret_field.py:24`, tracked in `lyfehardware/lh-erpnext`. Anyone who can read the repo can forge Shopify webhooks.
- C2: Shopify Admin token `shpat_****` in `lh` git history (commits 60b91113 and aaf02843).

**High**
- H1: Slack webhook hardcoded in `apps/lh/lh/patches/add_us_warehouse_delivered_slack_webhook.py:3`, plus older webhooks in history.
- H2: 1Click API keys in the tracked `apps/lh/lh/docs/oneclick_postman_collection.json`.
- H3: `update_all_real_time_tracking` is guest-callable (`shipment_analysis_dashboard.py:200`). Anyone can trigger up to 200 carrier API calls (cost and DoS).
- H4: Webhook verification fails open when the secret is empty:
  - ShipStation (`shipstation_webhook.py:24`)
  - Shopify (`shopify.py:2411`)
  - 17Track (`order_tracking_service.py:327`)
- H5: Production data on the dev laptop:
  - DB dump in the bench root
  - `api_key (1).docx`
  - backups under `sites/*/private/backups`
  - customer POs and XLSX files

**Medium**
- M1: Quotation approval HMAC falls back to the literal `"lh_approval_secret"`, and tokens never expire (`quotation.py:2110`, `shopify.py:2521`).
- M2: Credentials stored as plain Data fields:
  - S3 `aws_key`/`aws_secret`
  - `ShipStation Settings.ss_api_key`
  - Slack webhook URLs in five settings doctypes
- M3: 38 whitelisted methods use `ignore_permissions=True` with no role check.
- M4: The `ecommerce_integrations` Shopify guest endpoint compares the secret with `!=` (not constant-time). CI actions are old and pinned to tags, and `lh` has no CI.
- M5: Custom apps are hosted on personal GitHub accounts (viralpatel15, viralkansodiya, Viral-LyFe). That is a supply-chain and bus-factor risk.

**Low**
- Unpinned deps (python-barcode, taxjar, requests, pycountry, pandas, pysftp, cypress ^13).
- `os.system("pip install …")` in three scripts.
- Reflected `error` parameter in `slack/api.py:142`.
- `developer_mode`/`allow_tests` on locally only.

**Confirmed OK**
- No SQL injection from user input found.
- Uploads are private.
- Gorgias and Slack webhooks fail closed.
- The MCP connector was hardened in Aug.
- Local services bind to 127.0.0.1.

## Content of 02 – Review plan, phases and dates (proposed commitments)
| Phase | Dates | Scope |
|---|---|---|
| **P0 Containment** | Oct 23–24 | See P0 list below. |
| **P1 Assessment** (six areas) | Oct 26 – Oct 30 | See P1 list below. |
| **Report to TJ** | **Fri 2026-10-30** | Checked / exposure / risks / actions with dates. |
| **P2 Fix Highs** | by Nov 6 | All remaining High findings plus anything P1 rates High. |
| **P3 Fix Mediums, CI, hardening** | by Nov 20 | Mediums, CI pipeline for `lh`, dependency pinning. |
| **First restore drill + access review** | by Nov 27 | Then quarterly. |

**P0 Containment (Oct 23–24)**
- Rotate the Shopify secret and token, the Slack webhooks and the 1Click keys.
- Remove the hardcoded values from HEAD and read them from Settings instead.
- Confirm the lh repo is private and who can access it.
- Confirm the production frappe/erpnext version and update to ≥15.121.3.
- Delete or encrypt the prod dumps and docx on the dev box.
- Make the three webhooks fail closed.
- Require login on the guest tracking endpoint.

**P1 Assessment (six areas), Oct 26 – Oct 30**
1. **Infra & hosting:**
   - Frappe Cloud plan, region and server type (shared vs dedicated).
   - Firewall/IP rules in the Security tab.
   - SSL/TLS grade (SSL Labs) and security headers.
   - Exposed ports (external nmap of the prod domain).
   - Custom domains and DNS (SPF/DKIM/DMARC).
   - Frappe Cloud account MFA and team members.
2. **Supply chain:**
   - `pip-audit` on the bench env and `npm audit`/`yarn audit` on the frappe/lh/print_designer/insights frontends.
   - Pin floating deps.
   - Check each third-party app (harro, vesta_si_erpnext, frappe_s3_attachment, india_compliance, insights, raven, lms, crm): maintainer, last update, whether it is needed in prod at all, and remove unused apps.
   - Move personal-account repos into the `lyfehardware` GitHub org.
   - Dev tooling: Claude Code, VS Code extensions, node/nvm, MCP servers on the dev box.
3. **APIs & integrations:**
   - Shopify (app scopes, webhook HMAC, token age), ShipStation, ShipEngine, CJ, 1Click, 17Track, TaxJar, ZipTax, Gorgias, Slack, Anthropic, Insightful, Clarity, S3 (bucket policy, public access block, IAM key scope and age).
   - The MCP connector and its OAuth clients.
   - Claude.ai connectors with org access: Gmail, Drive, Slack, ERPNext, Asana, n8n.
   - Least-privilege scopes and an owner for each integration.
4. **Credentials & access:**
   - Enforce 2FA for System Users in System Settings.
   - Audit Administrator and System Manager role holders.
   - Disable ex-staff accounts.
   - Inventory user API keys/secrets and OAuth bearer tokens, and revoke stale ones.
   - Password policy and session expiry.
   - Move Data-field secrets to Password fields.
   - GitHub org 2FA and collaborators, and AWS IAM users and MFA.
   - Search Slack, Drive and email for shared credentials (like `api_key (1).docx`) and rotate whatever turns up.
5. **Deployment & code:**
   - Branch protection on `prod`, required review, no direct pushes.
   - Add a CI workflow to `lh` that runs ruff, bandit/semgrep, gitleaks and the tests.
   - Purge secrets from git history (`git filter-repo`), but only after rotation.
   - Review the remaining `ignore_permissions` and guest endpoints.
   - Upgrade cadence for Frappe Cloud deploys, with staging-first testing.
6. **Monitoring & recovery:**
   - Frappe Cloud backup frequency, retention and offsite copy.
   - **An actual test restore to staging**, with RTO/RPO written down.
   - Review the Access Log, Activity Log, failed logins and the MCP audit log.
   - Alerting to Slack for repeated failed logins, new System Managers and new API keys.
   - Incident runbook.
7. **Extra:**
   - Email/phishing posture (DMARC).
   - Laptop security for the people with prod access (disk encryption, OS updates).
   - Data minimisation: no prod dumps on laptops.

## Content of 03 – Ongoing process
- **Weekly (Mon, ~30 min):**
  - Check advisories: Frappe/ERPNext GitHub security advisories, GitHub Dependabot alerts on all org repos, `pip-audit` and `npm audit`.
  - Note "known incidents this week vs us" in the worklog.
- **Monthly (first working day):**
  - Run gitleaks over all repos and review new guest/whitelisted endpoints in that month's diffs.
  - Review user, role and API key changes.
  - Review the Frappe Cloud version and apply patch updates via staging.
  - Send a one-line status to TJ.
- **Quarterly:**
  - Backup restore drill.
  - Full access review (ERP, GitHub, AWS, Frappe Cloud, Shopify staff, Claude connectors).
  - Rotate high-value keys.
  - Re-run this six-area checklist and send a short report.
- **Per change:**
  - The PR checklist covers no secrets in code, guest endpoints authenticated, webhooks failing closed and permission checks on whitelisted methods.
  - CI gitleaks blocks merges that leak secrets.
- **Incident runbook:** detect → contain (rotate, revoke, firewall) → notify TJ within 1 hour for Critical → eradicate → restore → post-mortem.
- **Ownership:** Viral owns the process, with Arth and Srishti as backups. Tracked as a recurring Asana task.

## Execution steps after approval
1. Create the folder and the four .md files above. Doc-only: no code changes, no secret values.
2. Show the Slack reply text in chat so Viral can paste it. Nothing is posted to Slack without explicit OK.
3. Code fixes (P0) are a separate follow-up task, and each one is confirmed with Viral first.

## Verification
- Every finding in 01 has a file:line that still exists (re-grep before writing).
- No secret values appear in any file: grep the new folder for `shpss_|shpat_|hooks.slack.com|AKIA|xox`.
- The dates in 00 and 02 match exactly.
