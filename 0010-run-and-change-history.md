# 0010 — Record runs and config changes, and show what each change led to

- **Status:** Proposed. The two data sources are validated against
  chef/chef@0bc3cd7345 (client) and cinc-server@04e58d6bd (erchef and nginx):
  chef-client already reports every run to the Chef server by default, and
  erchef already builds an event for every config write. Neither is stored.
  The storage model and API shapes below are not yet validated upstream.
- **Affects:** Cinc/Chef Server (ingest on an existing route, new tables, new
  read endpoints), cinc-server-ng, `cinc-api` (new `Runs` and `Changes`
  services)
- **Builds on:** [0003](0003-node-run-status-in-core-api.md) (run status),
  [0004](0004-streaming-run-status-endpoint.md) (live stream),
  [0009](0009-object-timestamps.md) (object times)
- **Console surface that wants it:** the fleet dashboard, a new Changes page,
  and a change detail page
- **Client requirement:** none. No client config, no report handler, no
  Automate. Upgrading the server is enough.

## Summary

The most common thing an operator does is make a change (push a policy, edit a
role's run list, move an environment's cookbook pin, upload a cookbook) and
then ask three questions: did it reach the nodes, did their runs succeed or
fail, and did anything get worse? The server sees both halves of that. Every
config write goes through erchef, and every chef-client already POSTs a report
of every run to the server's own URL by default. The server keeps neither.

We'd like the server to:

1. store a summary of each run from the route clients already call,
2. record each config change with its diff and the nodes it targets,
3. link each targeted node's first run after a change to that change, and
4. serve read APIs shaped for fleets of tens of thousands of nodes.

## The workflow this serves

Three console views, prototyped against a simulated fleet of about 6,000 nodes:

- **Fleet timeline.** Every check-in in the org as one dot, in one band per
  environment, with nodes ordered by role or policy so a failing role shows
  up as a red streak. Under it: check-ins per time bucket with a failure-rate
  line, a strip counting config changes per bucket, and a few rows for the
  changes worth watching (yours, and any that are failing).
- **Changes.** Every change with its outcome: how many targeted nodes
  succeeded, failed, are still waiting, or are offline, plus the failure rate
  on those nodes before and since.
- **Change detail.** What the change led to (succeeded / failed / waiting /
  offline), the check-ins on just the targeted nodes in the order they ran it,
  failure rate and median run time before vs since, failures grouped by error,
  and the other changes that touched the same nodes.

Every one of these depends on two records the server doesn't keep (runs and
changes) and one link between them (which run applied which change).

## Where it bites

The dashboard can only show what a node search returns, and that has no run
outcome and no change history:

```ts
// cinc-console lib/cinc/search.ts:77-87
const FLEET_KEYS: Record<string, string[]> = {
  name: ["name"],
  ohai_time: ["ohai_time"],
  run_list: ["run_list"],
  policy_name: ["policy_name"],
  policy_group: ["policy_group"],
  chef_version: ["chef_packages", "chef", "version"],
  chef_environment: ["chef_environment"],
  platform: ["platform"],
  platform_version: ["platform_version"],
};
```

Every fleet stat is derived from `ohai_time`, which says when a node last
started a run, not how it went (`cinc-console lib/cinc/fleet.ts:146-168`). To
look live, the dashboard re-runs that full-fleet search every 10 seconds
(`components/fleet-dashboard.tsx:43-46,190`), which costs the same whether one
node changed or none did, and still can't show a failure or a change.

## What the server does today

### Runs: clients already send them, and the server drops them

chef-client registers its data collector on every run
(`lib/chef/client.rb:251`), and with no configuration at all it is on and
points at the Chef server:

```ruby
# chef-config/lib/chef-config/config.rb:1110-1118
default(:server_url) do
  if config_parent.solo_legacy_mode || config_parent.local_mode
    nil
  else
    File.join(config_parent.chef_server_url, "/data-collector")
  end
end
```

`ConfigValidation.should_be_enabled?` (`lib/chef/data_collector/config_validation.rb:74-110`)
returns true in client mode unless why-run is on or `data_collector.mode`
excludes client runs. No token is needed: the request is signed with the
client's own key.

The `run_converge` message it sends at the end of each run
(`lib/chef/data_collector/run_end_message.rb:47-79`) carries what every view
above needs: `run_id`, `node_name`, `start_time`, `end_time`, `status`,
`total_resource_count`, `updated_resource_count`, `expanded_run_list`,
`policy_name`, `policy_group`, `cookbooks`, the `error` class and message, and
the node object. For Policyfile nodes, that node object includes the exact
`policy_revision` used (`lib/chef/policy_builder/policyfile.rb:210`). A
`run_start` message goes out at the beginning of the run.

On the server, the route exists only when Automate is configured. It checks
the request signature and forwards the request to Automate:

```nginx
# cinc-server omnibus/.../nginx/nginx_chef_api_lb.conf.erb:149-160
<% if node['private_chef']['data_collector']['root_url'] -%>
location ~ "^/organizations/([^/]+)/data-collector$" {
  ...
  access_by_lua_block { validator.validate("POST") }
  ...
  proxy_pass <%= ... %>://data-collector;
}
<% end -%>
```

Without Automate the client gets a 404, stops sending for that run, and logs
that this "is normal if you do not have Chef Automate"
(`lib/chef/data_collector.rb:175-200`).

A second, older path exists too: `Chef::ResourceReporter`
(`lib/chef/client.rb:278`, `lib/chef/resource_reporter.rb:76,147`) posts to
`/organizations/{org}/reports/nodes/{node}/runs` for the retired Chef Reporting
add-on, and is on by default (`enable_reporting`,
`chef-config/lib/chef-config/config.rb:622`). It also gets a 404 today. The data
collector message is richer, so it's the preferred source.

### Changes: erchef builds the event, then sends it away

After every successful non-GET request on nodes, roles, environments,
cookbooks, data bags, policies, clients, users, groups, keys and ACLs,
erchef builds an action message and posts it to the data collector
(`src/oc_erchef/apps/oc_chef_wm/src/oc_chef_data_collector.erl:9-55`). The
message (`oc_chef_action.erl:36-80`) has the entity type and name, parent (for
example the policy group of a policy), task (create / update / delete),
requestor name and type, and `data`, which is the request body. It has the new
state only, not the old, and erchef stores none of it.

## Why it matters

1. **"Did my change work?" has no answer without a second product.** The
   console can show that a node checked in. It can't show that the node's run
   failed, or that the failure started right after a role edit.
2. **Failures are invisible.** A node whose run fails looks the same as one
   that succeeded until it goes stale (see 0003).
3. **Changes have no history.** Nothing records who changed a role, what
   changed, or which nodes it reached. Policies, policy groups and revisions
   don't even have timestamps (see 0009).
4. **The data is already paid for.** Clients already POST every run to the
   server, and erchef already builds every change event. The work is to stop
   dropping both.
5. **Polling full snapshots doesn't scale.** Large fleets need queries bounded
   by time and filters, not a full node search every few seconds.

## Proposed improvements

### 1. Accept run reports on the route clients already call

erchef answers `POST /organizations/{org}/data-collector` itself, using the
signature check nginx already does. When Automate is configured it keeps
forwarding the request unchanged, so existing Automate users lose nothing.

On `run_start`, insert a run row with `status = running`. On `run_converge`,
read the summary fields out of the body as it streams in and discard the
resource list and the node body:

| Column | From |
|---|---|
| `run_id`, `node_name`, `org_id` | message |
| `start_time`, `end_time`, `received_at` | message, server clock |
| `status` | `running` / `success` / `failure` |
| `resources_total`, `resources_updated` | `total_resource_count`, `updated_resource_count` |
| `error_class`, `error_message` | `error`, message truncated to a few hundred bytes |
| `policy_name`, `policy_group`, `policy_revision` | message, node `automatic.policy_revision` |
| `run_list_ref`, `cookbooks_ref` | expanded run list and cookbook versions, stored once per distinct value and referenced by id |
| `client_version` | node `chef_packages.chef.version` |

Return `202 Accepted` before any further processing. Partition the table by
day. A `running` row with no end after twice the node's usual interval is a
run that never finished. That is 0003 option 3 for free.

### 2. Record every config change

On the write path that already builds the action message, also write a change
record. The handler already loads the current object to check permissions and
apply the update, so it has the before state too:

```jsonc
{
  "id": "chg_8f2c…",
  "at": "2026-11-17T19:16:04Z",
  "actor": { "type": "user", "name": "alice" },
  "type": "policy_push",            // see table below
  "entity": { "type": "policy_group", "name": "prod", "child": "jobs" },
  "diff": [{ "path": "revision_id", "before": "9d0e7c2…", "after": "1a6f4b8…" }]
}
```

| Change type | Trigger | Tracked fields |
|---|---|---|
| `node_edit` | node PUT where a tracked field differs, whoever made it (`chef-client -r` counts) | `run_list`, `chef_environment`, `policy_name`, `policy_group` |
| `role_edit` | role PUT | `run_list`, `env_run_lists`, default/override attribute paths |
| `environment_edit` | environment PUT | `cookbook_versions`, default/override attribute paths |
| `policy_push` | `PUT /policy_groups/{g}/policies/{p}` | revision id |
| `cookbook_upload` | cookbook version or artifact create/delete | name, version or identifier |
| `data_bag_item_edit` | item PUT/POST/DELETE | item id only. Never values, since these often hold secrets. |

Attribute diffs record the paths that changed and truncated values. A node's
routine end-of-run save changes none of the tracked fields, so it records
nothing.

### 3. Work out the target nodes when the change is written

Store the set of nodes each change should reach, along with how exact that
set is:

| Change type | Targets | Exactness |
|---|---|---|
| `policy_push` | nodes with that `policy_name` and `policy_group` | exact |
| `node_edit` | that node | exact |
| `environment_edit` | nodes in that environment, narrowed to those whose last run used a cookbook whose pin changed (or all of them for attribute changes) | exact for attributes, good for pins |
| `role_edit` | nodes whose last expanded run list includes the role, nested roles included | good: uses the expanded run list from each node's last run (section 1) |
| `cookbook_upload` | nodes whose last run used that cookbook in an environment whose pin allows the new version | approximate, unless the server re-runs dependency resolution |

The node-to-roles and node-to-cookbooks indexes needed here come from run
ingest, so they cost nothing extra.

### 4. Link each run to the change it applied

When a `run_converge` arrives for node N with start time S, check the open
changes that target N and were made before S. That run is the one that applied
them:

- a successful run closes the change for N (`succeeded`),
- a failed run marks N `failing` and leaves the change open, so later runs
  update it,
- a newer change that targets N marks the older one `superseded` for N.

For policy pushes, match on revision instead of time: the run applied the push
if its `policy_revision` equals the pushed revision. This is exact.

Store one row per (change, node): `state`, `first_run_id`, `first_run_at`,
`last_run_id`. The outcome counts, the before/since comparison, the "after X"
tags on runs and the change history for a set of nodes are all queries over
this table.

### 5. Read APIs

All are under `/organizations/{org}`. All return measurements, not verdicts:
the console decides what counts as "failing" or "slow".

**5.1 `GET /runs`**: the run feed, in compact column form so a dashboard can
draw one dot per run.

```
?from=…&to=…&env=prod&status=failure&node=…&targets_of=chg_8f2c…&cursor=…
```

```jsonc
{
  "nodes": ["web-prod-001", "web-prod-002", "…"],    // referenced by index below
  "runs": {
    "node":     [0, 1, 0],
    "start":    [1794971000, 1794971012, 1794971903],
    "status":   ["s", "f", "s"],                    // s / f / r (running)
    "updated":  [0, 1, 4],
    "duration": [41.2, 18.0, 44.9],
    "change":   [null, "chg_8f2c…", "chg_8f2c…"]    // the change this run applied, if any
  },
  "next": null
}
```

`targets_of` narrows the feed to the nodes a change targets, which is what the
change detail page draws. When a window has more runs than `max_points`
(default 100,000), the response carries failed runs in full and successful runs
as counts per cell (`"cells": {"t_bucket": …, "node_rows": …, "counts": [...]}`).
A 50,000-node fleet produces about 200,000 runs an hour, so the dashboard needs
this mode.

**5.2 `GET /runs/histogram`**: run counts per time bucket.

```
?from=…&to=…&bucket=300&env=prod
→ {"bucket": 300, "series": [{"t": 1794970800, "success": 812, "changed": 41, "failure": 3}, …]}
```

**5.3 `GET /changes`**: the change list.

```
?from=…&to=…&actor=me&state=open|failing|done&touching=chg_…&cursor=…
```

Each entry carries the change record plus an outcome summary:

```jsonc
{
  "id": "chg_8f2c…", "at": "…", "actor": {…}, "type": "policy_push", "entity": {…},
  "targets": { "total": 197, "exactness": "exact" },
  "outcome": { "succeeded": 102, "failing": 19, "waiting": 73, "offline": 3, "superseded": 0,
               "expected_by": "2026-11-17T19:41:00Z", "done_at": null },
  "comparison": {
    "before": { "window_s": 3600, "runs": 1180, "failure_rate": 0.011, "median_duration_s": 34.0 },
    "since":  { "window_s": 1520, "runs": 121,  "failure_rate": 0.157, "median_duration_s": 36.5 }
  }
}
```

`touching=chg_…` returns the other changes that targeted any of the same nodes,
each with an `overlap` count. This is the change history shown on the detail
page. `expected_by` comes from each target node's median gap between runs.

**5.4 `GET /changes/{id}`**: one change: everything in 5.3, plus the full diff
and the failures grouped by error:

```jsonc
"errors": [{ "class": "Chef::Exceptions::Service", "message": "service[sidekiq] failed to start…",
             "nodes": 19, "sample": ["worker-prod-014", "…"] }]
```

**5.5 `GET /changes/{id}/targets`**: the targeted nodes, one row each, with
`state`, `first_run_at` and `first_run_id`. Paginated, and filterable by
`state`.

**5.6 Live updates.** The stream from 0004 carries two event types, `run`
(same fields as one row of 5.1) and `change` (same fields as 5.3), and accepts
`since=<cursor>` so a client that reconnects can catch up instead of reloading.

**5.7 `GET /nodes/{name}/runs`**: one node's run history. This is 0003 option 2,
served from the same table.

### 6. Permissions

- A run is visible to callers who can read its node.
- A change is visible to callers who can read the changed object.
- Target lists and outcome counts are filtered by node read access.

Checking ACLs per node is expensive at 50,000 nodes, so: if the caller can read
the `nodes` container, and no node has a narrower ACL, skip per-node checks.
Otherwise filter per node, more slowly. Per-node ACLs are rare in practice.

### 7. Volume and retention

At 50,000 nodes running every 15 minutes, that's about 4.8 million runs a day,
roughly 0.5 GB a day at about 100 bytes per summary row. Proposed defaults:

- raw runs kept for 14 days, in day partitions,
- hourly rollups (counts per environment and per change) kept for 180 days,
- change records and their target rows kept for 365 days.

All three are server settings. Ingest adds two signed POSTs per run, about 110
requests a second at that size. The `run_converge` body is often hundreds of
KB, so the ingest path should parse it as it streams in, and nginx's
`client_max_body_size` for this route needs to allow it.

## What it can't see

| Case | Effect | Handling |
|---|---|---|
| Client points `data_collector.server_url` directly at Automate | its runs never reach the Chef server | Fall back to the node's end-of-run save. That shows success and its time, but no failures. Mark these nodes as having partial run data. |
| `data_collector.mode :solo`, why-run | no run messages | same fallback |
| Client older than the data collector (pre-12.11) | no run messages | same fallback |
| Client clock skew | `start_time` off | order by `received_at`, keep `start_time` for display, and clamp it to no later than `received_at` |
| Role and cookbook targets | depend on each node's last run | flagged by `exactness`, and refreshed as runs arrive |

## Rollout order

1. Run ingest, `GET /nodes/{name}/runs` and `GET /runs/histogram`. This alone
   delivers 0003.
2. Change records and target resolution, then the run-to-change link.
3. `GET /changes`, `GET /changes/{id}` and `/targets`, and `GET /runs` with
   `targets_of`.
4. `run` and `change` events on the 0004 stream.
5. Density mode on `GET /runs`, for large fleets.

## Workaround until then

There is no client-side workaround that doesn't break the no-client-config
rule. On the server, an operator can set `data_collector['root_url']` to a small
service of their own. nginx then forwards client run reports to it, and erchef
posts its action messages to it, so the service can store both and serve its
own copy of these APIs. That is what this proposal asks the server to do itself.
