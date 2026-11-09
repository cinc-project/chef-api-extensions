# Chef API extensions

This repository tracks proposed **extensions to the Cinc/Chef Server API**
(and the [`github.com/tas50/cinc-api`](https://github.com/tas50/cinc-api)
client library that fronts it) that would improve two things at once:

- **Client/server interactions** — letting clients like the CLI do their job
  more simply, cheaply, and correctly (fewer round trips, fewer workarounds,
  more correct results).
- **Advanced console experiences** — surfacing data the server already has (or
  could cheaply compute) so richer interactive views — fleet status, live
  dashboards, per-object summaries — become buildable instead of faked.

Each entry describes a rough edge a client works around today and the API
change that would remove it. These are **ideas and requests**, not shipped
features: each one is a candidate to upstream into `cinc-api` and, where it
reaches that far, into the Cinc/Chef Server itself.

This repo is **not** a place for client-side design docs — an entry here is
about the API surface, not about how any one client renders it.

## How to add one

One file per extension, named `NNNN-short-slug.md` (zero-padded,
monotonically increasing). Copy the shape of an existing entry:

- **Summary** — one or two sentences: what's awkward and what we want.
- **Where it bites** — the client code that pays the cost today, with
  `file:line` references so the entry stays anchored to real code.
- **Why it matters** — the concrete cost (extra round trips, missing
  permissions, wrong results, …).
- **Proposed improvement(s)** — ranked options, ideally with the API
  shape we'd want.
- **Workaround until then** — what a client can do on its own in the
  meantime, if anything.

Keep entries honest about status: an idea we haven't validated against the
server's real capabilities should say so.

## Index

| # | Extension | Status |
|---|-----------|--------|
| [0001](0001-user-admin-status-without-double-call.md) | Resolve a user's admin status without a second `groups/admins` call | Proposed |
| [0002](0002-group-description.md) | Carry a human-readable description on groups | Proposed |
| [0003](0003-node-run-status-in-core-api.md) | Track each node's run status (pass / fail / updated) in the core API | Proposed |
| [0004](0004-streaming-run-status-endpoint.md) | A streaming (WebSocket/SSE) endpoint for live run status | Proposed |
| [0005](0005-cookbook-list-counts-and-latest-metadata.md) | Return version counts and latest metadata on the cookbook list | Proposed |
| [0006](0006-org-list-for-non-superusers.md) | List the organizations the caller can see instead of all-or-403 | Proposed |
| [0007](0007-search-sort.md) | Server-side sort for search | Proposed |
