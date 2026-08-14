---
name: croft-shared-data
description: Share one Croft app's SQLite database with other apps in the same workspace, or read and write another app's shared data, through the Croft data broker. Use when two Croft apps should reuse the same data instead of duplicating it — e.g. one shared contacts list behind both a CRM and an invoicing app. Covers set_database_sharing, the POST /q/<app> query broker, discovering what's shared, and public-app access rules.
license: MIT
metadata:
  author: croft
  version: "1.0.0"
---

# Sharing data between Croft apps

Each app's SQLite database is private by default. An app can **share** its database with
the rest of the workspace so other apps reuse its data rather than duplicating it (e.g. one
`contacts` list used by both a CRM and an invoicing app).

## Share this app's data with the workspace

```
set_database_sharing(app, shared: true)
```

Turn it off with `shared: false`. Owner/admin only: pass `allow_public: true` to also let
apps that have public routes read this data (off by default — a public page could otherwise
expose it to anyone).

## Use another app's shared data

`POST` to the data broker with SQL:

```
POST http://croft-data:8080/q/<that-app>
header  Authorization: Bearer $CROFT_DATA_TOKEN   (Croft injects this into every app's env)
body    { "sql": "select * from contacts where email = ?", "params": ["a@b.com"], "limit": 200 }
```

You get `{ rows, row_count, truncated }`. **Reads and writes are both allowed.** To learn a
shared app's tables, read its `migrations/` files (they're the schema). Call
`get_platform_conventions` to see the live list of what's shared in this workspace right now.

## Scheduled tasks can use shared data too

A scheduled endpoint (see the `building-croft-apps` skill) runs inside your app container
with `$CROFT_DATA_TOKEN`, so a nightly job can pull from another app's shared database
exactly like a normal request handler.

## Public apps are restricted (leak guard)

A **public** app (one with any `public_paths`) may share its **own** data, but is blocked by
default from **querying another app's** shared database — so public traffic can't reach
private workspace data. If a public app legitimately needs it, call
`request_public_data_access(app)`: an owner/admin gets it instantly; a creator's call raises
a request an owner/admin approves. (The same approval also unblocks the app's connectors.)
