## Why OAuth 2

> **In short**: OAuth 2 is a **delegated authorization** protocol: it lets an application access an API on behalf of a user, with a limited and revocable access token issued by an authorization server, without ever knowing the user's password. OpenID Connect adds **authentication** on top, with an ID token telling the application who the user is.

Before OAuth, an application that wanted your Gmail contacts asked for your Gmail password: full access, impossible to revoke for that application alone. With OAuth 2 (RFC 6749, 2012), you log in on Google itself, authorize the application to read your contacts, and it receives a limited, short-lived, revocable access token instead of the password.

- **Authorization, not authentication**: an access token says "the bearer may read the contacts", not "this is Alice". OpenID Connect (2014) adds authentication on top, with the ID token (see below).
- Today it's also the standard way to secure one's own APIs, with a central authorization server (Keycloak, Auth0, Entra ID) issuing tokens to the front-ends, mobile apps and services.
- OAuth 2.1 (draft) and the Security BCP (RFC 9700, 2025) consolidate the best practices: PKCE everywhere, implicit and password flows removed.

## Actors and tokens

> **In short**: The resource owner (the user) authorizes a client (the application) to access a resource server (the API); the authorization server authenticates the user and issues the tokens. The access token is for the API, the refresh token gets new access tokens, and the ID token (OIDC) tells the client who the user is.

| Role | Example |
|---|---|
| Resource owner | the user |
| Client | the Angular app, a BFF, a mobile app, a batch |
| Authorization server | Keycloak, Auth0, Okta, Entra ID, Spring Authorization Server |
| Resource server | the API, which validates the access token |

| Token | For | Content / lifetime |
|---|---|---|
| Access token | the API, in `Authorization: Bearer ...` | scopes, audience, expiry; short (5-15 min); often a JWT |
| Refresh token | the authorization server | gets new access tokens without logging in again; long-lived, revocable |
| ID token (OIDC) | the client only, never sent to an API | who logged in (`sub`, `email`, `name`); always a JWT |

- **Confidential client**: runs on a server and can keep a secret (BFF, backend service). **Public client**: code visible to the user, no secret possible (SPA, mobile app), hence PKCE.
- A client is registered beforehand on the authorization server: `client_id`, secret if confidential, allowed `redirect_uri`, allowed scopes.
- **Scopes** are what the user consents to (`orders:read`); the API checks them (`SCOPE_orders:read` in Spring).

## Authorization code flow with PKCE

> **In short**: The client redirects the user to the authorization server, the user logs in there, and the authorization server redirects back with a short-lived code; the client exchanges the code for tokens in a direct call. PKCE binds the code to the client that started the flow: an intercepted code is useless without the secret `code_verifier`. It's the flow for any application with a user.

```text
Browser ──1. redirect /authorize?response_type=code&client_id=shop&redirect_uri=...&scope=openid orders:read
                                 &state=xyz&code_challenge=SHA256(verifier)──────> Authorization server
        2. the user logs in and consents (on the authorization server, never on the client)
        <─3. redirect https://shop.com/callback?code=abc&state=xyz───────────────
Client ──4. POST /token  code=abc + code_verifier (+ client secret if confidential)──>
        <─  access token, refresh token, ID token ──────────────────────────────
Client ──5. GET /api/orders  Authorization: Bearer <access token>──> Resource server
```

- **Why a code and not the token directly**: the redirect goes through the browser (URL, history, logs); the code is short-lived and single-use, and the tokens are obtained through a direct call (back channel).
- **PKCE**: the client generates a random `code_verifier`, sends its hash (`code_challenge`) at step 1 and the verifier itself at step 4. An attacker who steals the code can't exchange it.
- **`state`**: a random value checked at step 3, against CSRF on the callback (and `nonce` in the ID token, against replay).
- `redirect_uri` must match a registered value exactly, otherwise the code could be sent to an attacker's site.

## Grant types

> **In short**: Authorization code + PKCE for any application with a user, client credentials for service-to-service calls without a user, refresh token to renew an access token, device code for devices without a browser. Implicit and password grants are deprecated.

| Grant type | Use | How |
|---|---|---|
| Authorization code + PKCE | SPA, BFF, mobile app: any app with a user | see above |
| Client credentials | service to service, batch: no user | the client sends `client_id` + secret to `/token`, receives an access token |
| Refresh token | renew an expired access token | `POST /token grant_type=refresh_token`; with rotation, a new refresh token each time |
| Device code | TV, CLI, devices without a browser | the device shows a code, the user enters it on their phone |
| Token exchange (RFC 8693) | a service calls another one on behalf of the user | exchanges a token for another one with a different audience |
| ~~Implicit~~ | deprecated | the token came back in the URL: exposed in history and logs |
| ~~Password (ROPC)~~ | deprecated | the user gives their password to the client: the very problem OAuth solves |

## OpenID Connect

> **In short**: OIDC is an identity layer on top of OAuth 2: with the `openid` scope, the authorization server also returns an ID token, a signed JWT telling the client who logged in. It also standardizes the `userinfo` endpoint, the discovery document and the standard scopes (`profile`, `email`).

- **ID token**: `iss` (who issued it), `sub` (the user's unique id), `aud` (the `client_id`: the token is for this client only), `exp`, `nonce`, plus `email`, `name`... The client validates the signature and these claims, then considers the user logged in.
- **Never use the ID token to call an API**: its audience is the client; the API expects an access token.
- **Discovery**: `https://auth.example.com/.well-known/openid-configuration` lists the endpoints (`authorization_endpoint`, `token_endpoint`, `userinfo_endpoint`) and the URL of the public keys (`jwks_uri`). That's why `issuer-uri` is enough to configure Spring Security.
- **`userinfo`**: an endpoint returning the user's profile, called with the access token.

Pitfalls to mention in an interview:

- The API must check the access token's audience (`aud`), not only its signature: otherwise a token issued for another API of the same authorization server is accepted.
- Where an SPA keeps its tokens is a security choice (see [Storing the auth token in Angular](../front-end/angular/ANGULAR.md#storing-the-auth-token-in-angular)); the BFF keeps them server-side.
- In Spring: `spring-boot-starter-oauth2-client` for the client side (login, BFF), `spring-boot-starter-oauth2-resource-server` for the API (see [Spring Security](../back-end/spring/SPRING_SECURITY.md)).
