uses the bw client beacuse bw internally uses encryption sequence and post calls for admin.


### 429 Bad Requests: 

Three separate token/session caches, all written to plain files on the persistent `/data` volume (so they survive between runs/restarts of the container):

#### Where they're stored

| Token                                                       | File (default path)                               | What's actually in the file                                                                                                                                                                   |
| ----------------------------------------------------------- | ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bw` CLI login timestamp                                    | `/data/bw-last-login` (`BW_LOGIN_STAMP`)          | Just a Unix timestamp — _not_ the actual credential. The real `bw` session state lives wherever the Bitwarden CLI itself keeps it internally; this file only tracks "when did we last log in" |
| API bearer token (for raw `curl` calls to invite/reconcile) | `/data/access-token-cache` (`ACCESS_TOKEN_CACHE`) | Plain text, tab-separated: `<expiry_unix_timestamp>\t<access_token>`                                                                                                                          |
| Admin panel session                                         | `/data/admin-session` (`ADMIN_COOKIE_JAR`)        | A curl cookie jar file — the raw session cookie from `/admin/`                                                                                                                                |

The unlocked vault session (`BW_SESSION`) itself is **not written to disk** — it only lives in an environment variable for the current run, and is re-derived every pass by calling `bw unlock` again with the password.

#### Refresh intervals

| Token                                     | Interval / trigger                                                                                                                                                                                                                                                                                                                                   |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bw` CLI login (`bw login --apikey`)      | Re-run only if **≥270 seconds** (4.5 min) have passed since the timestamp in `bw-last-login`. Server-side API-key sessions (`BW_EXPIRATION`) last 300s (5 min); 270s leaves a 30s safety margin. Also force-triggered immediately, regardless of elapsed time, if a `bw sync` fails unexpectedly (e.g. server restarted)                             |
| `bw unlock` (vault unlock → `BW_SESSION`) | Every single pass, unconditionally — this one isn't cached at all, since it's cheap and doesn't share the same rate limit as login                                                                                                                                                                                                                   |
| API bearer token                          | Cached until the expiry it was issued with (server sends `expires_in`, defaults to 300s), minus a 30s margin. Checked lazily — only fetched/refreshed when enrollment or reconciliation actually needs it, not on a fixed clock                                                                                                                      |
| Admin cookie session                      | No fixed interval — it's tried optimistically on every pass; only if a request with the existing cookie fails does it re-login via `ADMIN_TOKEN` and get a fresh cookie. Server-side these last 20 min (`admin_session_lifetime`) by default, and logins are capped at 3 per 5 minutes, which is exactly why it avoids proactively re-authenticating |

**In short:** everything is refreshed as late as possible — right before it would actually expire, or reactively when a request fails — because Vaultwarden rate-limits logins fairly tightly, and refreshing "just to be safe" on every pass would burn through that budget and start causing `429 Too Many Requests` errors.


### Order in which bw commands are executed :
Here's the exact sequence, in the order the script issues `bw` commands during one pass:

## 1. Login (conditional — only if session is stale)

Checked first: `(now - last_login) >= 270s`?

- If yes:
    1. `bw logout`
    2. `bw config server "$VAULTWARDEN_URL"`
    3. `bw login --apikey`
    4. (timestamp written to `bw-last-login`)
- If no: skipped entirely, reuses the existing session.

## 2. Unlock (always runs, every pass)

5. `bw unlock --passwordenv BW_PASSWORD --raw` → sets `BW_SESSION`

## 3. Sync (always runs, every pass)

6. `bw sync`
    - **If this fails** (e.g. session actually dead despite the timestamp check), it force-retries the whole login chain: 6a. `bw logout` → `bw config server` → `bw login --apikey` (unconditional this time) 6b. `bw unlock` again 6c. `bw sync` again — if this _still_ fails, the script exits with an error.

## 4. Read members

7. `bw list org-members --organizationid "$ORG_ID"`

## 5. Enrollment side-effect (conditional)

If `ADMIN_TOKEN` is set and new users got invited (via plain `curl`, not `bw`) and `enrolled > 0`: 8. `bw sync` (again — to pull in the members just invited) 9. `bw list org-members --organizationid "$ORG_ID"` (again — refreshed list)

_(Note: the invite and reconcile steps themselves use raw `curl` against the REST API, not `bw` — they don't appear in this call order.)_

## 6. Confirm (always runs last, once per pending member)

For every member currently at status `1` (Accepted): 10. `bw confirm org-member "$member_id" --organizationid "$ORG_ID"` — repeated once per pending member, in sequence.

## Summary — best case, no failures, nobody newly enrolled

```
bw unlock
bw sync
bw list org-members
bw confirm org-member   ← repeated per pending member
```

## Summary — worst case, stale session + new enrollments happened

```
bw logout
bw config server
bw login --apikey
bw unlock
bw sync
bw list org-members
(curl: invite new users)
bw sync
bw list org-members
bw confirm org-member   ← repeated per pending member
```

The guiding principle behind the ordering: **never confirm against a stale member list.** `list org-members` is always re-fetched immediately before it's used, and re-fetched _again_ after enrollment adds anyone new, so the final confirm loop always sees the most current membership state.