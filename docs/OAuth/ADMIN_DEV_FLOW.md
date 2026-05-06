# OAuth Flow - BeatGacha

## Admin Setup (one-time)

1. Go to `/admin` → **OAuth Apps** tab
2. Click **Create App** — enter name (e.g. "BeatSaber Mod"), optional description
3. Copy the `client_id` shown
4. Click **Manage** on the new app:
   - Enable **Require PKCE** (recommended for mods/desktop apps — disables client_secret entirely)
   - Add `*` as a redirect URI → **Add URI** (allows any `http://localhost` port)

> **PKCE vs client_secret:** PKCE apps have no secret embedded in code. Use PKCE for any mod, desktop app, or public client. Use client_secret only for server-side apps where the secret stays on a private backend.

---

## User Flow - PKCE (recommended for mods)

### Step 1 - Generate PKCE values in your mod

```
code_verifier  = cryptographically random 64 bytes → base64url encode
code_challenge = SHA-256(code_verifier) → base64url encode
```

### Step 2 - Open this URL in the user's browser

```
https://beatgacha.com/oauth/authorize
  ?client_id=<your_client_id>
  &redirect_uri=http://localhost:7890/callback
  &response_type=code
  &scope=profile+packs+cards+trades
  &state=<random string your mod generated>
  &code_challenge=<base64url SHA-256 of verifier>
  &code_challenge_method=S256
```

1. If not logged into BeatGacha → Discord login → returns to consent page
2. User sees consent page: app name, scope list, "Secured with PKCE (S256)" badge
3. User clicks **Approve**
4. Browser redirects to `http://localhost:7890/callback?code=ABC123&state=<same random>`

Your mod must be running a tiny HTTP server on that port to catch the redirect.

### Step 3 - Exchange code for tokens (no client_secret needed)

```http
POST https://beatgacha.com/api/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=ABC123
&redirect_uri=http://localhost:7890/callback
&client_id=<your_client_id>
&code_verifier=<the original verifier from Step 1>
```

Response:
```json
{
  "access_token": "abc...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "xyz...",
  "scope": "profile packs cards trades"
}
```

Store both tokens. The server verifies `SHA-256(code_verifier) == code_challenge` — an attacker who intercepted the `code` cannot exchange it without the verifier that only your mod holds in memory.

---

## User Flow - client_secret (server-side apps only)

Only use this if your backend is a private server and the secret never ships in client code.

```http
POST https://beatgacha.com/api/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=ABC123
&redirect_uri=https://yourserver.com/callback
&client_id=<your_client_id>
&client_secret=<your_client_secret>
```

The `client_secret` is shown once on app creation and cannot be retrieved again.

---

## Calling the API on Behalf of a User

```http
GET https://beatgacha.com/api/packs/open
Authorization: Bearer <access_token>
```

Any non-admin endpoint works. Access token expires in **1 hour**.

### Refreshing (silent, no user interaction)

```http
POST https://beatgacha.com/api/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=xyz...
&client_id=<your_client_id>
```

> For PKCE apps omit `client_secret`. For secret apps include it.

Returns new `access_token` + new `refresh_token` (old one invalidated — rotation). Refresh token lasts **30 days**. If it expires the user must approve again.

---

## Revoking a Token

```http
POST https://beatgacha.com/api/oauth/revoke
Content-Type: application/x-www-form-urlencoded

token=<access_or_refresh_token>
&client_id=<your_client_id>
```

Always returns HTTP 200.

---

## What to Give Users / Documentation Checklist

Everything a user or third-party developer needs to integrate with BeatGacha OAuth.

### Endpoints

| Purpose | URL |
|---------|-----|
| Authorization (consent page) | `https://beatgacha.com/oauth/authorize` |
| Token exchange / refresh | `https://beatgacha.com/api/oauth/token` |
| User profile | `https://beatgacha.com/api/oauth/userinfo` |
| Token introspection | `https://beatgacha.com/api/oauth/introspect` |
| Token revocation | `https://beatgacha.com/api/oauth/revoke` |

### Values you hand the developer

| Value | Where to get it | Notes |
|-------|----------------|-------|
| `client_id` | Admin panel → Create App | Public, safe to ship in mod code |
| `client_secret` | Shown once on creation | **Server-side apps only** — never ship in mod/client code |
| Allowed scopes | See below | Tell them which scopes your app was registered with |

### Available Scopes

| Scope | What it grants |
|-------|---------------|
| `profile` | Username, avatar |
| `packs` | View pack counts, open packs |
| `cards` | View card collection, card shards |
| `trades` | Initiate and manage trades |

Request only what your app needs. Users see the scope list on the consent page.

### Redirect URI rules

- Must be registered in the admin panel before use
- Exact match required (byte-for-byte)
- `*` wildcard = any `http://localhost` URI (any port, any path) — for mods/local tools
- Production apps must use `https://`

### Template: what to paste into your mod's README

```
## BeatGacha Integration

This mod uses BeatGacha OAuth to access your account.

On first launch, your browser will open to:
  https://beatgacha.com/oauth/authorize

Log in with Discord if prompted, then click **Approve** to grant access.
Your token is stored locally and used to open packs / view your collection.
You can revoke access at any time from your BeatGacha settings.
```

### Template: what to paste into your mod's config / constants file

```csharp
// BeatGacha OAuth
const string CLIENT_ID    = "<your_client_id>";   // from admin panel
const string REDIRECT_URI = "http://localhost:7890/callback"; // your local callback port
const string SCOPES       = "profile packs cards trades invites";
const string AUTH_URL     = "https://beatgacha.com/oauth/authorize";
const string TOKEN_URL    = "https://beatgacha.com/api/oauth/token";
```
