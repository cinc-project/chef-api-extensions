# 0006 — List the organizations the caller can see, not just all-or-403

- **Status:** Proposed. erchef's behavior is validated from source; whether
  cinc-server-ng gates `GET /organizations` the same way has not been checked
  (its handler has no in-handler check, so any gate lives in middleware).
- **Affects:** Cinc/Chef Server (`GET /organizations`), `cinc-api`
  (`OrgsService.List`)
- **CLI surface that wants it:** `cinc org list`; any org picker in setup or
  the explorer

## Summary

`GET /organizations` answers 403 to anyone without read on the global
organizations container, which in practice means pivotal. Ordinary users
asking "which orgs can I use?" have to know about a different endpoint
(`/users/{name}/organizations`), and clients have no way to ask at all. We'd
like one call that answers "the orgs visible to me" for any caller.

## Where it bites

`cinc org list` tries the real endpoint, catches the 403, and falls back:

```go
// apps/cinc/cmd/org.go:74
names, err := listNames(cmd.Context(), c.Orgs.List)
if errors.Is(err, cinc.ErrForbidden) {
	names, err = fetchMemberOrgNames(cmd, c, err)
}
```

```go
// apps/cinc/cmd/org.go:92
func fetchMemberOrgNames(cmd *cobra.Command, c *cinc.Client, forbidden error) ([]string, error) {
	name := c.ClientName()
	orgs, _, err := c.Associations.ListUserOrgs(cmd.Context(), name)
	if err != nil {
		return nil, forbidden
	}
	...
	fmt.Fprintf(cmd.ErrOrStderr(),
		"You can't list every organization on this server, so here are the ones %q belongs to.\n", name)
```

## Why it matters

1. **Two requests and a guess for the common case.** Most callers are not
   pivotal, so most `org list` runs spend a guaranteed 403 first.
2. **The fallback guesses who is signing.** It assumes the signing name is a
   user. A client-signed request (a node's key, a CI validator) has no
   `/users/{name}/organizations`, so it gets the original 403 even though a
   client belongs to exactly one org the server could name.
3. **Different shapes for one question.** `/organizations` returns
   `{name: url}`; `/users/{name}/organizations` returns
   `[{organization: {name, full_name, guid}}]`. Every client normalizes both.

## What the server does today

erchef authorizes `GET /organizations` as `{container, organization}`
(`oc_chef_wm/src/oc_chef_wm_organizations.erl:81`): read on the global
organizations container. There is no filtering by membership; it's the whole
list or a 403.

## Proposed improvements

1. **`GET /organizations?scope=member`.** Returns the orgs the caller is
   associated with, in the same `{name: url}` shape as the full list: a user's
   associations, or a client's own org. Never 403s for an authenticated
   caller. Opt-in, so existing pivotal tooling that expects the full list (and
   a 403 otherwise) is unchanged. `cinc-api` adds
   `OrgsService.ListMine(ctx)` (or a `WithScope` option on `List`).

2. **Let the full list degrade instead of failing.** Without the parameter,
   a caller who can't read the container gets the member list plus a header
   such as `X-Ops-Scope: member` saying it's partial. Simpler for clients,
   but it changes what an existing call returns, so it's ranked second.

3. **At minimum, a useful 403 body.** Point at the alternative
   (`"use /users/{name}/organizations"`) so other clients find it without
   reading source.

Option 1 removes the 403-then-retry, covers clients, and breaks nothing.

## Workaround until then

What `cinc org list` does today: try the full list, fall back to the user's
associations on 403, and say on stderr that the list is partial. There is no
workaround for client-signed callers.
