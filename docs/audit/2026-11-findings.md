# Findings — November 2026

Working notes for the Hallan portal front end. The report is `2026-11-report.md`. Terms are defined in that report's glossary.

Each finding has a severity, a location, the evidence in this repository, and a suggested fix. Server behaviour is marked unverified when the API source is not here.

## Writes enumerated

Absolute claims such as "no mass assignment" are not made. Writes found by searching `fetch` and `apiPost` / `apiPut` / `apiPatch` / `apiDelete`:

- Staff portal: login, verify, resend, forgot-password, leave-request create, leave-request status patch, holiday upload. Bodies are named fields, except the upload, which posts every parsed spreadsheet row (`mode`, `rows`, `createMissing`, `replaceHistory`).
- Rota login: the same staff auth posts, plus signup. Schedule: `PUT /api/staff/rota-state` with the entire in-memory `state` object, and `PUT /api/staff/timesheet-entry` with punch fields. Change-password is a named body.
- Clock: `PUT /api/staff/timesheet-entry` with `employeeId`, `date`, and punch times chosen in the browser.
- Management: login, zone create, location create, check, user create, and the matching deletes. Bodies are named fields. The new-user password is read from an input of `type="text"`.
- Picking app: a long set of portal, stock, wholesale, restaurant, WhatsApp, goods-in, location, and damage writes. Two are called out below because they are not a tight named body: the rota-style wholesale price lock posts the on-screen price list, and `POST retail/lists/:id/track-time` sends `{ phase, seconds }` with no bearer token. Portal user save sends `name`, `email`, `role`, `modules`, and `passcode` from the admin form.
- PDF snippet: `POST /api/wholesale/flyer/pdfs` and the delete route. The auth header reads `localStorage` key `portal_token`, which the picking app never sets.

## Security

### SEC-1 — Clock downloads PINs and checks them in the browser

- **Severity:** High
- **Location:** `rota/clockin.html` around the employee fetch and the `loginId` / `pin` comparison. `rota/schedule.html` keeps `pin` on each employee and puts the whole `state` back with `PUT /api/staff/rota-state`.
- **Evidence:** The clock loads `GET /api/staff/rota-state` with whatever `rota_token` is in the browser, then accepts a person when `e.loginId` and `e.pin` match the typed values. The schedule employee form shows the PIN in a text field and saves it inside the rota document. Invariant 9 in `docs/00-security.md` says this payload must not contain PINs, and invariant 10 says the clock must not use a personal token. A working clock implies the API currently returns PINs. That response was not captured live.
- **Suggested fix:** Add a server check that accepts Login ID and PIN and returns only that person's name and today's punches. Stop returning `pin` on `rota-state`. Give the till a device credential that can write only the punch just verified.

### SEC-2 — Shared till keeps a personal rota token

- **Severity:** High
- **Location:** `rota/clockin.html`, the comment block above `API_BASE` and the read of `rota_token`.
- **Evidence:** The page tells a manager to log in via `rota/index.html` on the device. That stores `rota_token`, `rota_role`, and `rota_login_id`. The clock then reuses the token. The comment says it expires after about twelve hours. Anyone who can open the browser's storage on that device holds a manager or HR session, not just a clock button.
- **Suggested fix:** Same device credential as SEC-1. Do not leave a person's session on the shared screen.

### SEC-3 — Forgotten password is described as saved before the email code

- **Severity:** High
- **Location:** `staff-portal/index.html` in `submitForgotPassword`. `rota/index.html` follows the same two-step comment.
- **Evidence:** After `POST /api/staff/forgot-password` returns `ok`, the staff page says the password has already been updated and only then asks for the code. The department list is loaded from `GET /api/staff/departments` with no token, so the department is not a secret. Invariant 8 forbids storing the new password before the code succeeds. The API source was not reviewed, so this is what the page claims the server does.
- **Suggested fix:** On the API, hold the new password until `POST /api/staff/login/verify` succeeds. Until then, leave the old hash in place. Keep the public department list as a lookup, not as proof of identity.

### SEC-4 — Rota scope is decided from browser storage

- **Severity:** High
- **Location:** `rota/index.html` (`rota_role`, `rota_login_id`) and `rota/schedule.html` (`applyRoleNav`, `findAccessGrant`, `syncStateToServer`).
- **Evidence:** The schedule reads the role and the login id from `localStorage`, hides tabs from those strings, and filters departments by matching the login id to `accessGrants` after the full rota document has been downloaded. The save sends `{ data: state }` for the whole document with the bearer token and no department list. Invariants 1 and 4 say the API must scope this. Whether it does is unverified. The client will show and write whatever the API returns, and it will send the whole blob back.
- **Suggested fix:** Filter `GET /api/staff/rota-state` to the caller's departments. Reject a `PUT` that changes employees, grants, or shifts outside that scope. Do not trust `rota_role`.

### SEC-5 — Module tiles are hidden only in the page

- **Severity:** Medium
- **Location:** `picking-sheets/index.html`, `portalCan` and the `data-module` tiles.
- **Evidence:** A non-admin sees a tile only when `portalSession.modules` contains the key. The session object is in `localStorage`. The API must repeat the check on every route (invariant 3). That server check was not visible here. Hiding the tile is not the control.
- **Suggested fix:** On each wholesale, restaurant, WhatsApp, and stock route, reject a token whose module list does not include that area. Treat a hand-edited `modules` array as meaningless.

### SEC-6 — Wholesale and customer passwords are a single prompt

- **Severity:** Medium
- **Location:** `picking-sheets/index.html`, `openWholesalePricing` and the Manage-Customers prompt.
- **Evidence:** The page posts the shared password once to `wholesale/auth/verify` or `restaurant/admin/auth/verify`, then later reads prices and customers with the portal bearer token only. The wholesale change form accepts a new password of four characters. Invariant 12 asks for a server check on every sensitive read. Invariant 5 says the client length is not the control.
- **Suggested fix:** Make the API require a fresh wholesale or customer-admin grant on each price and customer request, and enforce a real minimum length on the server. Prefer named users over a shared password when those screens are next changed.

### SEC-7 — Picking timer posts with no token

- **Severity:** Medium
- **Location:** `picking-sheets/index.html`, `flushActiveTimer`.
- **Evidence:** `POST /api/retail/lists/:id/track-time` is sent with `Content-Type` only, and `sendBeacon` cannot add an `Authorization` header. The body is `{ phase, seconds }` and the list id is in the URL. If the API does not authenticate this route, anyone who can guess or observe a list id can add time.
- **Suggested fix:** Authenticate the route. If a beacon is required, use a short-lived single-purpose token in the URL that the API minted for that list, and cap the seconds.

### SEC-8 — PDF library inserts filenames and URLs as HTML

- **Severity:** Medium
- **Location:** `pdf-library-admin-snippet.html`, the `listBody.innerHTML` template.
- **Evidence:** `f.filename` and `f.url` are concatenated into HTML. The picking app's own PDF table should be checked the same way, but this snippet has no escape. A filename containing markup runs in the admin's browser, where tokens live in `localStorage`. Invariant 16 requires escaping. The snippet also reads `portal_token`, so it does not send the real session (see BUG-3).
- **Suggested fix:** Build the rows with `textContent`, or escape filename and URL. Read the token from `hallan_portal_session`.

### SEC-9 — Rota screen inserts names and reasons as HTML

- **Severity:** Medium
- **Location:** `rota/schedule.html`, `renderSchedule` and the employee `<select>` builders.
- **Evidence:** `off.reason`, shift type names, and `e.name` are placed into `innerHTML` without escaping. Those strings are saved inside the rota document and come back to every browser that can load it. Staff portal and management do use an `escapeHtml` helper for the names they render.
- **Suggested fix:** Escape every rota field before it becomes HTML, the same way the staff portal escapes a holiday reason.

### SEC-10 — Staff token is kept until sign-out

- **Severity:** Medium
- **Location:** `staff-portal/index.html`, the comment above `TOKEN_KEY`.
- **Evidence:** The page stores `hallanStaffToken` in `localStorage` and the comment says the only client-side clearance is sign-out or a server rejection, "never just time passing". Invariant 14 says the server must expire tokens anyway. The server TTL was not readable here. The same token family is not cleared when the person only signs out of the rota, because that uses `rota_token`.
- **Suggested fix:** Expire and rotate tokens on the API. Password change and deactivation must revoke `hallanStaffToken` and `rota_token` together (invariant 13).

### SEC-11 — New management password is visible as it is typed

- **Severity:** Low
- **Location:** `management/index.html`, the Manage Users inputs.
- **Evidence:** `newUserPassword` is `type="text"`. The sign-in field is `type="password"`. Any management token can `POST /api/management/users/`.
- **Suggested fix:** Use `type="password"`. On the API, limit user creation to a named owner if that is the intention.

### SEC-12 — Public pages send no security headers

- **Severity:** Low
- **Location:** Response headers from `GET https://portal.hallans.co.uk/` on 13 November 2026.
- **Evidence:** The response was `200` from GitHub Pages, with `access-control-allow-origin: *` and `cache-control: max-age=600`. It did not include `Content-Security-Policy`, `Strict-Transport-Security`, `X-Frame-Options`, `X-Content-Type-Options`, or `Referrer-Policy`. Tokens are in `localStorage`, so a missing content-security policy matters only together with an XSS bug such as SEC-8 or SEC-9. GitHub Pages does not give this repo a place to set those headers.
- **Suggested fix:** If the pages move behind a host that can set headers, add a content-security policy without `unsafe-inline`, nosniff, a referrer policy, and frame-ancestors. HSTS belongs on the HTTPS edge.

### Checks passed — security

- The staff login request body is `employeeId` and `password` only. The role `<select>` is not sent. Checked in `submitLogin`.
- Staff, rota, and portal login store a token only in the verify handler, not in the first response handler. The pages match invariant 7. The API was not inspected.
- No `.env`, private key, or cloud secret was found in the working tree. `WA_SIGNUP_APP_ID` and `WA_SIGNUP_CONFIG_ID` are public client ids, as the page comment says.
- Auth for the main apps is a bearer header, not a cookie, so a cross-site form post does not attach the session. The exception is SEC-7. Cookie flags were not reviewed because no cookie is set in these pages.
- There is no SQL or shell call in this repository. Injection risk here is HTML injection (SEC-8, SEC-9), not SQL.
- Password hashing, rate limits, CORS, and session revocation were not verifiable. They are not recorded as passes.

The public department list is not a finding. `docs/00-security.md` already says it is a lookup key, not a secret.

## Privacy

### PRIV-1 — No privacy notice, export, or deletion path in the product

- **Severity:** Medium
- **Location:** All public pages. There is no privacy link in `home.html`, the staff portal, or the rota.
- **Evidence:** The API is described, in the security model, as holding names, staff ids, emails, holiday balances, shifts, clock times, and restaurant customer names. Nothing in this repository explains that to the person, offers a copy, or deletes the row. Retention is not stated. This is a process gap, not a legal opinion.
- **Suggested fix:** Publish a short notice HR agrees, and write a runbook for a copy or deletion against the API database. Deletion must revoke the token families in invariant 13.

### PRIV-2 — A shared browser can hold everyone's rota, including PINs

- **Severity:** High
- **Location:** `rota/clockin.html` and `rota/schedule.html`.
- **Evidence:** The clock's token download is the full employee list (SEC-1). The schedule keeps `email`, `loginId`, and `pin` in the document and shows the PIN in a text input. That is personal data, and on a till it is visible to whoever can open developer tools. Need-to-know for a PIN is the person who sets it and the API that checks it.
- **Suggested fix:** The same server-side PIN check as SEC-1. Do not render stored PINs back into the employee form. Offer a reset that replaces the PIN without displaying the old one.

### PRIV-3 — Holiday spreadsheets and third-party scripts see personal data

- **Severity:** Medium
- **Location:** `staff-portal/index.html` (SheetJS from cdnjs, Excel upload). `picking-sheets/index.html` (Facebook SDK, loaded on demand).
- **Evidence:** Employee upload rows include name, department, email, and holiday figures, parsed in the browser by a script loaded from `cdnjs.cloudflare.com`, then posted as JSON. The page comment on the Facebook SDK says a third-party script can read the page, including the saved token. Google Fonts are loaded from `fonts.googleapis.com` on the staff and picking pages. No consent banner is present. Staff data is employment data, so a shop-floor cookie banner is the wrong tool, but the processors (Google, Cloudflare's CDN, Meta when WhatsApp signup runs) are not listed anywhere in the repo.
- **Suggested fix:** Host the spreadsheet library with the site, or accept the CDN in the privacy notice. Keep the Facebook SDK on the click that needs it (the page already does this). Name the processors in the notice from PRIV-1.

### PRIV-4 — Sessions and customer names stay in localStorage

- **Severity:** Low
- **Location:** Token keys in `docs/00-security.md` invariant 13, plus `hallan_restaurant_customer` and `hallan_staff_name`.
- **Evidence:** Sign-out removes the token for that app only. Restaurant sign-out also removes the customer object. A stolen XSS payload (SEC-8, SEC-9) can read every key for that origin. No analytics product was found writing emails to a third party.
- **Suggested fix:** Keep tokens out of pages that do not need them. Do not add a second copy of the customer record beyond the id the API requires.

### Checks passed — privacy

- The static repository does not contain a database dump, a staff spreadsheet, or a list of real employees. Searched the tree for committed secrets and did not find a personal-data file. The large flyer HTML files are product pages, not staff lists. They were not read line by line.
- Login forms do not write the password into `localStorage`. They post it and then store the token.
- No Sentry or other error reporter was found, so there is no error pipeline leaking emails. There is also no server log to review in this repo.
