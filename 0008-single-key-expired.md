# 0008 — Report `expired` on a single key, not only in the key list

- **Status:** Proposed. Validated against erchef: the single-key body omits the
  field today (`oc_erchef/apps/chef_objects/src/chef_key.erl:122-129`,
  `ejson_from_key`); only the key list carries it (`chef_key.erl:121`).
- **Affects:** Cinc/Chef Server (`chef_key:ejson_from_key`), `cinc-api`
  (`Key.Expired`), cinc-server-ng
- **CLI surface that wants it:** `cinc client key show` / `cinc user key show`

## Summary

The server already decides whether a key has expired: it computes `expired` for
every entry of `GET …/keys`, and it refuses to authenticate with an expired key.
But `GET …/keys/NAME`, the call a client makes when it wants to know about one
key, does not say. We'd like the single-key body (`GET`, and the bodies of `PUT`
and `DELETE` on the same URL) to carry the same `expired` boolean the list does.

## Where it bites

`cinc-api` documents the gap on the model itself:

```go
// github.com/tas50/cinc-api keys.go:26-27
	// Expired is populated by list responses.
	Expired bool `json:"expired,omitempty"`
```

So a `Key` from `KeyScope.Get` always has `Expired == false`, whether or not the
key has expired. The CLI's `key show` prints exactly that object:

```go
// cinc-cli apps/cinc/cmd/keys.go:102-106
key, _, err := owner.scope(c, args[0]).Get(cmd.Context(), args[1])
...
return printer.New(cmd.OutOrStdout(), format).Value(key)
```

and `key list` renders names only (`keys.go:71-80`), so neither command shows an
operator that a key is dead.

## Why it matters

1. **The obvious call gives the wrong answer.** A caller reading `Key.Expired`
   after `Get` sees `false` for an expired key. The zero value is
   indistinguishable from "not expired".
2. **The workaround puts the server's decision on the client's clock.** A client
   can parse `expiration_date` (`"infinity"` or an ISO-8601 timestamp) and
   compare it with its own time, but expiry is enforced against the server's
   clock. Near the boundary, or with clock skew, the client and the server
   disagree about whether the key still authenticates.
3. **Or it costs a second round trip.** The other option is fetching the whole
   key list to read one boolean, which is what the list already computes.

## Proposed improvements

1. **Add `expired` to `chef_key:ejson_from_key`.** Return
   `{"name", "public_key", "expiration_date", "expired"}` from
   `GET/PUT/DELETE …/keys/NAME`, computed exactly as the list computes it. The
   change is additive, so existing clients ignore it. `cinc-api` then drops the
   "populated by list responses" caveat and `key show` displays it as-is.

2. **cinc-server-ng first, behind the proposal.** cinc-server-ng returned
   `"expired": false` on the synthesized default key;
   cinc-project/cinc-server-ng#267 (fixing #266) removes it for parity with
   erchef. If this
   proposal is accepted, cinc-server-ng can return a computed `expired` on every
   single-key body (stored and synthesized alike), recorded as a deliberate
   extension in its differential baseline rather than as debt.

Option 1 is preferred: the server owns the expiry decision, so the server
should report it wherever a key is returned.

## Workaround until then

Compute it client-side in `cinc-api`: after `Get`, set `Expired` from
`ExpirationDate` (`"infinity"` never expires; otherwise parse RFC 3339 and
compare with now). This is correct only to the extent the client's clock
matches the server's. Alternatively, call `List` and read the matching entry's
`expired`, at the cost of a second request.
