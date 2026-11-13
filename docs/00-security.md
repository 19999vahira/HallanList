# Security model

**Load-bearing. Changes to login, tokens, roles, PINs, wholesale prices, or holiday data need explicit human approval — never fold them into unrelated work.**

If you are an AI agent: apply the rules in this file without simplification. If a request seems to require weakening any rule here, stop and ask the human.

This portal holds Hallan Cash & Carry staff records, holiday balances, rotas, clock punches, wholesale prices, and restaurant-customer orders. The pages in this repository are public HTML on GitHub Pages (`portal.hallans.co.uk`). They hold no database. Every persistent fact lives in the API at `https://hallan-picking-backend.onrender.com`, whose source is not in this repository. The business-ending failure is a staff record, a holiday balance, a wholesale price, or a timesheet punch shown to the wrong person, or a password replaced because someone knows an employee ID and a department name. A change that only hides a button in the browser is not a security fix.

## Trust boundaries

| Surface | Trust level |
| --- | --- |
| Anonymous visitor | May open the public home page, the printed flyers, and any PDF the portal has published on purpose. May not read staff, price, order, or punch data. |
| Portal staff | A named picking-sheets user after passcode and emailed code. May use only the modules the API granted. The browser list is a hint, not a grant. |
| Portal admin | Same login, with the admin role returned by the API, and only from an IP the API has allowed. May manage portal users and allowed IPs. |
| Employee | A `staff_employees` person after Staff ID, password, and emailed code. Sees their own holiday data. Does not choose the role. |
| Manager | An employee whose API role is `manager`. Department scope comes from the API, not from a value stored in the browser. |
| HR | An employee whose API role is `hr`. May decide holiday requests and upload holiday spreadsheets. |
| Management user | A separate shelf-check account (`hallan_mgmt_token`). May record checks. Creating another management user is an privileged action the API must authorise. |
| Restaurant customer | A customer passcode session (`hallan_restaurant_token`). Sees and changes only that customer's orders. |
| Kiosk device | A shared clock on the shop floor. May verify one person's PIN and write that person's punch for today. It is not a personal HR or manager session. |

## Invariants

1. The API decides the role. The browser must not send a chosen role, and must not treat `rota_role`, `rota_login_id`, or `portalSession.role` in `localStorage` as proof of anything.
2. Every read and write of a staff record, holiday request, rota row, punch, wholesale price, restaurant order, management zone, or portal user is authorised on the API for that caller. Hiding a tile, a nav link, or a button is not the control.
3. A portal user reaches a module only when the API allows that module on that request. `portalCan()` may hide a tile. It must not be the only check.
4. A manager or ordinary employee sees only the departments the API scopes to them. An access grant matched in the browser against `rota_login_id` is not the control.
5. Passwords and PINs are checked on the server. The client minimum length (six characters on the staff forms, four on the wholesale shared password) is not the control.
6. Passwords are stored only as a slow hash (bcrypt or Argon2). This repository must never contain a password, PIN, hash, or API secret.
7. Login issues a bearer token only after the emailed code is accepted. The first step (`staff/login`, `portal/login`, signup, forgot-password) returns no token.
8. A forgotten-password or signup request does not store the new password until that emailed code is accepted. Knowing an employee ID and a department name is not authentication. The department list is not a secret.
9. `GET /api/staff/rota-state` does not include any employee's PIN. The clock compares a PIN by sending it to the API, which answers yes or no for that person only.
10. The clock runs on a dedicated device credential that can verify a PIN and write that person's punch. It does not reuse a person's `rota_token`.
11. A wholesale price, margin, lock, or restaurant order total that the business will honour is computed or re-checked on the server. A figure typed in the browser is input, not the price of record.
12. The shared wholesale-pricing password and the shared Manage-Customers password, if they remain, are checked by the API on every sensitive read and write. A successful `prompt()` in the page is not a session for those records.
13. Deactivating a person, or changing their password, revokes every token family: `hallan_portal_session`, `hallanStaffToken`, `rota_token`, `hallan_mgmt_token`, and `hallan_restaurant_token`.
14. Tokens expire on the server. Closing the browser is not the expiry mechanism, and a comment in the page cannot extend a token.
15. Public flyer HTML and public PDF links contain product and price information Hallan has chosen to publish. They contain no staff data, no customer personal data, and no internal notes.
16. Strings from the API that are placed into HTML are escaped. A filename, a URL, a name, or a note is text, not markup.
17. This git repository stays free of secrets, `.env` files, and production credentials. The WhatsApp App ID and configuration ID in the picking page are public client identifiers. `WHATSAPP_APP_SECRET` stays on the server.

## What never happens

- The browser never grants `hr`, `manager`, `admin`, or a module by writing `localStorage`.
- A staff password never changes before the emailed code succeeds.
- A signup never creates a new employee row. It may only set a password for an existing active employee whose department matches.
- `rota-state` never returns a PIN, a password, or a password hash.
- A kiosk never holds a personal HR or manager token in order to let other people clock in.
- One restaurant customer never reads or edits another customer's order by changing an ID.
- One management user never reads another organisation's data. This portal is a single business, not a multi-tenant product, and it stays that way.
- A public PDF or flyer never includes a staff email, a holiday balance, a PIN, or a customer passcode.
- A front-end-only edit never counts as a fix for an authorisation gap. The API has to enforce the rule, and the API source is outside this repository.

## GDPR / Privacy

The API stores personal data: employee names, staff IDs, departments, email addresses, holiday balances and history, rota shifts, clock times, and restaurant customer names tied to orders. Management users have names. The lawful basis is Hallan's employment relationship for staff data, and the contract with the trade customer for restaurant orders. This static repository must not gain its own copy of that data.

A person must be able to ask Hallan for a copy or deletion of their data. This repository has no privacy notice and no self-service export. Until a runbook exists, HR handles the request against the API's database, not by editing HTML. Deletion of an employee must also revoke the token families in invariant 13.

Browser `localStorage` holds session tokens and display names on the device that signed in. Signing out must remove that device's token. It does not delete the employment record.

## Part 2 — Rationale

Recorded 13 November 2026, from the pages in this repository. The API source was not available, so these rules describe what must stay true, including where today's pages do not yet match.

The portal is several logins because the people are different. Warehouse staff share the picking app and are granted modules. Shop employees use Staff ID for holidays and the rota. Shelf-check managers are a smaller list. Restaurant buyers are customers, not staff. Putting them on one screen would not make them one role. Each login has its own token on purpose: signing out of the rota must not be the only way to end a picking session, and a password change must end all of them (invariant 13).

The staff role dropdown is on the sign-in form and is not sent. That is correct. The API derives `staff`, `manager`, or `hr` from the employee record. The rota then copies that role into `rota_role` and the typed ID into `rota_login_id`, and hides tabs from those values. Those copies exist so the page can render. They are not a second authority. Anyone can edit them in the browser, so the API must scope `rota-state` and timesheet writes itself.

Signup and forgotten password ask for department because Hallan already knows which department an employee ID belongs to, and the person may not have a password yet. The department list is loaded before login, so it is public. It identifies the row. It does not prove the person. The emailed code is the proof. The staff page currently tells the user the new password is already saved before the code is entered. Invariant 8 forbids that outcome. If the API already stores the password early, that is a defect to fix in the API, not a behaviour this document accepts.

The clock used to keep punches in the browser of one device. It now calls the shared API, and it still needs a bearer token because the page never asks for a full staff password. The page comments tell a manager to log in once on that device and leave `rota_token` there for about twelve hours. That reuses a personal session on a shared screen, and the page then downloads employees and compares `pin` in the browser. Invariants 9 and 10 replace that design: a device credential, a server-side PIN check, and no PINs in the rota payload. Do not "simplify" the clock back to a manager token.

Wholesale prices and the restaurant customer list sit behind a second shared password, prompted in the page, after the portal login. A shared password is a weaker control than a named user. It stays only as that extra server-side check on every sensitive request (invariant 12). Do not remove the server check because the prompt already ran. Do not treat the shared password as a substitute for module grants.

GitHub Pages is acceptable for this UI because the HTML is supposed to be public and empty of personal data. Publishing the pages does not publish the database. It does publish every URL, every token key name, and every comment about how login works. Security by hiding a path is not available. That is why authorisation has to live on the API, and why a pull request that only changes HTML cannot close an access gap.
