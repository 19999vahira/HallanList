# API

The pages call one API. Base URL in production is `https://hallan-picking-backend.onrender.com`. On `localhost` or `127.0.0.1` it is `http://localhost:3000`. Most picking calls append `/api/`. The rota pages put `/api/` in the path themselves.

Rate limits are not set in this repository. Webhook signature checks are not visible here. Both stay unknown until the API is reviewed. Ownership rules are in [00-security.md](00-security.md).

## Authentication

| Caller | Header | Stored as |
| --- | --- | --- |
| Picking app, after passcode and emailed code | `Authorization: Bearer` | `hallan_portal_session.token` |
| Staff holiday pages | `Authorization: Bearer` | `hallanStaffToken` |
| Rota and the clock | `Authorization: Bearer` | `rota_token` |
| Shelf-check | `Authorization: Bearer` | `hallan_mgmt_token` |
| Restaurant buyer | `Authorization: Bearer` | `hallan_restaurant_token` |

Login, signup, forgot-password, resend-code, and `GET /api/staff/departments` are called with no bearer token. `GET /api/staff/departments` is therefore public. The PDF snippet looks for `portal_token`, which the picking app does not set. Its calls will go out with no token unless something else writes that key.

A picking-list timer posts to `retail/lists/:id/track-time` with no `Authorization` header, including via `sendBeacon`.

## Staff and rota

Used by [staff-portal/index.html](../staff-portal/index.html), [rota/index.html](../rota/index.html), [rota/schedule.html](../rota/schedule.html), and [rota/clockin.html](../rota/clockin.html).

- `POST /api/staff/login`, `POST /api/staff/login/verify`, `POST /api/staff/login/resend-code`
- `POST /api/staff/signup`, `POST /api/staff/forgot-password`
- `GET /api/staff/departments` with no token
- `GET /api/staff/me`, `POST /api/staff/change-password`
- `GET /api/staff/holiday-summary`, `GET` and `POST /api/staff/leave-requests`, `PATCH /api/staff/leave-requests/:id/status`
- `GET /api/staff/holiday-employees`, `GET /api/staff/holiday-history`, `POST /api/staff/holiday-upload`
- `GET` and `PUT /api/staff/rota-state`
- `GET /api/staff/timesheet-state`, `PUT /api/staff/timesheet-entry`

A holiday or history read with an `employeeId` query must be limited to HR or to that person's own row. A rota save replaces the document the caller is allowed to see, not the whole shop, unless the caller is HR. A timesheet write must be for the person the PIN just verified, not for any id the browser names.

## Picking portal

Bearer token from `hallan_portal_session`, except the login pair.

- `POST /api/portal/login`, `POST /api/portal/login/verify`
- `GET /api/portal/modules`, `GET` `POST` `PATCH` `DELETE /api/portal/users`
- `GET /api/portal/my-ip`, `GET` `POST` `DELETE /api/portal/allowed-ips`
- Retail lists and `POST /api/retail/lists/:id/track-time`
- Warehouse damage logs and images
- `location/zones`, `shelves`, `rows`, `items`
- `goods-in` sessions, scans, items, quick-add, expiry list
- `stock/search`, `stock/info`, `stock/item`, `stock/barcode`, online-data, level overrides
- Shopify content, photos, generate, push, and the auto-run routes
- `stock/woo/product`, `stock/woo/queue`
- WhatsApp lists, contacts, send, inbound, flyer images, embedded-signup exchange

User, module, and IP routes are admin actions. Module routes must check the caller's module list even when the tile is hidden. See [00-security.md](00-security.md) invariants 2 and 3.

## Wholesale prices and flyers

Same portal bearer token, plus `POST /api/wholesale/auth/verify` and `POST /api/wholesale/auth/change` for the shared password.

- `GET /api/wholesale/pricing/`, price-lock get, block, release, and update
- Price-change list, log, and clear
- Schedule and schedule emails
- WhatsApp schedule
- Flyer images, PDFs, and `POST /api/wholesale/flyer/publish-webpage`
- `POST /api/vector-import/queue` with CSV text

A price the business will honour is the API's figure, not the number in the block payload. Public PDF URLs must stay free of staff and customer data.

## Restaurant orders

- `POST /api/restaurant/login` with a passcode, then the customer token
- `restaurant/order/search`, `suggested`, `week`, `three-month`, `substitutes`, `submit`
- `POST /api/restaurant/admin/auth/verify` and `auth/change` with the shared staff password
- `GET` `POST` `DELETE /api/restaurant/admin/customers`

Customer routes must ignore any customer id that is not the one in the token.

## Shelf-check

Bearer `hallan_mgmt_token` after `POST /api/management/login/`.

- `GET` `POST` `DELETE /api/management/zones/` and locations under a zone
- `POST /api/management/locations/:id/check/`
- `GET` and `POST /api/management/users/`

These zones are not the picking `location/zones` tree.
