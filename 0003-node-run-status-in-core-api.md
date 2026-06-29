# 0003 — Track each node's run status (pass / fail / updated) in the core API

- **Status:** Proposed — validated against today's behavior (the core Cinc/Chef
  Server demonstrably does *not* carry run status); the proposed change is a
  server-side feature request, not yet validated as feasible upstream
- **Affects:** Cinc/Chef Server (core node/run model), `cinc-api` (`Node` model,
  a new run/status surface)
- **CLI surface that wants it:** the explorer node summary pane
  (`cinc explore` → Nodes), and any future fleet-status / dashboard view

## Summary

The console and CLI can tell you *when* a node last reported (`ohai_time`), but
not *how that run went*. There is no field anywhere on the node object for the
outcome of the last chef-client run — success, failure, or how many resources
were updated. The only honest fact we can surface is "last seen N hours ago."
Today the outcome can only be recovered by bolting a report handler onto every
node or standing up Chef Automate alongside the server. **We want run status to
be a first-class part of the core API** so a node fetch answers "did the last
run pass, fail, or change anything?" without any out-of-band machinery.

## Where it bites

The node kind's summary pane already wants to be a fleet-status view, but it can
only render a timestamp because the timestamp is all the object holds:

```go
// cli/explore/kind_node.go
func nodeSummaryFields(n *cinc.Node, now time.Time) []summaryField {
	var ohai float64
	if v, ok := n.Automatic.Dig("ohai_time"); ok {
		if f, ok := v.(float64); ok {
			ohai = f
		}
	}
	return []summaryField{
		{"Environment", orDash(n.Environment)},
		{"Policy Group", orDash(n.PolicyGroup)},
		{"Run List", list(n.RunList, 0)},
		{"Client Version", orDash(n.Automatic.GetString("chef_packages", "chef", "version"))},
		{"Last Scan", relativeTime(ohai, now)}, // <-- the only run signal we have
	}
}
```

`Last Scan` is derived from `ohai_time`, and `ohai_time` is set when **ohai runs
at the start of the converge** — but the node object is only persisted at the
**end** of the run. So even this one signal is subtly wrong for our purposes:

```go
// cli/explore/summary.go — relativeTime
// A non-positive epoch means we never saw the node ...
```

```go
// cli/explore/kind_node.go — nodeTitle
// A node we've never scanned has no platform, so it shows the bare name.
```

The comments already reach for scan/run vocabulary the data can't actually back.
A node that **started a run and crashed before the final save** sends nothing to
the server, so it is indistinguishable from a node that is simply offline: both
just have a stale `ohai_time`. A node whose run **failed but completed** saves a
node object that looks identical to a clean success.

The relevant `cinc-api` shape today has nowhere to put any of this:

```go
// github.com/tas50/cinc-api — nodes.go
type Node struct {
	Name        string
	Environment string
	RunList     []string
	Normal      Attributes
	Default     Attributes
	Override    Attributes
	Automatic   Attributes // ohai_time lives in here; nothing else about the run
	PolicyName  string
	PolicyGroup string
	// ...no last-run status, no updated-resource count, no run id, no run start/end
}
```

## Why it matters

1. **The single most-wanted fleet fact is missing.** "Which nodes are failing?"
   is the first question an operator asks of a fleet, and the core API can't
   answer it. The CLI can show *staleness* as a weak proxy, but staleness
   conflates three distinct states — clean success, failed-but-completed run, and
   crashed/never-finished run — into one "last seen" timestamp.
2. **No notion of a run that started but never finished.** Because the node is
   saved only at the end of a converge, a run that begins and then hangs, crashes,
   or is killed leaves no trace on the server at all. There is no in-progress
   state and no run record, so "stuck" runs are invisible.
3. **"Updated" is as interesting as "passed."** Configuration management cares
   whether a converge *changed* anything (resources updated) versus ran clean
   (all up to date). That count is computed during every run and then discarded
   as far as the core API is concerned.
4. **The workarounds are heavyweight and non-portable.** Recovering run status
   today means either (a) a `report`/`exception` handler on every node writing
   status back into a node attribute — a per-fleet convention the API doesn't
   guarantee and the CLI can't assume — or (b) running Chef Automate as a
   separate data warehouse with its own API and its own deployment. Both push a
   fact the server already computes out into optional infrastructure. **This
   should not require a special handler or a second product.**
5. **Inconsistent with the data the run already produces.** Every chef-client run
   ends knowing its status, its elapsed time, and its updated-resource count. The
   server receives the node save that closes the run; it is the natural place to
   record the outcome of the run that produced it.

## Proposed improvements

In rough order of preference:

1. **Carry last-run status on the node object, end to end.** Have the server
   record the outcome of the run that produced each node save and return it on
   `GET /organizations/{org}/nodes/{name}`. A minimal, useful shape:

   ```jsonc
   {
     "last_run": {
       "run_id":          "…",        // ties to a run record if/when one exists
       "status":          "success",  // success | failure
       "start_time":      "…",        // run start (when ohai ran)
       "end_time":        "…",        // run end (when this save happened)
       "elapsed_seconds": 42.7,
       "resources_total":   120,
       "resources_updated":   3,       // 0 == ran clean, >0 == made changes
       "error": { "message": "…", "backtrace_excerpt": "…" } // present on failure
     }
   }
   ```

   `cinc-api` exposes this as a `Node.LastRun *RunStatus` field, and
   `nodeSummaryFields` adds `Status` (`success`/`failure`, paired with an icon —
   never color alone) and `Changed` (`resources_updated`) rows next to `Last
   Scan`. One fetch, no handler, no Automate.

2. **A run-status / run-history endpoint.** If the outcome shouldn't live on the
   mutable node object, expose it adjacently — e.g.
   `GET /organizations/{org}/nodes/{name}/runs` returning recent run records
   (the same fields as above, newest first), and/or a fleet-wide
   `GET /organizations/{org}/runs?status=failure`. This is what unlocks a real
   dashboard: counts of failing / changed / stale nodes become one query each
   instead of N node fetches. `cinc-api` adds a `RunsService`.

3. **At minimum, distinguish "ran" from "saved."** Even without a full run model,
   recording a server-side run-start marker (so a started-but-never-finished run
   is detectable as start-without-end after a timeout) would remove the worst
   ambiguity — a crashed run currently looks exactly like an offline node.

Option 1 is the smallest change that answers the question operators actually ask;
option 2 is what a fleet-status/dashboard view ultimately needs. They compose:
last-run status on the node for the detail pane, a runs endpoint for aggregate
counts.

## Workaround until then

There is **no** satisfying CLI-side workaround, and that gap is the point of this
entry:

- **Staleness as a proxy.** The CLI can keep deriving `Last Scan` from
  `ohai_time` and flag nodes past a threshold as "stale." This catches offline
  and badly-broken nodes but cannot distinguish a failed run from a successful
  one, and cannot see a run that crashed mid-flight.
- **Report-handler convention (out of scope for the CLI).** A fleet that installs
  a status-writing handler could expose `node['last_run_status']` in `normal`
  attributes, which the CLI *could* opportunistically read — but the CLI must not
  assume it exists, so this can only ever be a best-effort enhancement, not the
  feature.
- **Chef Automate (separate product).** Fleets running Automate already have this
  data in a different API; pointing the CLI at Automate is a much larger
  integration and still leaves Automate-less fleets without an answer.

The underlying ask is for the core server to record what every run already
computes, so run status is a property of the node — readable in one call, with no
handler and no second product required.
