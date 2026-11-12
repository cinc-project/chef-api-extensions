# 0009 — Expose when each object was created and last changed

- **Status:** Proposed. Validated against erchef at chef/chef-server@76adaa4:
  the database records `created_at`/`updated_at` for nearly every object, and
  no API response carries them. The API shape below is not yet validated
  upstream.
- **Affects:** Cinc/Chef Server (every object GET and collection list),
  `cinc-api` (object models), cinc-server-ng (which doesn't record the times
  yet: cinc-project/cinc-server-ng#273)
- **CLI surface that wants it:** the explorer summary panes for every kind,
  and any "recently changed" or audit view

## Summary

The server already knows when every node, role, environment, data bag item,
cookbook version, client, user, group, key and organization was created and
last changed, and who changed it. It writes those times on every create and
update, and never returns them. We'd like the API to expose them: on a single
object, on collection lists for bulk views, and without changing the object
bodies clients write back.

## Where it bites

Each summary pane in the explorer shows only what the object body holds. None of
them can say when the object appeared or last changed:

```go
// cinc-cli cli/explore/kind_cookbook.go:121-125
func cookbookVersionSummaryFields(cb *cinc.Cookbook) []summaryField {
	fields := []summaryField{{"Version", orDash(cb.Version)}}
	fields = append(fields, cookbookIdentityFields(cb)...)
	return append(fields, summaryField{"Files", count(len(cb.AllFiles()))})
}
```

```go
// cinc-cli cli/explore/kind_client.go:56-65
func clientSummaryFields(cl *cinc.APIClient) []summaryField {
	...
	return []summaryField{
		{"Type", role},
		{"Public Key", presence(cl.ChefKey.PublicKey)},
	}
}
```

```go
// cinc-cli cli/explore/kind_user.go:88-96
func userSummaryFields(u *cinc.User, userType string) []summaryField {
	return []summaryField{
		{"Type", userType},
		{"Display Name", orDash(u.DisplayName)},
		...
	}
}
```

"When was 2.3.1 uploaded?", "when did this role last change?", "is this client
new, or has it been here for years?" and "which users haven't been touched since
we migrated?" all need data the server has and won't hand out. The node pane
gets by on `ohai_time` (see [0003](0003-node-run-status-in-core-api.md)), which
is when chef-client last ran, not when anyone changed the node.

## What the server does today

**It records the times.** The schema has `created_at` and `updated_at`
(`TIMESTAMP WITHOUT TIME ZONE NOT NULL`) on:

| Object | Schema (`src/oc_erchef/schema/`) |
|---|---|
| nodes | `baseline/deploy/nodes.sql:13-14` |
| roles | `baseline/deploy/roles.sql:12-13` |
| environments | `baseline/deploy/environments.sql:12-13` |
| data bags | `baseline/deploy/data_bags.sql:12-13` |
| data bag items | `baseline/deploy/data_bag_items.sql:19-20` |
| clients | `baseline/deploy/clients.sql:14-15` |
| cookbook versions | `baseline/deploy/cookbook_versions.sql:17-18` |
| users | `deploy/users.sql:16-17` |
| groups | `deploy/groups.sql:24-25` |
| organizations | `deploy/organizations.sql:19-20` |
| containers | `deploy/containers.sql:12-13` |
| keys | `deploy/multiple_keys.sql:18` (`created_at`), `deploy/keys_update_tracking.sql:8` (`updated_at`) |

Most of these tables also have `last_updated_by`, the authz id of the actor
that made the last change. Cookbook artifact versions have `created_at` and
`created_by` only (`deploy/cookbook_artifact_versions.sql:11-12`), since they
can't change. Policies, policy revisions and policy groups have
`last_updated_by` but **no timestamps** (`deploy/policies.sql`,
`deploy/policy_revisions.sql`, `deploy/policy_groups.sql`).

**It keeps them current.** The object modules stamp them on every write, for
example:

```erlang
%% apps/chef_objects/src/chef_node.erl:209,214
Object#chef_node{created_at = Now, updated_at = Now, last_updated_by = ActorId}.
Object#chef_node{updated_at = Now, last_updated_by = ActorId}.
```

and the same in `chef_user.erl:450,455`, `chef_cookbook_version.erl:135,140`
and `chef_key.erl:99,103`.

**It never returns them.** Nodes, roles, environments and data bag items are
served from the stored `serialized_object`, which holds only what the client
sent. Users are assembled from the username, email and a whitelist of stored
fields (`chef_user.erl:260-273`). Keys return name, public key and expiration
only (`chef_key.erl:118-130`). erchef sends no `Last-Modified` header either.

## Why it matters

1. **The console can't answer basic "when" questions.** Upload times, last
   changes and account age are core to an object detail view, and to any audit
   or "what changed recently" view.
2. **Stale-object cleanup is guesswork.** Finding roles nobody has touched in
   two years, or clients created and never used, needs a change time. Today
   the only option is querying Postgres directly, which needs shell access to
   the server host.
3. **Clients can't cheaply tell whether an object changed.** With no change
   time or validator, a client re-fetches the full body to find out.
4. **The data is already paid for.** The server writes these columns on every
   change. Exposing them costs a read of a column it already loads.

## Proposed improvements

1. **Response headers on single-object GET.** Return the times as headers, so
   object bodies (which chef-client and knife write back) don't change:

   ```
   Last-Modified: Wed, 12 Nov 2026 02:07:41 GMT     (updated_at)
   X-Ops-Created-At: 2024-03-01T17:22:09Z           (created_at)
   ```

   `Last-Modified` is standard HTTP, and it also enables `If-Modified-Since`
   conditional GETs (`304 Not Modified`) for clients that cache. The change is
   additive: clients that don't read the headers are unaffected.

2. **Opt-in times on collection lists.** For list and dashboard views, a query
   parameter that adds the times to each entry instead of the bare URL:

   ```
   GET /organizations/acme/roles?times=true
   {"web": {"uri": ".../roles/web", "created_at": "...", "updated_at": "..."}, ...}
   ```

   Without the parameter the list is unchanged, so existing clients keep
   working. This lets a console sort by "recently changed" with one request
   instead of one per object.

3. **Who made the last change.** Resolve `last_updated_by` to an actor name
   (`X-Ops-Last-Updated-By: alice`, and `last_updated_by` in the opt-in list
   form), shown only to callers who can read that actor. This is more involved,
   since the column holds an authz id that has to be mapped back to a user or
   client, which may since have been deleted. It's a separate step after 1 and 2.

4. **Fill the gaps.** Add `created_at`/`updated_at` to policies, policy
   revisions and policy groups, which have none today.

Options 1 and 2 are preferred first steps. They expose data the server already
loads, change no request or response body, and cover both the detail pane and
the list views.

## Workaround until then

None through the API. An operator with shell access to a Chef Infra Server host
can read the columns from Postgres (`opscode_chef` database), which is what
cinc-server-backup's planned host tier does to carry the times across a
migration. cinc-server-ng doesn't record them yet; cinc-project/cinc-server-ng#273
tracks adding that, so it can serve this extension once the shape is agreed.
