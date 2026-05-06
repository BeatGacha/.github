# BeatGacha OAuth - Developer Guide

This guide covers everything needed to integrate "Login with BeatGacha" into your app or mod.
You will need a `client_id` and (for server-side apps) a `client_secret` from a developer.

---

## What You Get

| Value | Example | Notes |
|-------|---------|-------|
| `client_id` | `8f6fc706-f8c0-49ee-...` | Public — safe to include in client code |
| `client_secret` | `a3f9...` | **Server-side only** — never ship in mod/desktop code |
| Auth URL | See below | Pre-built by developer, includes your `client_id` and allowed scopes |

---

## Endpoints

| Purpose | URL |
|---------|-----|
| Authorization (consent page) | `https://beatgacha.com/oauth/authorize` |
| Token exchange / refresh | `https://beatgacha.com/api/oauth/token` |
| User profile | `https://beatgacha.com/api/oauth/userinfo` |
| Token revocation | `https://beatgacha.com/api/oauth/revoke` |

---

## Which Flow To Use?

| App type | Flow |
|----------|------|
| **Mod / desktop app / CLI** | PKCE — no client_secret needed |
| **Server-side web app** | Authorization Code + client_secret |

---

## Available Scopes

Scopes control exactly what your app can see or do. Configured per-app by a BeatGacha admin.

| Scope | What it grants | Fields exposed via `/api/oauth/userinfo` |
|-------|---------------|------------------------------------------|
| `profile` | View and update profile, settings, notifications, ScoreSaber link, featured cards | `id`, `username`, `avatar` |
| `packs` | Claim free packs, buy packs with shards, open packs | `current_packs`, `reward_packs`, `last_pack_awarded` |
| `cards` | View, discard, and lock/unlock cards; create, edit, delete, and pin binders; buy binder slots | `card_shards` |
| `trades` | View own trades; send, accept, deny, and cancel trade offers | _(no extra fields — grants action permission only)_ |
| `invites` | View, create, and redeem invite codes | _(no extra fields — grants action permission only)_ |

Only request the scopes your app actually needs. Users see exactly what each scope gives access to on the consent page before approving.

## Flow A - PKCE (Mods / Desktop Apps)

No `client_secret` required. A developer will enable PKCE on your app.

### Step 1 - Generate PKCE values

Every time a user logs in, generate fresh values:

```csharp
// C# example
byte[] verifierBytes = RandomNumberGenerator.GetBytes(64);
string code_verifier = Base64UrlEncode(verifierBytes);

byte[] challengeBytes = SHA256.HashData(Encoding.UTF8.GetBytes(code_verifier));
string code_challenge = Base64UrlEncode(challengeBytes);

static string Base64UrlEncode(byte[] bytes) =>
    Convert.ToBase64String(bytes).TrimEnd('=').Replace('+', '-').Replace('/', '_');
```

```python
# Python example
import secrets, hashlib, base64

code_verifier = secrets.token_urlsafe(64)
digest = hashlib.sha256(code_verifier.encode()).digest()
code_challenge = base64.urlsafe_b64encode(digest).rstrip(b'=').decode()
```

```typescript
// TypeScript example
import crypto from 'crypto';
const code_verifier = crypto.randomBytes(64).toString('base64url');
const challengeBytes = crypto.createHash('sha256').update(code_verifier).digest();
const code_challenge = challengeBytes.toString('base64url');
```

### Step 2 - Generate a state value

```csharp
string state = Convert.ToBase64String(RandomNumberGenerator.GetBytes(16));
// Save it — you must verify it later
```
```python
state = secrets.token_urlsafe(16)
# Save it — you must verify it later
```
```typescript
const state = crypto.randomBytes(16).toString('base64url');
// Save it — you must verify it later
```

### Step 3 - Open the authorization URL in the user's browser

Take the base URL provided by a developer from BeatGacha and append `code_challenge` and `state`:

```
https://beatgacha.com/oauth/authorize
  ?client_id=<your_client_id>
  &redirect_uri=http://localhost:7890/callback
  &response_type=code
  &scope=profile packs cards trades
  &state=<your random state>
  &code_challenge=<code_challenge from Step 1>
  &code_challenge_method=S256
```

> The `redirect_uri` must exactly match one registered with BeatGacha.
> For mods/local tools, `http://localhost:<any port>` is accepted.

The user will see a BeatGacha consent page. After they click **Approve**, their browser
redirects to your `redirect_uri`:

```
http://localhost:7890/callback?code=ABC123&state=<your state>
```

Your app must be listening on that port to receive this redirect.

### Step 4 - Verify state, then exchange code for tokens

**First - verify state matches what you generated in Step 2. If not, abort.**

```http
POST https://beatgacha.com/api/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=ABC123
&redirect_uri=http://localhost:7890/callback
&client_id=<your_client_id>
&code_verifier=<code_verifier from Step 1>
```

Response:

```json
{
  "access_token": "3a9f...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "7bc2...",
  "scope": "profile packs cards trades"
}
```

Store both tokens securely. The `access_token` expires in **1 hour**. The `refresh_token` expires in **30 days**.

---

## Flow B - Authorization Code + client_secret (Server-Side Apps)

Use this only if your backend is a private server and the secret never reaches client code.

### Step 1 - Generate state

```js
const state = crypto.randomBytes(16).toString('base64url');
// Store in session
```

### Step 2 - Redirect user to authorization URL

```
https://beatgacha.com/oauth/authorize
  ?client_id=<your_client_id>
  &redirect_uri=https://yourserver.com/callback
  &response_type=code
  &scope=profile packs cards trades
  &state=<your state>
```

### Step 3 - Exchange code for tokens

After user approves and is redirected to `https://yourserver.com/callback?code=ABC&state=...`:

**Verify state first**, then:

```http
POST https://beatgacha.com/api/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=ABC123
&redirect_uri=https://yourserver.com/callback
&client_id=<your_client_id>
&client_secret=<your_client_secret>
```

---

## Calling the API

Include the access token as a Bearer header on every request:

```http
GET https://beatgacha.com/api/oauth/userinfo
Authorization: Bearer <access_token>
```

### Scopes and fields returned

| Scope | What it grants | Fields in `/api/oauth/userinfo` response |
|-------|---------------|------------------------------------------|
| `profile` | View/update profile, settings, notifications, ScoreSaber, featured cards | `id`, `username`, `avatar` |
| `packs` | Claim, buy, and open packs | `current_packs`, `reward_packs`, `last_pack_awarded` |
| `cards` | Discard/lock cards, manage binders | `card_shards` |
| `trades` | View, send, accept, deny, cancel trades | _(action permission only — no extra fields)_ |
| `invites` | View, create, and redeem invite codes | _(action permission only — no extra fields)_ |

`sub` (the user's BeatGacha ID) is always included regardless of scope.

Only scopes granted by the admin for your app are available. Requesting a scope not on the allowed list will result in an error on the consent page.

---

## Refreshing Tokens (Silent - No User Interaction)

Access tokens expire after **1 hour**. Use the refresh token to get a new pair silently:

```http
POST https://beatgacha.com/api/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=<your_refresh_token>
&client_id=<your_client_id>
```

For secret-based apps, also include `&client_secret=<your_client_secret>`.

Response is a new `access_token` + new `refresh_token`. **The old refresh token is immediately invalidated** — store the new one.

If the refresh token itself expires (after 30 days of no use), the user must log in and approve again.

---

## Revoking a Token

Call this when the user logs out of your app:

```http
POST https://beatgacha.com/api/oauth/revoke
Content-Type: application/x-www-form-urlencoded

token=<access_or_refresh_token>
&client_id=<your_client_id>
```

Always returns HTTP 200. Revokes access immediately.

---

## State Parameter - What It Is

`state` is a random value you generate before sending the user to BeatGacha.
BeatGacha echoes it back unchanged in the redirect. You check it matches.

**Why it matters:** without this check, an attacker can trick your app into accepting a code they obtained, linking their BeatGacha account to your user's session.

**Minimum requirement:** at least 16 random bytes, base64url or hex encoded. Never reuse it.

---

## Error Responses

All errors from `/api/oauth/token` follow RFC 6749 format:

```json
{ "error": "invalid_grant", "error_description": "Authorization code invalid, expired, or already used" }
```

Common errors:

| `error` | Cause |
|---------|-------|
| `invalid_client` | Wrong `client_id` or `client_secret` / missing `code_verifier` |
| `invalid_grant` | Code expired, already used, or `code_verifier` mismatch |
| `unsupported_grant_type` | Unknown `grant_type` value |

The consent page returns errors as query params on your `redirect_uri`:

```
http://localhost:7890/callback?error=access_denied&state=<your state>
```

| `error` | Cause |
|---------|-------|
| `access_denied` | User clicked Deny |
