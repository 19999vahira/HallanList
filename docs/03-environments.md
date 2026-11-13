# Environments

One public site, one live API, and a localhost switch. There is no staging host in this repository.

| Name | Purpose | URL | What publishes it |
| --- | --- | --- | --- |
| Production pages | The portal staff open | `https://portal.hallans.co.uk` | A push to the GitHub Pages repo `19999vahira/HallanList`. The `CNAME` file is `portal.hallans.co.uk` |
| Production API | All saved data | `https://hallan-picking-backend.onrender.com` | Not this repository. Render, outside this workspace |
| Local API | A developer's own backend | `http://localhost:3000` | Whatever is running on that machine. This repo does not start it |

## What changes between them

Each interactive page sets its API base in the browser:

- Hostname `localhost` or `127.0.0.1` uses `http://localhost:3000`.
- Every other hostname, including `portal.hallans.co.uk` and a Herd `*.test` name, uses the live Render API.

Opening a file directly (`file://`) is not localhost. Those pages call production. Do not experiment with passwords, PINs, or price locks that way.

There is no feature flag, no separate log level, and no preview deployment in this repo. A branch that is served under any public hostname talks to the live database. There is no second `NEXTAUTH_SECRET` or signing key here to isolate, because this repo does not sign tokens. Token signing lives only on the API. Do not add a second frontend that shares the production API and call it staging.

## How to run the pages locally

Serve the repository root over HTTP on `localhost` or `127.0.0.1`, and run the API on port 3000 if you need data. There is no package manifest and no build. The pages are the site.

## How production is updated

GitHub Pages serves the files on the published branch. A push of HTML is a production release of the UI. Nobody approves it inside this repository: there is no workflow, no required check, and no `.gitignore`. An unauthenticated request to `https://portal.hallans.co.uk/` on 13 November 2026 returned `200` from `GitHub.com`, `content-type: text/html`, `cache-control: max-age=600`, and `access-control-allow-origin: *`. The response did not include `Content-Security-Policy`, `Strict-Transport-Security`, `X-Frame-Options`, `X-Content-Type-Options`, or `Referrer-Policy`.

## What this repo cannot tell you

Backups, point-in-time recovery, database pooling, and whether the Render service has deployment protection are properties of the API host. They are not in this folder. Secrets that must exist on the API, and must not be copied here, include the database URL, the token signing secret, mail credentials, and `WHATSAPP_APP_SECRET`.
