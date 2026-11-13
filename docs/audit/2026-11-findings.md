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

## Accessibility

High findings in this section are WCAG 2.2 AA failures. They are not commercial-secrecy or data-isolation blockers. Fix them after SEC-1 to SEC-4 unless Hallan has promised AA at a particular date.

### A11Y-1 — Sign-in fields have no labels

- **Severity:** High
- **Location:** `picking-sheets/index.html` inputs `loginName` and `loginPass`. `management/index.html` inputs `loginName` and `loginPassword`.
- **Evidence:** Both screens use a `placeholder` and no `<label>`, no `aria-label`, and no wrapping label. Placeholder text disappears once the field has a value. The staff portal and the rota login do associate `<label for>` with each input.
- **Suggested fix:** Add a visible label for each field, matching the staff portal. Keep the placeholder only as an example.

### A11Y-2 — Dialogs do not trap focus

- **Severity:** Medium
- **Location:** `staff-portal/index.html`, `#hrPartialModal` and `#hrHistoryModal`.
- **Evidence:** Both are `<div>` overlays toggled with a class. The history modal closes on a backdrop click. Neither uses the `dialog` element, and the script has no focus trap and no Escape handler. Keyboard users can tab into the page behind the overlay.
- **Suggested fix:** Use `<dialog>`, move focus into it on open, restore focus on close, and close on Escape.

### A11Y-3 — Several pages have no main landmark or skip link

- **Severity:** Low
- **Location:** `home.html` (header, sections, footer, no `<main>`). `staff-portal/index.html` and `management/index.html` (no `<main>`). No page has a skip link.
- **Evidence:** The rota login and the mock timesheet do use `<main>`. The home page's logo image has `alt="Hallan"`. Viewports set `width=device-width` and do not set `user-scalable=no`.
- **Suggested fix:** Wrap the primary content in `<main>` and add a skip link on pages whose header has more than a couple of controls.

### Checks passed — accessibility

- The rota login fields have real labels, and focus is shown with a blue border plus a shadow after `outline: none`.
- The home page has one `h1` in the header. The rota login has one `h1` per form, and inactive forms are `display: none`.
- Colour contrast, touch-target size, and screen-reader behaviour were not measured. They are not recorded as passes.
- The generated flyer HTML was not walked control by control. `wholesale-flyer.html` and `veg-flyer.html` are published artwork.

## Data structures

Database types, foreign keys, and migrations are not in this repository. That absence is a limit of the review, not a finding that the API has no migrations.

### DATA-1 — Holiday days are whatever the browser sends

- **Severity:** Medium
- **Location:** `staff-portal/index.html`, `submitHolidayRequest`.
- **Evidence:** The form has start date, end date, and a free-text "Days requested" field. The post sends `daysRequested` as typed, or `null`. Nothing in the page checks that the number matches the date range, or that it is a number. A partial approval later sends `partialDays` the same way.
- **Suggested fix:** On the API, derive the day count from the dates and the employee's working pattern, and reject a client number that does not match. Keep the text field as a hint if HR still wants one.

### DATA-2 — A holiday upload can create employees by default

- **Severity:** Medium
- **Location:** `staff-portal/index.html`, the upload checkbox `optCreateMissing`.
- **Evidence:** "Create employees that don't exist yet" is checked by default. The post sends `createMissing: true` unless someone unticks it. A spreadsheet with a new staff ID therefore asks the API to insert a person, including email and holiday figures.
- **Suggested fix:** Default the checkbox off. On the API, allow creates only for an HR token, and ignore the flag for everyone else.

### DATA-3 — Selling prices are snapshotted from the screen

- **Severity:** Medium
- **Location:** `picking-sheets/index.html`, `toggleWholesalePriceBlock`.
- **Evidence:** Block sends `{ stock_name, price, margin, changed }` taken from `wholesaleAdjust`, which the page has been editing. Invariant 11 says the API's stored decimal is the price of record. The client treats the on-screen number as the snapshot to freeze. Cost is deliberately left out of that snapshot.
- **Suggested fix:** Have the API recompute the locked price from its own row, or reject a client price that does not match the last value it issued. Do not accept an arbitrary number because the button was labelled Block.

### Checks passed — data

- The rota save sends `expectedUpdatedAt` and reloads on `409`, so two browsers do not silently overwrite each other. Client ids from `uid()` are still not database keys.
- `rota/schedule.html` builds dates from local year, month, and day after an earlier UTC bug. The clock does not. That bug is BUG-1.
- No second writable copy of the wholesale price was found in this repo besides the lock payload and the published flyer.

## Bugs

### BUG-1 — The clock still uses a UTC date

- **Severity:** Medium
- **Location:** `rota/clockin.html`, `toDateStr`.
- **Evidence:** It returns `d.toISOString().slice(0, 10)`. `rota/schedule.html` documents that this shifts the calendar day back by one during British Summer Time, and that file has already stopped doing it. A punch just after midnight UK time, or late in the evening while BST is in force, can be stored against the previous UTC date. The timesheet tab on the schedule will then disagree with the clock.
- **Suggested fix:** Use the same local year-month-day function as the schedule.

### BUG-2 — The front door describes the wrong page

- **Severity:** Low
- **Location:** `index.html`.
- **Evidence:** The meta refresh and script send people to `home.html`, which is the portal. The visible sentence says "Redirecting to the Wholesale Vegetable Flyer". A second link points at `/home/`, and there is no `home/` directory.
- **Suggested fix:** Point the sentence at the portal, and remove or retarget the `/home/` link.

### BUG-3 — The timesheet page is a mock that looks live

- **Severity:** Medium
- **Location:** `timesheet/index.html`.
- **Evidence:** The script comment says "Front-end mock" and names endpoints that are not called. Clock in only toggles a class. The week table is filled with fixed sample hours (`8:58 AM`, `5:02 PM`, and a total of `28h 08m`). The nav highlights Time Clock. Someone can believe a punch was recorded.
- **Suggested fix:** Remove the page from any linked nav, or replace the sample rows with the real `timesheet-state` API and label it until that exists. The home page does not link this file. The rota's own timesheet tab is the live one.

### BUG-4 — The PDF snippet reads a token key nothing sets

- **Severity:** Medium
- **Location:** `pdf-library-admin-snippet.html`, `authHeader`.
- **Evidence:** It reads `localStorage.getItem('portal_token')`. The picking app stores the session under `hallan_portal_session`. Dropping the snippet onto a page that is already logged in still sends no bearer token, so upload and list fail or, if the API is loose, run unauthenticated. See also SEC-8.
- **Suggested fix:** Read `hallan_portal_session` and send `token` from that object. Do not ship the snippet until that matches.

### BUG-5 — The staff role dropdown does nothing

- **Severity:** Low
- **Location:** `staff-portal/index.html`, `#staffRole`.
- **Evidence:** The select offers Staff, Manager, and HR. `submitLogin` never reads it. The API returns the role. A person who picks HR and signs in with a staff password still lands on the staff holiday page, after the API says so. The control implies they can choose.
- **Suggested fix:** Remove the dropdown. The security model already requires that the browser does not choose the role.

### BUG-6 — Restaurant orders are marked coming soon on the home page

- **Severity:** Low
- **Location:** `home.html`, the Restaurant Orders card. `picking-sheets/index.html`, the `restaurant` tile.
- **Evidence:** The home card is a non-link with status "Coming soon". The picking app has a live restaurant module, passcode login, and order submit. Staff who only look at the home page will think the feature does not exist.
- **Suggested fix:** Either link the home card to the real flow, or change the picking tile so it is not offered until the home page says it is live.

### Checks passed — bugs

- The rota schedule's date helper is the fixed local-date version, with a comment that names the old UTC bug. BUG-1 is the clock not sharing that fix.
- No `TODO` or `FIXME` comments were found in the HTML.
