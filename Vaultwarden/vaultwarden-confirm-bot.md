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


----------------------------
---
# Vaultwarden Confirm Bot — Notes

## 1. Org membership states

Four states per member, per org (not the same as account status — see below):

|Status|Value|Meaning|
|---|---|---|
|Invited|0|Invited to org, no action taken yet|
|Accepted|1|User has a real account and accepted the invite (or it was auto-accepted) — but not yet usable|
|Confirmed|2|Org key wrapped with the member's public key — now usable|
|Revoked|-1|Access removed|

**Why "Accepted" isn't the finish line:** Bitwarden/Vaultwarden is zero-knowledge — the server never holds the org's symmetric key in usable form, only ciphertext. Only an existing confirmed member (a real client with the key) can wrap that key for a new member's public key. That's the "confirm" step — the one thing no server-side automation can do. This bot runs the `bw` CLI as a dedicated org-owner account purely to perform that step (`bw confirm org-member`).

**Reference:** [confirm-members.sh:5-8](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L5-L8), [confirm-members.sh:337](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L337), [README.md:24-36](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/README.md#L24-L36)

---

## 2. Account status vs. org membership status (two different things)

- **`User.status`** (account-level): `Enabled` (0) = finished signup, master password set. `Invited` (1) = account row exists but signup isn't finished (e.g. signed in via SSO but never set a master password).
- **Org membership status** (table above): about being a member of _this org_, separate from whether the account itself is finished.

A user can show an SSO identity + "Verified" email in `/admin/users` and still be stuck at `Invited` (account-level) if they never completed the master-password step after SSO login. **The bot skips these on purpose** — inviting a mid-signup account would strand them (no invite email since mail is disabled, and they can't act on an org invite from an unfinished account).

**Reference:** [confirm-members.sh:185-196](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L185-L196)

---

## 3. `auto_enrol` — why it's gated on `ADMIN_TOKEN`

```bash
auto_enrol=1
[[ -n "${ADMIN_TOKEN:-}" ]] || auto_enrol=0
```

The script does two independent jobs with two different credentials:

- **Confirm** (always runs) — needs only the org-owner's `bw` session.
- **Enrol** (opt-in) — needs `GET /admin/users`, a full server-admin endpoint (`ADMIN_TOKEN`), far broader than an org-owner login.

Without `ADMIN_TOKEN`, the bot degrades gracefully: it still confirms anyone already `Accepted` (via manual invite or a server that self-enrols), it just can't discover unenrolled registered users on its own.

**Reference:** [confirm-members.sh:49-53](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L49-L53), [README.md:79-98](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/README.md#L79-L98)

---

## 4. Full SSO user lifecycle

1. User authenticates via IdP → Vaultwarden creates/links a local account. They are **not yet an org member**.
2. Account may sit at `User.status = Invited` if they haven't finished setting a master password (SSO login alone doesn't complete Bitwarden's encryption setup).
3. Once `Enabled`, bot's **enrol** step invites them into the org (`type: 2`, plus configured groups/collections).
4. With mail disabled, Vaultwarden **auto-accepts** the invite immediately (no email round-trip) since the account is already `Enabled` → membership status `Invited → Accepted`.
5. Bot's **confirm** step (`bw confirm org-member`) wraps the org key for them → `Accepted → Confirmed`. Now usable.
6. Bot's **reconcile** step keeps their groups/collections in sync with current config on every subsequent pass.

**Reference:** [confirm-members.sh:10-16](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L10-L16), [README.md:8-22](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/README.md#L8-L22)

---

## 5. Why reconciliation is a separate step from enrol/confirm

`invite_member()` only assigns groups/collections **at invite time**, using whatever `ORG_GROUP_IDS`/`ORG_COLLECTION_IDS` are set _then_. Neither enrolment nor confirm ever revisits that. If you change the config later (e.g. add a new collection), existing members keep whatever they got originally — forever, unless something goes back and fixes it.

`reconcile_member_access` runs every pass on every existing member (skipping Owners/Admins/`accessAll`), diffs current vs. configured access, and grants the **union** (never a smaller set — the edit-member API replaces the whole list, so reading-then-adding is what prevents accidentally revoking existing access).

**Reference:** [confirm-members.sh:14-16](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L14-L16), [confirm-members.sh:214-274](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L214-L274), [README.md:100-119](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/README.md#L100-L119)

---

## 6. Credential refresh cadence — what, how often, why

|Credential|Refresh cadence|Why|
|---|---|---|
|Admin session cookie|Reactive — only re-logs in when the cached cookie fails (~every `admin_session_lifetime`, default 20 min)|Admin login is rate-limited to burst of 3 per 300s; logging in every pass would 429 quickly|
|API access token (bearer, invite/reconcile calls)|Every ~270s (cached with expiry, server token TTL is 300s)|`client_credentials` grant never returns a refresh token — full relogin every time, shares the server's general login rate limiter|
|`bw` CLI login (org-owner)|Every ~270s, tracked via `/data/bw-last-login`; force-retries once if `bw sync` fails|Same rate limiter as above; logging in every pass would run ~2x the sustainable rate|
|`bw unlock` (vault session)|**Every single pass**, no caching|Purely local crypto (decrypts already-synced vault) — no server call, no rate limit concern|

**Reference:** [confirm-members.sh:55-58](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L55-L58), [confirm-members.sh:103-107](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L103-L107), [confirm-members.sh:276-297](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L276-L297), [README.md:58-77](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/README.md#L58-L77)

---

## 7. Why `INTERVAL_SECONDS` defaults to 60

This is the sleep between passes in [loop.sh:6](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/loop.sh#L6), not something derived from the credentials themselves.

60s lines up with the **slowest-recovering constraint in the system**: the server's general login rate limiter refills at only **1 per 60s** (burst of 10). Even cached logins eventually need to refresh (~every 4.5 min); polling faster than 60s wouldn't make those any more reliable — it would just add extra non-login calls every pass (`bw sync`, `bw list org-members`, admin/reconcile GETs) for state that doesn't actually change faster than once a minute in practice (a person finishing signup, an admin editing config).

Net effect: worst case, a new SSO user waits ~60s from finishing signup to being confirmed — often less, since enrol→accept→confirm can complete within a single pass.

**Reference:** [loop.sh:6](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/loop.sh#L6), [README.md:68](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/README.md#L68), [confirm-members.sh:323-326](vscode-webview://01m0pm37ec0pr5ao6ifr3mc2d4hchfhnj6vd3iscp8ecf4ipho55/vaultwarden-confirm-bot/confirm-members.sh#L323-L326)