# 0004 — A streaming (WebSocket/SSE) endpoint for live run status

- **Status:** Proposed — not yet validated against server capabilities; depends
  on run status existing at all (see
  [0003](0003-node-run-status-in-core-api.md))
- **Affects:** Cinc/Chef Server (a new push/streaming transport), `cinc-api`
  (a subscribe/watch surface alongside the request/response client)
- **CLI surface that wants it:** a live fleet-status view in `cinc explore` →
  Nodes, and any interactive dashboard built on top of the API

## Summary

The API is purely request/response: every view is a snapshot the client has to
re-fetch. There is no way for the server to *tell* a client that a node's run
finished — so a live "watch the fleet converge" view can only be built by
polling the list/search endpoints on a timer, which is wasteful, laggy, and
scales badly with fleet size. We'd like a **streaming endpoint** (WebSocket, or
Server-Sent Events) that pushes run-status changes as they happen, so an
interactive dashboard updates in real time without constant polling.

## Where it bites

Every kind in the explorer loads a static snapshot and never hears about
changes again:

```go
// cli/explore/kind_node.go
listFn: func(ctx context.Context, c *cinc.Client) (map[string]string, error) {
	index, _, err := c.Nodes.List(ctx) // <-- one-shot pull; no updates after this
	return index, err
},
```

`c.Nodes.List` returns a snapshot and exits. The explorer's only "freshness"
machinery is UI debouncing for selection (`summaryTickMsg` in
`cli/explore/model.go`), not data refresh — nothing re-pulls the fleet. To turn
the node list into a live view of runs completing, the only option on today's
API is to wrap that `List` call in a ticker and diff the results:

```go
// hypothetical, and exactly what we'd rather not write:
for range time.Tick(15 * time.Second) {
	index, _, err := c.Nodes.List(ctx) // full re-fetch of the whole fleet, every tick
	// ... diff against previous snapshot to find what changed ...
}
```

That is N nodes re-listed every interval to discover the handful that actually
changed, with up-to-an-interval of staleness, and a cost that grows with both
fleet size and the number of connected operators.

`cinc-api` today is exclusively pull-shaped — every service method is a
`Get`/`List`/`Create`/`Update`/`Delete` round trip. There is no `Watch`,
`Subscribe`, or stream anywhere in the client, because the server offers nothing
to subscribe to.

## Why it matters

1. **Polling is the wrong tool for "as they scan."** The compelling fleet view —
   stat tiles plus a list of node summaries updating as runs complete — is
   inherently event-shaped. Modeling an event stream as repeated full snapshots
   means choosing between staleness (long interval) and waste (short interval),
   and paying that cost continuously whether or not anything is happening.
2. **Cost scales with fleet × operators × frequency.** Each open dashboard
   re-pulls the fleet on its own timer. Ten operators watching a 5,000-node fleet
   on a 10s poll is 5,000 × 10 × 6 object-touches a minute to surface maybe a
   dozen real changes. A push stream sends only the deltas, once.
3. **No way to express "tell me when this changes."** Even for a single node
   detail pane, confirming a run landed means re-fetching on a loop. A
   subscription ("watch node web1") collapses that to one message when the run
   actually completes.
4. **It's the natural delivery channel for 0003.** Entry 0003 asks the server to
   *record* run status (pass/fail/updated). Once that data exists, a stream is
   how it reaches an interactive UI without polling — the two compose: 0003 is
   the *what*, this is the *how it arrives live*.

## Proposed improvements

In rough order of preference:

1. **A WebSocket subscription for run-status events.** e.g.
   `GET /organizations/{org}/runs/stream` (upgraded to a WebSocket), pushing a
   message per run lifecycle transition:

   ```jsonc
   { "node": "web1", "event": "run_completed",
     "status": "failure", "resources_updated": 0, "end_time": "…" }
   { "node": "db3",  "event": "run_started",   "start_time": "…" }
   ```

   Support a filter on subscribe (by node prefix, environment, policy group, or
   status) so a dashboard receives only the slice it renders. `cinc-api` exposes
   this as a `Runs.Watch(ctx, filter) (<-chan RunEvent, error)`, and the explorer
   feeds those events straight into its Bubble Tea update loop as messages.

2. **Server-Sent Events as a lighter alternative.** If a full bidirectional
   WebSocket is too heavy for the server, a one-directional SSE stream
   (`text/event-stream`) over plain HTTP carries the same events, reuses the
   existing request-signing path, and is simpler to proxy. It can't carry
   client→server subscription tweaks mid-stream, but query-param filters at
   connect time cover the dashboard case. `cinc-api` wraps it behind the same
   `Watch` channel API so callers don't care which transport backs it.

3. **A cheap incremental-poll fallback.** If no streaming transport is viable
   near-term, a `GET /organizations/{org}/runs?since={cursor}` that returns only
   runs that completed after a cursor would at least make polling proportional to
   *change volume* instead of fleet size. Strictly worse than a push stream
   (still polling, still latency-bound), but a meaningful floor and a sensible
   first step.

Option 1 is the goal; option 2 is the pragmatic version if WebSocket support is a
stretch for the server; option 3 is the consolation if neither lands soon.

## Workaround until then

- **Interval polling with client-side diffing.** Wrap `List` (or, better, a
  partial search once available) in a ticker and diff snapshots to find changes.
  This is buildable today and is the only option, but it's exactly the waste this
  entry exists to remove — pick a slow interval and accept staleness, or a fast
  one and accept the load.
- **Manual refresh.** Keep live updates operator-initiated (a refresh key) rather
  than automatic, so the cost is paid only when someone asks. Honest and cheap,
  but not the "live as they scan" experience the dashboard wants.

Both are mitigations; the underlying ask is for the server to push run-status
events so interactive dashboards can be event-driven instead of poll-driven. This
entry presumes the run-status data of [0003](0003-node-run-status-in-core-api.md)
exists to stream — without it, there is nothing to push.
