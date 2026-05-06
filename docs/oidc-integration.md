# OpenID Connect (OIDC) Login

This document describes how to enable OIDC login for Huly and highlights two deployment details that matter in practice:

- Huly's account service is often mounted behind a reverse-proxy prefix such as `/_accounts`
- some identity providers, including certain Keycloak setups, sign ID tokens with an algorithm other than the OpenID Connect default `RS256`

## Overview

Huly enables the OIDC login provider when these environment variables are set for the account service:

- `OPENID_CLIENT_ID`
- `OPENID_CLIENT_SECRET`
- `OPENID_ISSUER`

Optional variables:

- `OPENID_DISPLAY_NAME`
- `OPENID_ID_TOKEN_SIGNING_ALG`

The login callback URL is:

```text
https://<your-huly-host>/_accounts/auth/openid/callback
```

## Required Environment Variables

Configure these values for the account service:

```bash
OPENID_CLIENT_ID=<your-client-id>
OPENID_CLIENT_SECRET=<your-client-secret>
OPENID_ISSUER=<your-issuer-url>
OPENID_DISPLAY_NAME=OpenID Connect
```

Example for Keycloak 15:

```bash
OPENID_ISSUER=https://<your-keycloak-host>/auth/realms/<your-realm>
```

Use the `issuer` value from the provider's `.well-known/openid-configuration` document whenever possible.

## Optional: Explicit ID Token Signing Algorithm

If your provider issues ID tokens with an algorithm other than `RS256`, set:

```bash
OPENID_ID_TOKEN_SIGNING_ALG=ES256
```

This is particularly relevant for some Keycloak installations where the effective realm or client signing algorithm is `ES256`, but the client metadata available to Huly does not expose that fact clearly enough for automatic configuration.

Why this is needed:

- OpenID Connect client metadata defaults `id_token_signed_response_alg` to `RS256` when unspecified
- `openid-client` follows that default
- some providers can still issue `ES256` ID tokens for the client

In that case Huly will fail login with an error similar to:

```text
unexpected JWT alg received, expected RS256, got: ES256
```

Setting `OPENID_ID_TOKEN_SIGNING_ALG=ES256` resolves that mismatch.

## Reverse Proxy Requirements

If Huly is served behind a reverse proxy, the account service must see the public HTTPS request context correctly.

Required behavior:

- `ACCOUNTS_URL` must point to the public prefixed endpoint, for example:

```bash
ACCOUNTS_URL=https://huly.example.com/_accounts
```

- the reverse proxy must forward HTTPS semantics correctly to the account service
- if the account service is mounted behind `/_accounts`, the public callback URL must remain:

```text
https://huly.example.com/_accounts/auth/openid/callback
```

If the proxy strips the `/_accounts` prefix before the request reaches the account service, Huly still reconstructs the public callback URL from `ACCOUNTS_URL` so that `openid-client` validates the callback against the correct external URL.

## Cookie Requirements

OIDC login relies on the session surviving a cross-site redirect to the identity provider and back.

For HTTPS deployments, Huly uses secure session cookies with cross-site support:

- `Secure`
- `HttpOnly`
- `SameSite=None`

If your reverse proxy does not forward HTTPS correctly, the account service may reject secure cookies and the OIDC flow will fail before the callback completes.

## Keycloak Notes

For Keycloak:

- configure the client as a confidential client with a client secret
- enable the standard authorization code flow
- set the redirect URI to:

```text
https://<your-huly-host>/_accounts/auth/openid/callback
```

- if the Keycloak user is already signed in, you may not see a login page at all; a silent redirect back to Huly is expected
- if Keycloak signs ID tokens with `ES256`, set `OPENID_ID_TOKEN_SIGNING_ALG=ES256`

## Troubleshooting

### Redirects back to `/login`

Check:

- `OPENID_ISSUER`
- `OPENID_CLIENT_ID`
- `OPENID_CLIENT_SECRET`
- reverse proxy forwarding for HTTPS
- cookie behavior across the redirect

### `unexpected JWT alg received, expected RS256, got: ES256`

Set:

```bash
OPENID_ID_TOKEN_SIGNING_ALG=ES256
```

### Secure cookie errors

If logs contain errors similar to:

```text
Cannot send secure cookie over unencrypted connection
```

then the reverse proxy is not forwarding HTTPS correctly to the account service.

### Callback path mismatches

If Huly is mounted behind `/_accounts`, ensure that:

- the public callback URL uses `/_accounts/auth/openid/callback`
- `ACCOUNTS_URL` also includes `/_accounts`

## Verification

After deployment:

1. Check that the provider is listed:

```text
GET /_accounts/providers
```

2. Confirm the OIDC redirect starts from:

```text
/_accounts/auth/openid
```

3. Verify the login button appears in the UI with the configured display name.

4. Complete the login flow and confirm Huly redirects to the normal authenticated application instead of returning to `/login` or a callback error page.
