# 0005 — Cookbook list with version counts and latest metadata

- **Status:** Proposed. Part 1 is validated against erchef's list handler (the
  count is already in hand when the response is built); part 2 is not yet
  validated for cost.
- **Affects:** Cinc/Chef Server (`GET /organizations/{org}/cookbooks`),
  `cinc-api` (`CookbooksService.List` / `ListVersions` result types)
- **CLI surface that wants it:** the explorer cookbook list and summary pane
  (`cinc explore` → Cookbooks); `cinc cookbook list` if it ever shows counts

## Summary

To show how many versions each cookbook has, the CLI downloads the **entire**
version index (`num_versions=all`) and counts it. To show what a cookbook *is*
(description, maintainer, license), it then fetches the latest version's full
manifest. The server already knows the count and stores the metadata; we'd
like the list endpoint to return both directly.

## Where it bites

The explorer's cookbook list asks for every version only to call `len()`:

```go
// cli/explore/kind_cookbook.go:20
func (cookbookKind) List(ctx context.Context, c *cinc.Client) ([]Row, error) {
	// A bare List returns only each cookbook's latest version on erchef, so
	// ask for all of them to count them.
	index, _, err := c.Cookbooks.ListVersions(ctx, "all")
	...
		Cells: []string{name, fmt.Sprintf("%d", len(entry.Versions))},
```

The summary pane then makes two more calls for one cookbook: the version list
again (to find the latest), and the full manifest of that version just to read
identity fields:

```go
// cli/explore/kind_cookbook.go:46
entry, _, err := c.Cookbooks.GetVersions(ctx, name, "all")
...
cb, _, err := c.Cookbooks.Get(ctx, name, v) // whole manifest: every file entry
```

## Why it matters

1. **Payload grows with history, not with what we show.** An org with 200
   cookbooks averaging 40 versions each returns 8,000 `{url, version}` entries
   so the client can print 200 numbers.
2. **A manifest to read three strings.** The latest manifest carries every
   file's name, path, checksum and URL. The summary reads `description`,
   `maintainer` and `license` from `metadata`.
3. **Richer lists are blocked.** A cookbook list that shows a description
   column needs one manifest fetch per row today, which no client will do.

## What the server does today

erchef's list handler already loads every version before trimming to
`num_versions` (`oc_chef_wm/src/chef_wm_cookbooks.erl:115-118`,
`make_cookbook_list/3` at `:191`). Each entry is built as
`{url, versions}`; the full count is known at that point and discarded.

Per-version metadata lives in `cookbook_versions.metadata`, a compressed
`bytea` column stored `EXTERNAL`
(`schema/baseline/deploy/cookbook_versions.sql`), so returning it means
reading and decompressing one row per cookbook. That cost is why part 2 below
is opt-in.

## Proposed improvements

1. **Add `version_count` and `latest_version` to each list entry.** Always on,
   additive, and free given the handler above:

   ```json
   {
     "apache2": {
       "url": "https://.../cookbooks/apache2",
       "version_count": 41,
       "latest_version": "9.1.3",
       "versions": [ { "version": "9.1.3", "url": "..." } ]
     }
   }
   ```

   Existing clients ignore unknown keys. `cinc-api` adds the two fields to its
   index entry type; the explorer drops `num_versions=all` and counts nothing.

2. **Opt-in latest metadata: `?include=metadata`.** Adds a small, fixed subset
   of the latest version's metadata to each entry (`description`,
   `maintainer`, `license`, `source_url`). Not the whole metadata blob:
   dependencies and attributes stay on the per-version route. Needs a cost
   check on a large org before committing to it.

3. **Same shape on `/cookbook_artifacts`.** Policyfile users browse artifacts
   the same way; if part 1 lands, mirror it there so the two lists match.

Part 1 is the clear win and nearly free to implement. Part 2 is what makes a
descriptive cookbook list possible at all.

## Workaround until then

- Keep `num_versions=all` for counts; there's no cheaper way to count.
- Cache the per-cookbook summary for the explorer session so revisiting a
  cookbook doesn't refetch its manifest.
