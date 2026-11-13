# Roles

Who can do what in the Hallan portal. Enforcement belongs on the API. The pages only decide what to show. See [00-security.md](00-security.md).

Roles are assigned in different places, and they do not inherit from each other. A portal admin is not an HR user. An HR user is not a management-shelf user. A restaurant customer is not staff.

## Roles

| Role | Who holds it | Read | Write |
| --- | --- | --- | --- |
| Anonymous | Anyone who can open the site | Public home page, published flyers, published PDFs | Nothing |
| Portal staff | A picking-sheets user with role `staff` | Modules listed on their user | Changes inside those modules only |
| Portal admin | A picking-sheets user with role `admin` | Every module, plus users, modules, and allowed IPs | Users, modules, allowed IPs, and every module |
| Employee | A `staff_employees` row whose API role is `staff` | Own holiday summary, requests, and history | Own holiday request. Own rota view only where an access grant says so |
| Manager | API role `manager` | Rota and analytics for departments on their access grant | Those departments only, if the API allows the write |
| HR | API role `hr` | All holiday requests, all employees' holiday figures, the whole rota, timesheets | Approve, decline, or partially approve requests. Upload holiday spreadsheets. Edit rota employees, shifts, access grants, and timesheet rows |
| Rota editor | Not a login role. An access-grant row matched by Login ID | Schedule for the granted departments | Shifts in those departments, if the API allows it |
| Management user | A name and password on the shelf-check login | Zones, locations, check history, the user list | Zones, locations, checks, and new management users |
| Restaurant customer | A customer passcode | That customer's catalogue and orders | That customer's order |
| Kiosk device | A dedicated clock credential. Not a person | Whether the PIN just typed matches one employee | That employee's punch for today |

## How a role is enforced

The picking app stores `{ token, name, role, modules }` in `hallan_portal_session` after `POST /api/portal/login/verify`. `portalCan()` in [picking-sheets/index.html](../picking-sheets/index.html) shows a tile when `role` is `admin` or `modules` contains the tile's key. That is display logic. The API must refuse the underlying route when the module is missing, and must refuse user and IP administration unless the caller is an admin. Admin sign-in also depends on the API's IP allow-list. The page cannot be the allow-list.

The staff portal shows a role dropdown (`staff`, `manager`, `hr`) on the sign-in form in [staff-portal/index.html](../staff-portal/index.html). The login request sends Staff ID and password only. The API returns the role. HR is sent to the holiday admin. Everyone else sees their own holiday page. The dropdown must stay unused. Do not start sending it.

The rota copies the API role into `rota_role` and the typed employee ID into `rota_login_id` in [rota/index.html](../rota/index.html). [rota/schedule.html](../rota/schedule.html) hides tabs from `rota_role`, and limits departments by matching `rota_login_id` to `accessGrants` inside the rota document. Both values are writable in the browser. The API must apply the same scope to `GET` and `PUT /api/staff/rota-state` and to timesheet routes. A person who changes `rota_role` to `hr` in `localStorage` must still receive only their own data.

Management has no second role. Any valid `hallan_mgmt_token` can load zones and create users. The API must treat user creation as a privileged action, not as "any bearer token".

Restaurant calls send `hallan_restaurant_token`. The API must scope every order and customer read to that token's customer. The shared Manage-Customers password is a second check for the customer list, not a role.

## How a role is assigned

- Portal users: an admin, in the picking app, via `POST /api/portal/users` or `PATCH /api/portal/users/:id`, with `name`, `email`, `role`, `modules`, and an optional passcode. The API must ignore that body unless the caller is an admin.
- Employees: HR's existing staff list. Signup in the rota only sets a password when the employee ID already exists and the department matches. It must not insert a person. Holiday spreadsheets can create or update holiday figures through `POST /api/staff/holiday-upload`, and only for HR.
- Rota Login ID, PIN, and department access grants: edited on the rota schedule and saved with the whole rota document. PINs are secrets. They must not come back on a later `GET` for anyone except the admin path that needs to set them, and they must never be checked in the browser.
- Management users: any signed-in management user can `POST /api/management/users/` with a name and password. Tighten this on the API if only a named owner should invite people.
- Restaurant customers: created after the shared Manage-Customers password, via `POST /api/restaurant/admin/customers`.

There is no inheritance. Granting portal admin does not grant HR. Granting HR does not grant the shelf-check login.

## Staleness

Roles copied into `localStorage` stay there until sign-out or until a `401` clears them. The staff page does not expire its own token. The rota comment says the token lasts about twelve hours, which is a server concern, not a page timer. After HR changes someone's department or role, the old token must fail on the API immediately. Waiting for the browser to be closed is not enough.
