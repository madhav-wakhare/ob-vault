### The problem it solves

Vaultwarden supports SSO login (logging in via your company's identity provider — Okta, Azure AD, etc. — instead of typing a password into Vaultwarden directly).

**Before this PR:** when a brand-new user logged in via SSO for the first time, Vaultwarden created their account... and stopped there. They weren't part of any Organization. An admin had to _manually_ go find that person and send them an organization invite before they could see any shared/team vaults, collections, or policies. Also, a related endpoint always returned a fake, meaningless organization ID, so nothing about the "real" org's policies (like password requirements) applied during that first-time signup.

**This PR adds** a `SSO_DEFAULT_ORGANIZATION_UUID` setting. If the admin sets it, every brand-new SSO user is _automatically_ enrolled into that organization the moment they finish setup — no manual invite needed. But it stops short of giving them real access: an org admin still has to "confirm" them afterward before they get collections/groups assigned.

### Vaultwarden's normal SSO flow (unchanged parts)

1. User clicks "log in with SSO" → gets redirected to the identity provider, authenticates there.
2. Comes back to Vaultwarden with proof of identity (`sso::redeem` in the code).
3. Since SSO doesn't hand Vaultwarden an encryption key, the user is sent to a "set your master password" screen (`post_set_password`) — this is what actually initializes their vault encryption.
4. _(Previously, nothing else happened automatically — org membership was 100% manual.)_

### What the PR changes, step by step

- **At step 2** (`invite_user_to_default_organization`, new function in `sso.rs`): if a default org is configured, the new user is silently given an "Invited" membership row in that org — before they've even set a password.
- **At step 3** (`post_set_password` in `accounts.rs`): once they finish setting their password, if they were using the default org's identifier, their membership flips from "Invited" straight to "Accepted" — skipping the usual "click accept on the invite email" step (no email is even sent for this).
- **The org policy check still runs** (`OrgPolicy::check_user_allowed`) so things like a mandatory master-password policy for that org still get enforced.
- **The org admin still has a final gate**: "Accepted" isn't the same as fully active — a human admin still has to confirm the member and assign them collections/groups, exactly like Bitwarden's official SSO-with-domain-verification flow works.
- A smaller fix: the `/organizations/domain/sso/verified` endpoint used to _always_ return a dummy org ID. Now, if a default org is set, it returns the _real_ org ID, so the client correctly ties the signup to that org's policies from the start.

### Analogy

Think of a big company using a security badge (SSO) to let new hires into the building.

**Before:** A new hire badges in on day one, gets a desk and a laptop — but they're on nobody's team. Someone from HR has to remember to manually add them to a team's directory later, or they can't access any shared drives.

**After this PR:** The moment they badge in for the first time, they're automatically put on the "New Hires" roster (Invited) for a pre-designated onboarding team. Once they finish their laptop setup (setting a password), they move from "Invited" to "Accepted" on that roster automatically — no one had to click "yes, let them in" for that first step. But they _still_ don't get real access to the shared drives or tools — their actual manager still has to review them and flip the switch that hands out folder permissions. The badge system just removed the tedious "someone has to remember to add them to a team" step; it didn't remove the manager's final approval.