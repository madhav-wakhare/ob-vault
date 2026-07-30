
# 

  

A small sidecar that removes the manual admin clicks from onboarding a new Vaultwarden org member —

confirming them, inviting them if they registered via SSO but were never added to the org, and

granting them the vault access they need. It runs against a plain, unmodified

[`vaultwarden/server`](https://hub.docker.com/r/vaultwarden/server) image; no server fork required.

  

## The problem it solves

  

Vaultwarden organization membership has four states: `Invited` → `Accepted` → `Confirmed` (or

`Revoked`). Getting someone from nothing to a usable vault normally takes an admin clicking through

the web vault UI at least twice — once to invite, once to confirm — plus a third click to assign

collections/groups if they need something. For a team onboarding through SSO, none of that should

require a human.

  

This bot automates all three steps, running as a loop against a dedicated org-owner service account:

  

| Step | What it does | Why a human doesn't have to |

|---|---|---|

| **Enrol** | Invites any registered user who isn't in the org yet | See below — the one step that has no CLI equivalent |

| **Reconcile** | Grants any existing member the configured groups/collections they're missing | Access is normally set once, at invite time, by whoever clicks "Invite" |

| **Confirm** | Confirms any member sitting at `Accepted` | The one step *no* automation can skip server-side — see below |

  

## Why confirming can't be done by the server itself

  

Confirming a member writes the organization's symmetric key, encrypted to that member's public key.

The server never holds the org key in usable form — only clients that already have it (an existing

confirmed member) can wrap it for someone new. That's Vaultwarden/Bitwarden's zero-knowledge design:

the server stores ciphertext it can't read, by construction. So confirmation has to be performed by a

real client — this container runs the Bitwarden CLI (`bw`) as one dedicated org-owner account to do

exactly that, and nothing else.

  

**This container can decrypt the organization's vault.** That's unavoidable for any auto-confirm

scheme — the tradeoff is keeping that capability in one small service you control, rather than baking

it into the server. Treat its environment as secret material, don't expose any ports (it only makes

outbound calls), and keep it on the same host as Vaultwarden.

  

## How each pass works

  

Every `INTERVAL_SECONDS` (default 60), [`confirm-members.sh`](confirm-members.sh) runs once:

  

```

bw login/unlock (session) ─┐

├─► bw sync ─► bw list org-members

ADMIN_TOKEN set? ───────────┘ │

│ │

[enrol] [reconcile]

GET /admin/users (who's registered) GET .../users?includeCollections&includeGroups

→ diff against org members → diff each member's current access vs configured

→ POST .../users/invite for the gap → PUT .../users/<id> to grant the gap (additive only)

│ │

└──────────────┬─────────────────────────┘

▼

bw confirm org-member

for everyone at status `Accepted`

```

  

### Confirm (always runs)

  

`bw sync`, then `bw list org-members` filtered to `status == 1` (`Accepted`), then

`bw confirm org-member` for each. Already-confirmed members and anyone still at `Invited` (`status ==

0`, mid-signup — there's nothing to confirm yet) are left alone. A failed confirm is logged and

retried next pass.

  

### Enrol (opt-in via `ADMIN_TOKEN`)

  

`bw` has no command to invite a member — I checked: `bw create` only supports

`item, attachment, folder, org-collection`, and there's no `bw invite` at all. So this step calls the

server's HTTP API directly instead of going through the CLI:

  

1. `GET /admin/users` — every registered account, admin-console JWT auth (`ADMIN_TOKEN` posted once to

`/admin/`, then the session cookie is cached at `/data/admin-session` and reused). Vaultwarden

rate-limits admin logins to a burst of 3 per 300s, so this only re-authenticates once the cached

session actually expires (20 minutes by default).

2. Diff against the org's current member list (case-insensitive email match).

3. `POST /api/organizations/<org_id>/users/invite` for anyone missing, **but only accounts at

`_status == 0`** (`UserStatus::Enabled` — master password already set). With mail disabled,

Vaultwarden auto-accepts an invite straight to `Accepted` only for a user that already has a

password; inviting someone mid-signup (`_status == 1`) would strand them at `Invited` forever, with

no invitation email to click.

  

`ADMIN_TOKEN` is a wide grant — full server administration — so this whole step is opt-in. Leave it

unset and the bot still confirms anyone a manual invite (or anything else) has already moved to

`Accepted`; it just won't discover new members on its own.

  

### Reconcile (opt-in via `ORG_GROUP_IDS` / `ORG_COLLECTION_IDS`)

  

An invite only grants access *at the moment it's sent*. A member invited before you set

`ORG_GROUP_IDS`/`ORG_COLLECTION_IDS` — or before you changed the value — keeps whatever they got at

invite time forever, unless something goes back and fixes it. This step is that fix, and it runs every

pass regardless of `ADMIN_TOKEN` (it only touches org membership, not the server-wide user list):

  

1. `GET /api/organizations/<org_id>/users?includeCollections=true&includeGroups=true` — every member's

*current* groups/collections.

2. For each member (skipping Owners/Admins and anyone already at `accessAll` — they already see

everything, and there's no reason to touch an Owner's own access), compute `configured − existing`.

3. `PUT /api/organizations/<org_id>/users/<member_id>` with the **union** of existing + missing.

  

The `PUT` here replaces a member's *entire* collection/group list rather than adding to it — that's

the API's behavior, not a choice this script made. So the union in step 3 is load-bearing: reading

current access first and only ever adding to it means pre-existing access unrelated to these two

variables is preserved, never dropped, even though the underlying call is a full replace.

  

Prefer `ORG_GROUP_IDS` over `ORG_COLLECTION_IDS` where possible — group membership keeps granting

access to collections created *later*; a fixed collection list does not.

  

## Environment variables

  

| Variable | Required | Purpose |

|---|---|---|

| `VAULTWARDEN_URL` | yes | Public URL of your Vaultwarden, no trailing slash |

| `ORG_ID` | yes | UUID of the organization to manage |

| `BW_CLIENTID` | yes | API key `client_id` (`user.<uuid>`) of the dedicated org-owner account |

| `BW_CLIENTSECRET` | yes | API key `client_secret` for that account |

| `BW_PASSWORD` | yes | That account's master password (`bw unlock` needs it to derive the org key) |

| `INTERVAL_SECONDS` | no (default `60`) | Delay between passes |

| `ADMIN_TOKEN` | no | Enables enrolment — same value as the server's own `ADMIN_TOKEN` |

| `ORG_GROUP_IDS` | no | Comma-separated group UUIDs granted at invite time and reconciled every pass |

| `ORG_COLLECTION_IDS` | no | Comma-separated collection UUIDs, same as above — prefer groups |

| `ADMIN_COOKIE_JAR` | no (default `/data/admin-session`) | Where the cached admin session is stored |

  

Any of the five required variables being unset makes the script log `not configured yet, nothing to

do` and exit `0` — it idles quietly rather than crash-looping, which matters if the compose file lands

before the secrets are set (e.g. in Coolify). Enrolment and reconciliation are each independently

gated on their own variables even once the required five are set — see the two sections above.

  

## Setup

  

### 1. Create the service account

  

Sign in to Vaultwarden as yourself, invite e.g. `vaultwarden-bot@one2n.in`, confirm it, and make it an

**Owner** of the org (Admin is enough unless you also need it to confirm other Admins/Owners). Give it

a long random master password — this becomes `BW_PASSWORD`.

  

`SSO_ONLY=true` does not block the bot: it only rejects the `password` grant (`src/api/identity.rs`),

while `bw login --apikey` and this bot's own token calls use `client_credentials`. It does mean the

account needs a master password some other way — create it before turning `SSO_ONLY` on, let it sign

in through the IdP once and set a master password, or flip `SSO_ONLY=false` briefly.

  

Do not enable 2FA on it — `bw login --apikey` skips 2FA, but keep that in mind when auditing.

  

### 2. Find each configuration value

  

| Variable | Where it comes from |

|---|---|

| `VAULTWARDEN_URL` | Your instance's own public URL, no trailing slash — nothing to look up |

| `BW_CLIENTID` / `BW_CLIENTSECRET` | Sign in as the service account → **Settings → Security → Keys → View API key**. Gives `client_id` (`user.<uuid>`) and `client_secret` |

| `ORG_ID` | See below |

| `ADMIN_TOKEN` | Not looked up — this is a value **you set** on the Vaultwarden server itself (its own `ADMIN_TOKEN` env var). Reuse that same value here to enable enrolment |

| `ORG_GROUP_IDS` / `ORG_COLLECTION_IDS` | See below |

  

**`ORG_ID`** — either:

- Web Vault → **Organizations** → open the org → the URL bar shows `.../organizations/<org_id>/vault`, or

- `bw list organizations` (once logged in as any member) — returns `id` + `name` in plaintext.

Unlike collection/group names, an org's `name` is **not** client-side encrypted (confirmed against

`Organization` in `src/db/models/organization.rs` — plain `String`, no encryption handling — and

it's exactly why the admin panel can show the org name as a plaintext badge).

  

**`ORG_GROUP_IDS` / `ORG_COLLECTION_IDS`** — get a bearer token once, then list either:

```bash

TOKEN=$(curl -s "$VAULTWARDEN_URL/identity/connect/token" \

-d grant_type=client_credentials -d scope=api \

-d deviceType=8 -d deviceName=cli -d deviceIdentifier=cli \

--data-urlencode "client_id=$BW_CLIENTID" \

--data-urlencode "client_secret=$BW_CLIENTSECRET" | jq -r .access_token)

  

# Groups (Organizations → Groups in the web vault)

curl -s "$VAULTWARDEN_URL/api/organizations/$ORG_ID/groups" \

-H "Authorization: Bearer $TOKEN" | jq -r '.data[] | .id'

  

# Collections (Organizations → Collections)

curl -s "$VAULTWARDEN_URL/api/organizations/$ORG_ID/collections" \

-H "Authorization: Bearer $TOKEN" | jq -r '.data[].id'

```

Unlike `Organization.name`, both `name` fields here **are** ciphertext (client-side encrypted, same

reason confirming needs a real client) — the `id` is all you need for the env vars, but if you have

several and need to tell them apart by name, use `bw list org-collections --organizationid $ORG_ID`

instead, which decrypts client-side and shows the real name. There's no CLI equivalent for groups —

use the Web Vault's Groups page if you need to match a group by name rather than just grab its `id`.

  

Multiple IDs are comma-separated, e.g. `ORG_GROUP_IDS=id1,id2` (whitespace around each is trimmed, so

`id1, id2` also works).

  

### 3. Set the environment and deploy

  

Fill in the table from the [Environment variables](#environment-variables) section above and deploy.

  

In Coolify: new resource → Docker Compose (or point a "Dockerfile" resource at this repo), paste the

variables, deploy. No domain or port needed — it makes outbound calls only.

  

## Building and publishing the image

  

CI builds `linux/amd64` and pushes to GHCR on every push to `main`

([`.github/workflows/image.yml`](.github/workflows/image.yml)).

  

To publish to Docker Hub (or anywhere else) by hand, use [`build.sh`](build.sh):

  

```bash

TAG=v2 ./build.sh push # build for linux/amd64 and push

IMAGE=docker.io/you/confirm-bot TAG=v2 ./build.sh push # different registry/namespace

TAG=test ./build.sh load # build and load locally instead, for a smoke test

```

  

`--platform linux/amd64` is required regardless of the build machine's architecture — Coolify's host

is amd64, and on Apple Silicon this cross-builds under QEMU emulation rather than matching the host.

`push` finishes by running `docker buildx imagetools inspect` so you can confirm what actually landed

in the registry is really `amd64`. Bump the tag (`v2`, `v3`, …) rather than reusing one:

`docker-compose-coolify.yml` pins the tag deliberately, so a redeploy is explicit rather than silently

picking up whatever `:latest` happens to point at.

  

## Running once, locally

  

```bash

docker compose run --rm confirm-bot /usr/local/bin/confirm-members.sh

```

  

## Files

  

| File | Purpose |

|---|---|

| [`confirm-members.sh`](confirm-members.sh) | The actual logic: enrol, reconcile, confirm |

| [`loop.sh`](loop.sh) | Runs `confirm-members.sh` every `INTERVAL_SECONDS`, forever, with graceful shutdown on `SIGTERM`/`SIGINT` |

| [`Dockerfile`](Dockerfile) | `node:22-slim` + `@bitwarden/cli` + `curl`/`jq` |

| [`build.sh`](build.sh) | Cross-builds for `linux/amd64` and pushes/loads |

| [`docker-compose.yml`](docker-compose.yml) | Standalone compose file for this service alone (`build: .`) |

| [`.github/workflows/image.yml`](.github/workflows/image.yml) | CI: builds and pushes to GHCR on push to `main` |