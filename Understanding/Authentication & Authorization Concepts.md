
## JWT & Bearer token

The core difference is that a **Bearer Token is a transport mechanism (how a token is sent)**, whereas a **JWT (JSON Web Token) is a data format (what the token contains)**. [[1](https://www.descope.com/blog/post/jwt-vs-bearer-token)]

They are not competing technologies. Instead, they work together: **a JWT is frequently transmitted as a Bearer Token**. Think of a Bearer Token as an envelope addressed to whoever holds it, and a JWT as the specific letter format inside that envelope.


## OpenID Connect and Oauth2

Arcticle : https://supertokens.com/blog/openid-connect-vs-oauth2

OAuth 2.0 is an **authorization** framework that decides _what_ resources an app can access, while OpenID Connect (OIDC) is an **authentication** layer built on top of OAuth 2.0 that figures out _who_ the user is. OAuth gives a key to access data; OIDC adds an ID badge showing user identity.

Core Differences
- **Primary Purpose**: OAuth 2.0 handles access delegation; OIDC handles user login and identity.
- **The Question Answered**: OAuth asks, "Can this app access this data?"; OIDC asks, "Who is signed in?"
- **Tokens Issued**: OAuth 2.0 issues an **Access Token**; OIDC issues an **ID Token** (usually a JWT) alongside the Access Token.
- **Endpoints**: OAuth uses resource/API endpoints; OIDC standardizes scopes (`openid`, `profile`, `email`) and adds a `/userinfo` endpoint. [[1](https://developer.okta.com/docs/concepts/oauth-openid/), [2](https://medium.com/@QuarkAndCode/oauth-2-0-vs-openid-connect-oidc-best-practices-security-in-2025-0c82f071a9a9)

When to Use Which

- **Use OAuth 2.0 Alone**: When you only need to grant an external service or microservice safe, scoped access to APIs or backend data without managing a human login session. [[1](https://aembit.io/blog/oauth-vs-oidc-difference-when-to-use/), [2](https://www.geeksforgeeks.org/websites-apps/oauth-vs-openid-connect/)]

- **Use OIDC (with OAuth 2.0)**: Whenever you need a "Log in with Google" or Single Sign-On (SSO) feature for users, letting your app verify their identity and fetch profile details.

