---
name: building-croft-apps
description: Build, deploy, and maintain web apps on a Croft workspace through the Croft MCP. Use whenever the user asks to create, build, change, ship, deploy, or fix an app on Croft (oncroft.net). Covers the required create_app → write_files → deploy → poll-status workflow and Croft's app conventions (Node/Hono/SQLite, $PORT, /healthz, no auth code, secrets, file uploads, public paths, scheduled tasks).
license: MIT
metadata:
  author: croft
  version: "1.0.0"
---

# Building apps on Croft

Croft is a passwordless PaaS. Each workspace has its own private server; you build and
run apps on it entirely through the **Croft MCP** tools. Every app is small, self-
contained, and looks identical to the infrastructure so Croft can host, secure, back up,
and maintain it.

## Always start with `get_platform_conventions`

Before building or changing anything, call **`get_platform_conventions`**. It returns the
authoritative, up-to-date golden path **plus this workspace's live connectors and shared
apps**. This skill is a faithful summary so you know what to expect — but the tool is the
source of truth. Follow it.

## The workflow — "create an app" is a whole sequence, not one call

When asked to create/build/deploy an app, run the **entire** sequence in the same turn and
do not stop partway:

1. `get_platform_conventions`
2. `create_app` — provisions an empty repo + URL only. It implements nothing and does
   **not** make the app live.
3. `write_files` — write the **actual implementation** the user asked for, following the
   conventions below.
4. `deploy` — build + publish.
5. `status` — poll until `deploy_state` is `succeeded` (it passes through `queued` /
   `running` first — those are **not** done) and `health` is `healthy`.
6. Only now tell the user it's live — cite the release version, deploy job id, final
   deploy state, and URL.

**Hard rules — do not violate:**
- Provisioning a repo is **not** completion. Scaffold-only = empty; you've built nothing.
- **Never** claim an app is live/deployed/working while the deploy is `queued`/`running`
  or before you've seen `deploy_state: succeeded`. `status` returns a `live` boolean for
  exactly this — trust it, not the existence of a URL/repo/release.
- After `write_files` for a build request, call `deploy` in the same turn unless the user
  explicitly asked to only edit source without shipping.

## App conventions (what every Croft app must follow)

1. **One process, one container.** Listen on `$PORT`. No daemons; no self-run cron (use
   scheduled tasks, below).
2. **Default stack:** Node 22 + Hono + server-rendered HTML (htmx ok) + better-sqlite3.
   Python/Flask is fine when it fits. No SPA frameworks unless the user insists.
3. **Database:** SQLite only, at `/data/app.db`, WAL on. Numbered `.sql` migrations in
   `migrations/`, run at boot.
4. **`GET /healthz` → 200 `ok`**, no auth dependency. Required.
5. **No auth code, ever.** The signed-in user arrives as `req.header('remote-email')` /
   `remote-user` — the Croft proxy guarantees these. For an admin view, check the
   **workspace role** in `remote-groups`, which the app cannot forge: it contains a
   `_role:<role>` token (`_role:owner`, `_role:admin`, `_role:creator`, `_role:member`) —
   treat `_role:owner`/`_role:admin` as workspace admins. An app's own `ADMINS` env list
   is app config, **not** a security boundary — use `_role:` for "is this a workspace admin".
6. **Secrets via env only** (`set_secret`) — never in code or the repo.
7. **File uploads:** POST the file (multipart field `file`) to
   `http://croft-storage:8080/put` with header `X-Croft-App: <your-app-name>`; you get
   `{url, key}` back — store the `url` and render it. Croft holds the storage credentials
   and validates size (≤25 MB) + type. Never write user uploads to disk.
8. **Timezone:** store UTC; render in the browser or via the account TZ env.
9. **Keep it small.** No build steps beyond what Nixpacks infers; vendor nothing. If a
   feature needs a queue, a second service, or Postgres — stop and tell the user Croft v1
   doesn't do that.

## Public pages & webhooks

Apps are **private by default** (login required). To expose specific paths to the internet
(a webhook, a public form), use **`set_public_paths`** — list only the specific paths;
root/wildcard is rejected. Keep results/admin on an **unlisted** path so Croft's login
shields it. On any public path assume hostile traffic: validate + escape all input, use
parameterised queries, add CSRF to public POSTs, never render secrets or internal errors.

## Scheduled tasks (cron)

For anything recurring (a daily digest, a nightly sync, cleanup) do **not** write your own
scheduler — it dies on deploys/reboots. Add a POST endpoint (e.g. `/tasks/digest`), keep
it **out** of public_paths, and register it with
`set_schedules([{name, cron, path}])` (5-field UTC cron). Croft calls it on schedule with
a signed `X-Croft-Scheduled` header; make the endpoint idempotent.

## Other things this app might need

- **Talk to an external API** (Stripe, a CRM, email, Azure DevOps…): use the
  **`croft-connectors`** skill — always via a connector, never a vendor SDK or a stored key.
- **Share data with another app** in the workspace, or read another app's data: use the
  **`croft-shared-data`** skill.

## Ownership & access (multi-person workspaces)

The workspace owns its apps; access is **grant-based**. You can build/deploy/edit any app
you've been **granted access to** (creating one grants you access); an admin brings another
creator onto an app by granting access. `delete_app` is owner/admin-only. `list_apps` shows
the apps you can reach. Use `grant_access` / `invite_user` (admin) to manage people.
