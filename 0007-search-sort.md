# 0007 — Server-side sort for search

- **Status:** Proposed. Validated against erchef's search path; the finding
  is that a general `sort=<attribute>` is **not** a small change, because the
  index has no per-attribute fields to sort on. The options below are scoped
  to match.
- **Affects:** Cinc/Chef Server (search API and index document shape),
  `cinc-api` (`SearchService`, a `WithSort` option)
- **CLI surface that wants it:** `cinc node status`; any "stalest N nodes" or
  "most recently updated" view in the explorer or a dashboard

## Summary

Search results always come back ordered by object id. A client that wants
"the 20 nodes that checked in longest ago" must page through **every**
matching node and sort locally. We'd like the server to sort, at least on the
handful of fields fleet views actually sort by, so `rows=20` means the right
20.

## Where it bites

`cinc node status` reads every match, then sorts by `ohai_time`:

```go
// apps/cinc/cmd/node_status.go:59
for node, err := range c.Search.Nodes(cmd.Context(), query) {
	...
	statuses = append(statuses, nodeStatusOf(node, now))
}
// Most recently checked-in nodes first; nodes that never checked in
// (ohai_time 0) sort to the end.
sort.SliceStable(statuses, func(i, j int) bool {
	return statuses[i].OhaiTime > statuses[j].OhaiTime
})
```

There's no way to add a `--limit` that means anything without reading the
whole fleet first.

## Why it matters

1. **Top-N costs the full fleet.** Paging is already supported (`start`,
   `rows`, default 1000 per page), but paging through id order doesn't help a
   question about check-in order.
2. **Dashboards can't be cheap.** A "stale nodes" widget refreshing every
   minute on a 10k-node org re-reads 10k nodes every minute.

## What the server does today

- The sort is fixed: `X_CHEF_id_CHEF_X asc` for OpenSearch/Elasticsearch
  (`chef_index/src/chef_opensearch.erl:136,154`) and a fixed `&sort=` for Solr
  (`chef_index/src/chef_solr.erl:105`). The request handler reads only `q`,
  `start` and `rows` (`oc_chef_wm/src/chef_wm_search.erl:356-359`).
- Each object is indexed as **one flattened text field**, `content`, holding
  `key__=__value` tokens (`chef_index/src/chef_index_expand.erl:162,200-272`).
  Only the `X_CHEF_*` metadata fields are separate. So there is no
  `ohai_time` field to sort on, and adding `sort=` to the URL alone can't work.

## Proposed improvements

1. **A small set of sortable fields, chosen by the server.** At expand time,
   copy a whitelist of well-known values into dedicated typed fields (e.g.
   `X_CHEF_ohai_time_X` as a number for nodes, `X_CHEF_name_X` for every
   index), and accept `sort=ohai_time:desc` only for whitelisted names (400
   otherwise). Requires a reindex, but keeps the index shape otherwise
   unchanged. `cinc-api` adds `WithSort(field, desc bool)`.

2. **Answer the fleet question outside search.** If
   [0003](0003-node-run-status-in-core-api.md) lands a run/check-in model in
   Postgres, "the N nodes with the oldest check-in" becomes an indexed SQL
   query on that table, with no search index involved. For the node use case
   this may be the better home, and it would make option 1 needed only for
   generic sorting.

3. **Not recommended: sort after fetch.** The server could load every match
   from Postgres and sort before paging. It saves the client's bandwidth but
   not the server's work, and costs more than a client-side sort under load.

We'd pursue option 2 for check-in ordering and option 1 only if a second
sortable field shows real demand.

## Workaround until then

- Use partial search so reading every match is cheap: `node status` needs
  about six fields per node, not the whole object (cinc-api already supports
  `WithPartialPaths`).
- Sort locally, as today.
