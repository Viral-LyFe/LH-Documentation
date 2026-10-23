# Dependency & supply-chain check

**Date:** 2026-10-23
**Where:** **local dev bench** (`/home/frappe/frappe-bench`, WSL). Prod can run different versions, so run `bench version` and `env/bin/pip freeze` on prod and compare before acting.
**Scope:** the 11 apps in use: frappe, erpnext, hrms, crm, insights, lms, payments, ecommerce_integrations, frappe_s3_attachment, shopify_tax, lh.
**Sources:**
- App versions against each app's published GitHub security advisories.
- Python packages reachable from those apps (190), as installed in the bench env, checked against OSV (osv.dev).
- npm packages pinned in those apps' `yarn.lock` files (3,056 name/version pairs), checked against OSV.

Only package names and versions were sent to osv.dev and GitHub. No project code was run. Raw results are in `raw/deps_osv.json` and `raw/deps_app_advisories.json`.

---

## Summary

| Area | Result | Priority |
|---|---|---|
| Frappe / ERPNext / HRMS | 3, 3 and 4 v15 point releases behind. Those releases contain many security backports. | **High**: update now |
| LMS | On the `develop` branch at 2.45.2. About 17 published advisories apply, including **2 critical SCORM path traversals**. | **High** |
| Insights | 2.2.14. Two v3 advisories (critical sandbox escape, password-hash disclosure via Native Query) may also affect v2. | **High**: verify |
| CRM | 1.82.0. A personal-data disclosure advisory lists `<= 1.82.0` as affected. | Medium |
| Python packages | 12 vulnerable. The ones that matter on the server: **Pillow, certifi, pypdf, WeasyPrint, oauthlib, aiohttp, lxml**. | **High** for Pillow and certifi |
| Node runtime (realtime server) | socket.io / engine.io / ws versions with DoS issues | Medium |
| Browser-bundled JS | Sanitizer and editor XSS issues (DOMPurify, ProseMirror, Quill, showdown, ECharts) | Medium (fixed by app updates) |
| Build-only JS | ~170 issues in vite, rollup, postcss, babel, glob, etc. | Low: developer machines and build only |
| Supply chain | 2 in-use apps come from **public repos on a personal GitHub account**. One app is unmaintained since 2023. Local edits in insights and lms. | Medium |

---

## 1. App versions

| App | Installed (local) | Latest same line | Latest | Notes |
|---|---|---|---|---|
| frappe | v15.121.1 | **v15.124.0** | v16.53.0 | Upstream v15 commits since then include: file-access and attachment permission fixes, backup-API restricted to System Manager, email-preview restricted, Workspace edit permission, RestrictedPython 8.5, `restrict_globals` for notifications. Advisory "IDOR in update_onboarding_step" explicitly lists **15.0.0–15.122.x affected, fixed 15.123.0**. |
| erpnext | v15.121.4 | **v15.124.1** | v16.52.1 | 900 commits since then, with many permission fixes. Includes `fix(permissions)!: authorise whitelisted methods against the records they return`, a **breaking change**: test lh flows that call ERPNext whitelisted methods. |
| hrms | v15.64.2 | **v15.64.6** | v16.52.0 | Includes "Better perm checks for HRMS" and attendance and roster validations. |
| lms | 2.45.2 on **`develop`** | — | v2.67.1 | Unfixed advisories with a fix after 2.45.2 include: **Path traversal in SCORM file serving (critical, fixed 2.52.0)**, **path traversal in SCORM (critical, fixed 2.50.1)**, client-controlled course price (2.52.1), quiz score manipulation (2.46.0, 2.54.0), restricted-content access (2.56.0), grading data modification (2.59.0), user info disclosure (2.61.0) and several stored XSS issues. |
| insights | v2.2.14 | v2.2.15 | v3.15.1 | Advisories "Privilege escalation via Script Query sandbox escape" (**critical**) and "Credential and password hash disclosure via Native Query" (high) list `< 3.9.4`. They don't say whether v2 is affected. Check with Frappe, or restrict who can create Insights queries. |
| crm | v1.82.0 (+3 local commits) | — | v1.89.1 | "Unauthorised disclosure of personal information" lists `<= 1.82.0` as affected. |
| payments | `version-15` branch | — | — | No advisories |
| ecommerce_integrations | 1.17.0, **2-commit snapshot** in `viralpatel15/Ecommerce-Integrations` | — | — | A copy rather than a fork, so upstream security fixes don't flow in. |
| shopify_tax | `viralpatel15/Shopify-Tax-ERPnext` | — | — | Custom app in a **public** personal repo. History scanned: no secrets. |
| frappe_s3_attachment | `shridarpatil/Frappe-attachments-s3`, last commit **2023-11** | — | — | Unmaintained upstream. lh monkeypatches it (`lh/__init__.py`). |

**About the advisory numbers.** Frappe and ERPNext publish most advisories with only the **v16** fix version (e.g. "< 16.34.0"). That includes recent critical ones: Frappe "Remote code execution via path traversal" (2026-10-15), "Possibility of Server Side Template Injection" (2026-10-15), "Arbitrary file read via an attribute in outgoing email HTML" (2026-09-16), and ERPNext SQL injection and SSTI advisories (Aug–Sep 2026). Many are backported to v15, but the advisories don't say which. So the real exposure of v15.121 can't be read off the list. Updating to the latest v15 is the minimum, and v16 is the only line with every listed fix.

---

## 2. Python packages (server-side, run in production)

| Package | Installed | Fix | Worst | Why it matters here |
|---|---|---|---|---|
| **Pillow** | 12.3.0 | 13.0.0 | Critical | 39 advisories, including heap buffer overflows in TIFF/JPEG decoding. Frappe processes **uploaded images** (thumbnails, optimisation), so a crafted upload reaches it. |
| **certifi** | 2026.7.22 | 2026.10.19 | Critical | Bundles distrusted emSign root CAs. Affects every outbound HTTPS call (Shopify, S3, 1Click, CJ, Slack). |
| **pypdf** | 6.15.0 | 6.21.1 | High | 30 advisories (CPU/memory DoS on crafted PDFs). Used when PDFs are read or merged. |
| **WeasyPrint** | 69.0 | 70.0 | High | SSRF and EPS → Ghostscript. Used for PDF rendering. |
| **pdfkit** | 1.0.0 | none | High | Path traversal in `from_string`. Frappe's own advisory "Authenticated SSRF / LFI via pdfkit" is fixed in Frappe itself, so update Frappe. |
| **oauthlib** | 3.3.1 | 4.0.0 | Medium | PKCE timing attack and JSONP callback injection. Frappe's **OAuth provider is what the MCP connector uses**. 4.0.0 is a major version, so follow Frappe's pin. |
| **aiohttp** | 3.14.3 | 3.14.4 | High | 8 advisories (DoS, missing header timeout). Pulled in by crm. |
| **lxml** | 6.0.4 | 6.1.0 | High | Default `iterparse()` / ETCompat config (XXE-style). Pulled in by lms. |
| GitPython | 3.1.62 | 3.2.1 | High | Submodule path validation. Used by bench tooling, not exposed to users. |
| Werkzeug | 3.1.3 | 3.1.9 | Medium | `safe_join` issues are **Windows-only**, so not exploitable on Linux. Update anyway. |
| Markdown | 3.5.2 | 3.8.1 | Medium | Uncaught exception (DoS). |
| opentelemetry-api | 1.44.0 | 1.45.1 | Medium | Unbounded tracestate parsing. |

Most of these are pinned by Frappe/ERPNext, so updating the apps (section 1) moves many of them. After updating, check what's left with `env/bin/pip list --outdated`, and raise the rest (Pillow, certifi) explicitly if Frappe's pins allow.

---

## 3. JavaScript packages

**a) Server-side at runtime: the Frappe realtime (socket.io) Node process**

| Package | Installed | Fix | Issue |
|---|---|---|---|
| engine.io | 6.5.4 | 6.6.7 | WebTransport SID DoS, polling connection exhaustion |
| socket.io-parser | 4.2.4 | 4.2.7 | Unbounded binary attachments (memory exhaustion) |
| ws | 8.11.0 | 8.21.0 | DoS on crafted headers and tiny fragments, uninitialized memory disclosure |
| cookie | 0.4.2 | 0.7.0 | Out-of-bounds characters in name/path/domain (Low) |

These run on prod and are reachable by any client that can open the websocket. Fixed by updating Frappe and rebuilding.

**b) Shipped to the browser (XSS-class)**

DOMPurify (3.2.6 / 3.4.0 / 3.4.9 in crm and lms; bypasses of `SAFE_FOR_TEMPLATES`, `ADD_TAGS`, `IN_PLACE`), prosemirror-view (XSS, in hrms/insights/crm/lms), Quill 1.3.7 / 2.0.3 (XSS, no fix, used by the Frappe text editor), showdown 2.1.0 (stored XSS, no fix), html-janitor (sanitizer bypass, no fix), ECharts 5.x (XSS, fix 6.1.0), lodash `_.template` code injection, Tiptap prototype pollution, `@vue/server-renderer` attribute XSS.

These are pulled in by the upstream apps. Updating the apps is the fix. The ones with no fix (Quill, showdown, html-janitor) depend on Frappe sanitising content on the server, which it does on `save()`. This is why the lh items about writes that skip sanitising (`db.set_value`, raw SQL) matter.

**c) Build and dev only (~170 entries)**

vite, rollup, postcss, babel, esbuild, braces/micromatch/picomatch/minimatch, glob, cross-spawn, launch-editor, playwright, vitest, undici, protobufjs/grpc/firebase (hrms PWA build), etc. These run during `bench build` or on developer machines, not in user requests. Low priority for prod. They do matter for anyone running a vite dev server on a shared network (e.g. `server.fs.deny` bypass, esbuild dev server CORS).

---

## 4. Supply chain

1. **Public personal repos.** `ecommerce_integrations` and `shopify_tax` are installed from `github.com/viralpatel15/…`, which are **public** and owned by a personal account. Risks: whoever controls that account controls code that runs on prod; company integration code is public; and access isn't removed when people leave. **Fix:** transfer both repos to the `lyfehardware` org, make them private, and update the bench remotes.
2. **ecommerce_integrations is a snapshot, not a fork** (2 commits). Upstream `frappe/ecommerce_integrations` security fixes never arrive. **Fix:** re-base on upstream (keep the lh changes as commits on top) or track upstream as a second remote.
3. **frappe_s3_attachment is unmaintained** (last upstream commit 2023-11) and lh monkeypatches its internals. **Fix:** vendor it into the lyfehardware org and own it, or move to a maintained S3 attachment app. Frappe's built-in S3 support covers backups only, not file attachments.
4. **LMS runs from `develop`,** the unstable branch. **Fix:** pin it to a release tag.
5. **Local modifications to upstream apps:** `insights` and `lms` have uncommitted changes on this bench. If prod also carries edits to upstream apps, a `bench update` will overwrite or conflict with them. Check prod with `git status` in each app.
6. **No automated dependency alerts.** **Fix:** enable Dependabot (or OSV-Scanner in CI) on `lh-erpnext`, `Shopify-Tax-ERPnext` and the ecommerce fork. Add `pip-audit` / `osv-scanner` to the schedule's CI step (SCHEDULE.md 5.9).

---

## Recommended actions

| # | Action | Effort |
|---|---|---|
| D1 | On **prod**, run `bench version` and `pip freeze` and compare with this report. | 10 min |
| D2 | Update frappe → v15.124.0, erpnext → v15.124.1, hrms → v15.64.6 on staging. Run the lh smoke tests, especially calls into ERPNext whitelisted methods (breaking permission change). Then prod. | ½–1 day |
| D3 | LMS: move from `develop` to the latest release compatible with v15, which fixes the SCORM path traversals. If LMS SCORM isn't used, also disable SCORM uploads. | ½ day |
| D4 | Insights: update to v2.2.15 and ask Frappe whether the v3.9.4 advisories affect v2. Meanwhile restrict Native/Script Query creation to admins. | 1 h + vendor |
| D5 | CRM → v1.89.1. | 1 h |
| D6 | Make sure Pillow ≥ 13.0.0 and certifi ≥ 2026.10.19 after D2; raise them if Frappe's pins allow. | 1 h |
| D7 | Move `ecommerce_integrations` and `shopify_tax` to the lyfehardware org (private). Re-base ecommerce on upstream. | ½ day |
| D8 | Pin `frappe_s3_attachment` in the org or plan to replace it. | plan |
| D9 | Enable Dependabot / OSV-Scanner in CI. | 1 h |
| D10 | Plan the v16 upgrade. It's the only line where every published Frappe/ERPNext fix is confirmed. | project |

D1–D6 fit into the security week (SCHEDULE.md): D1 on Monday, D2–D6 on staging mid-week, prod on Friday.
