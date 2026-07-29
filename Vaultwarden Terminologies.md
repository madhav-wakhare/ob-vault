
- **Organizations**: A shared corporate or family workspace that manages members and overall data ownership.
- **Collections**: Shared folders within an organization used to group passwords and assign specific user access.
- **Groups**: are collections of users used to assign permissions to many people at once.
---
- **Groups** are bundles of **people**.
- **Collections** are bundles of **passwords**.
- **Organizations** house both of them under one roof. 
---
# SSO Auto-Enrollment

Additions to upstream Vaultwarden that let SSO users receive shared credentials with **no administrator action**. Everything here is opt-in: leave the new settings unset and the server behaves exactly like upstream.

## The problem

Out of the box, getting a new person access to shared credentials takes four manual steps: invite them to the organization, wait for them to accept, confirm them, then assign collections. For an SSO-backed team that is the only part of onboarding that is not automatic — the identity provider already decided this person belongs.

## What this does

A user signs in through SSO for the first time and, seconds later, has the shared credentials in their vault. Nobody clicks anything.

Behind that, four things happen automatically:

1. **Joined** — the user becomes a member of a nominated organization on their first SSO sign-in.
2. **Grouped** — they are added to a nominated group, which is what carries collection access.
3. **Accepted** — the membership is accepted for them, with the organization's join policies still enforced.
4. **Confirmed** — the organization key is wrapped for their account, so shared items actually decrypt.

No invitation email is sent, and the user never sees an invitation prompt.

## Settings

| Setting | Required | What it does |
| --- | --- | --- |
| `SSO_DEFAULT_ORGANIZATION_UUID` | to enable anything | Organization that first-time SSO users join |
| `SSO_DEFAULT_GROUP_UUID` | for shared access | Group they are added to; grant this group your shared collections |
| `SSO_DEFAULT_ORGANIZATION_KEY` | for zero-touch | Organization key, base64. Without it members stop at *Accepted* and still need confirming |

All three are editable from the admin panel. The organization key is masked there. The two UUID settings accept any UUID spelling (uppercase, braces, no hyphens) and are normalised.

Both `SSO_DEFAULT_GROUP_UUID` and `SSO_DEFAULT_ORGANIZATION_KEY` require the organization UUID to be set, and the server refuses to start on a malformed value rather than failing later during a login.

## Setup

1. Create the organization that will hold the shared credentials, and note its UUID.
2. Create a group in it, and give that group access to the collections you want shared.
3. Put the shared credentials in those collections.
4. Set `SSO_DEFAULT_ORGANIZATION_UUID` and `SSO_DEFAULT_GROUP_UUID`.
5. Set `SSO_DEFAULT_ORGANIZATION_KEY` to that organization's key (see *Obtaining the organization key*).
6. Sign in with a throwaway SSO account and confirm the shared items appear.

Access is changed afterwards by editing the group's collections in the UI — no config change and no restart. Point the group at a different collection and every existing member follows.

## Behaviour details

**Existing accounts are handled too.** Someone who already has a Vaultwarden password and then uses SSO for the first time is enrolled in a single pass. Brand-new accounts are enrolled in two passes, because their encryption keys do not exist until they choose a master password — this is invisible to the user.

**Existing memberships are never disturbed.** If the user is already in the organization in any state, their membership is left exactly as it is. Revoked members are not resurrected, and confirmed members are not re-processed.

**Policies are still enforced.** Organization join policies — two-step login required, single organization — are evaluated before a member is accepted. If a policy blocks a particular user, that user is left pending and a warning is logged; their sign-in still succeeds. One person's policy conflict cannot lock anyone else out.

**Everything is audited.** Joins, group assignment and confirmation each write to the organization's event log, so automatic enrollment is as traceable as the manual flow.

**Works without SMTP.** No email is required at any point.

## Obtaining the organization key

This is the one manual step, done once at setup.

The organization key never exists in plaintext on the server — normally it only lives inside a logged-in client, wrapped separately for each member. To let the server confirm members on its own, it needs a copy, which means extracting it from an administrator's account once and putting it in the config.

> A helper for this is **not yet written**. Until it exists, `SSO_DEFAULT_ORGANIZATION_KEY` cannot be filled in, and enrollment stops at *Accepted* — steps 1–3 are automatic, confirmation is not.

## Security

Setting `SSO_DEFAULT_ORGANIZATION_KEY` is a deliberate trade of confidentiality for automation, and it is the only way to remove the last manual step. Consequences:

- **The server can read that organization's items.** Everything in it is decryptable by the server.
- **The key sits in plaintext at rest**, in the environment or in `config.json`.
- **A compromise of this host or its database exposes that entire shared vault.**

Scope this to one purpose-built organization holding only the credentials you accept as server-readable. Personal vaults and every other organization are unaffected — they never hand their keys to the server.

Leaving the key unset keeps end-to-end encryption fully intact, at the cost of one confirmation click per user.

## Status

| | |
| --- | --- |
| Formatting | passes |
| Compiler, linter, tests | **not yet run** |
| Confirmation verified against a real client | **no** |

The confirmation step wraps the organization key the way Bitwarden clients do. That has been tested for internal consistency but **not** against an actual Bitwarden client, and this is the first place the server generates such a value rather than storing one a client produced. **Validate with a throwaway SSO account before onboarding real users.** The symptom of getting it wrong is the organization appearing in the vault with items that will not decrypt.

## Known gaps

- No helper yet for extracting the organization key, which blocks the zero-touch path.
- Membership is always the *User* role; there is no setting for enrolling people as managers or admins.
- A single organization and a single group; no mapping from identity-provider groups to different organizations or collections. Provider-driven group mapping would be the better long-term design and would supersede most of this.