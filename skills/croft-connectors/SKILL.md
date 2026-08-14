---
name: croft-connectors
description: Call external APIs (Stripe, Azure DevOps, a CRM, email, payments, anything) from a Croft app through Croft connectors — never by installing a vendor SDK or holding an API key. Use whenever a Croft app must talk to a third-party service. Covers list_connectors/describe_connector, grant_connector, the freeform POST /call/<connector> data broker, custom request headers and raw bodies, propose_connector, and public-app approval.
license: MIT
metadata:
  author: croft
  version: "1.0.0"
---

# Using external services on Croft (connectors)

**Rule: every external API call goes through a Croft connector.** Do NOT `npm install` /
`pip install` a vendor SDK, and do NOT ask the user for, or store, an API key in the app.
Croft holds the credential and attaches it for you at call time — the app never sees it. An
app that bundles a third-party SDK or a stored key is wrong on Croft and will be rejected.

You already know these vendors' HTTP APIs — build plain HTTP requests through the broker
instead of reaching for an SDK.

## Setup

1. **See what's connected:** `list_connectors` / `describe_connector`.
2. **Give your app access:** `grant_connector(app, connector)` (instant for a private app).
3. **If the service isn't connected yet:** `propose_connector` from the API's docs (base
   URL, auth style, a few operations). A workspace member then adds the key in the panel.
4. **If your app has PUBLIC pages**, it's blocked from connectors by default (leak guard).
   After granting, call `request_public_data_access(app)` — an owner/admin approves it
   (instantly if you are one) before the connector will work.

## Making calls (the data broker)

Call **any** endpoint of the connector's API — you are not limited to listed operations:

```
POST http://croft-data:8080/call/<connector>
header  Authorization: Bearer $CROFT_DATA_TOKEN   (Croft injects this into every app's env)
body    { "method": "POST", "path": "/v1/customers", "params": { "email": "a@b.com" } }
```

Croft attaches the connector's key, sends the request to the connector's own domain only
(SSRF-guarded, host-locked), and returns `{ status, body }`.

- **GET/DELETE** → `params` become the query string.
- **POST/PUT/PATCH** → `params` become the JSON body.
- `{placeholders}` in `path` are filled from `params`.
- Named shortcuts also work: `POST /call/<connector>/<operation>`.

### Custom headers and raw bodies

For APIs that need a specific content type or a non-object body, add optional `headers` and
`body` fields (the body is sent verbatim, replacing `params`). Example — an Azure DevOps
work-item PATCH (needs `application/json-patch+json` and a JSON **array**):

```
POST http://croft-data:8080/call/azure-devops
body {
  "method": "PATCH",
  "path": "/{project}/_apis/wit/workitems/{id}?api-version=7.1",
  "params": { "project": "Acme", "id": "42" },
  "headers": { "Content-Type": "application/json-patch+json" },
  "body": [ { "op": "add", "path": "/fields/System.Title", "value": "New title" } ]
}
```

The connector's credential is always attached **last** and can't be overridden by your
headers.

### Example — Stripe

```
POST http://croft-data:8080/call/stripe
body { "method": "POST", "path": "/v1/customers", "params": { "email": "a@b.com" } }
```

## Handle "not enabled yet" gracefully

`no_connector` (awaiting its key) or `no_grant` (app not granted) mean setup isn't
finished — show a calm "this integration isn't switched on yet" message, not an error.
Access is per **app**: once on, it works for everyone who can open the app.
