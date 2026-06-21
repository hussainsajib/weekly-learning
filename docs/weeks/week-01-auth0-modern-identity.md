# Week 1 — Auth0 & Modern Identity

**Week of:** June 8, 2026  
**Estimated study time:** ~2 hours  
**Tags:** `auth` `security` `identity` `jwt`

---

## Overview

Identity and authentication are the front door to every system you build. Auth0 is one of the most widely adopted Identity-as-a-Service (IDaaS) platforms, and understanding it deeply means understanding the entire modern auth ecosystem — OAuth 2.0, OpenID Connect (OIDC), JWTs, and secure token flows. This is foundational knowledge that underpins everything from your own apps to the integration platform middleware's service-to-service calls.

By the end of this week you will understand: how Auth0 is structured, how JWTs work internally, which OAuth flow to use in which situation, and how to integrate Auth0 into a Python/FastAPI service.

---

## 1. Auth0 Architecture

### Tenant

A **tenant** is Auth0's top-level isolation unit — think of it as a dedicated namespace for your identity configuration. It has its own domain (e.g., `your-company.auth0.com`), its own user database, its own set of applications, APIs, and rules.

You typically create one tenant per environment:
- `myapp-dev.auth0.com`
- `myapp-staging.auth0.com`
- `myapp.auth0.com` (production)

### Applications

An **Application** in Auth0 represents a client — something that will request tokens. Auth0 has four application types:

| Type | When to use |
|------|-------------|
| **Regular Web App** | Server-side apps (Flask, Django, FastAPI with server rendering) |
| **Single Page App (SPA)** | React, Vue, Angular — runs in the browser |
| **Native** | iOS, Android, Electron desktop |
| **Machine to Machine (M2M)** | Service-to-service, background jobs, CLIs |

Each application gets a **Client ID** (public) and a **Client Secret** (keep private for confidential clients).

### APIs (Resource Servers)

An **API** in Auth0 is the resource you want to protect. You define it by giving it a unique **Audience** (a URI, e.g., `https://api.myapp.com`). When an application requests a token, it specifies which audience it needs access to, and Auth0 mints a token scoped to that audience.

```
Application (client) → requests token for → API (audience)
```

### Connections

**Connections** are sources of identity. Auth0 supports:
- **Database connections** — Auth0 stores usernames/passwords
- **Social connections** — Google, GitHub, Facebook OAuth
- **Enterprise connections** — SAML, Active Directory, LDAP
- **Passwordless** — email magic links, SMS OTP

---

## 2. JSON Web Tokens (JWT) In Depth

A JWT is a compact, URL-safe token with three Base64URL-encoded parts separated by dots:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9   ← Header
.eyJzdWIiOiJ1c2VyfDEyMyIsImV4cCI6MTc1MH0   ← Payload
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c   ← Signature
```

### Header

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "abc123"
}
```

- `alg` — signing algorithm. Auth0 uses `RS256` (RSA + SHA-256) by default, which means it signs with a **private key** and you verify with a **public key** (JWKS endpoint). Avoid `HS256` for public APIs — it requires sharing the secret.
- `kid` — Key ID, used to look up the correct public key when Auth0 rotates keys.

### Payload (Claims)

```json
{
  "iss": "https://your-tenant.auth0.com/",
  "sub": "auth0|64abc123def456",
  "aud": ["https://api.myapp.com", "https://your-tenant.auth0.com/userinfo"],
  "iat": 1717804800,
  "exp": 1717808400,
  "scope": "read:data write:data",
  "azp": "CLIENT_ID_OF_APP",
  "permissions": ["read:data", "write:data"]
}
```

**Registered claims (IANA standard):**

| Claim | Meaning |
|-------|---------|
| `iss` | Issuer — must match your Auth0 domain |
| `sub` | Subject — the user or M2M client ID |
| `aud` | Audience — the API(s) this token is valid for |
| `iat` | Issued At (Unix timestamp) |
| `exp` | Expiration (Unix timestamp) |
| `nbf` | Not Before — token invalid before this time |
| `jti` | JWT ID — unique identifier, used to prevent replay |

**Auth0-specific claims:**
- `scope` — space-separated list of scopes granted
- `permissions` — RBAC permissions (requires Auth0 RBAC enabled)
- `azp` — Authorized Party (the client that requested the token)

### Signature Verification

When you receive a JWT, **never trust it without verification**:

```python
import jwt
from jwt import PyJWKClient

jwks_uri = "https://your-tenant.auth0.com/.well-known/jwks.json"
jwks_client = PyJWKClient(jwks_uri)

def verify_token(token: str, audience: str) -> dict:
    signing_key = jwks_client.get_signing_key_from_jwt(token)
    payload = jwt.decode(
        token,
        signing_key.key,
        algorithms=["RS256"],
        audience=audience,
        issuer="https://your-tenant.auth0.com/"
    )
    return payload
```

The library fetches the public key from Auth0's JWKS endpoint, verifies the signature, and checks `exp`, `iss`, and `aud`. **All three must pass.**

### Access Token vs. ID Token

| | Access Token | ID Token |
|--|-------------|----------|
| **Purpose** | Authorize API requests | Authenticate the user |
| **Audience** | Your API | Your client app |
| **Contains** | Scopes, permissions | User profile (name, email, picture) |
| **Send to API?** | Yes (Authorization header) | Never |
| **Standard** | OAuth 2.0 | OpenID Connect |

A common mistake: sending the ID token to your API. Always send the access token to APIs.

---

## 3. OAuth 2.0 Flows

OAuth 2.0 is an **authorization** framework. It defines how an application obtains an access token without handling the user's credentials. Auth0 implements all major flows.

### Flow Selection Guide

```
Is there a user involved?
├── Yes
│   ├── Is the client a browser/mobile app?
│   │   ├── Yes → Authorization Code + PKCE
│   │   └── No (server-side) → Authorization Code (+ PKCE still recommended)
│   └── No user, just a service/job?
│       └── Client Credentials
└── Special cases
    ├── Smart TV / CLI with no browser → Device Authorization
    └── Legacy trusted first-party apps only → Resource Owner Password (avoid)
```

### Authorization Code Flow + PKCE

The most secure and most common flow for user-facing apps. PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks.

```
User → App → Auth0 (login page) → Auth0 → App (code) → App → Auth0 (code + verifier) → App (tokens)
```

**Step by step:**

1. App generates a random `code_verifier` (43-128 chars) and derives `code_challenge = BASE64URL(SHA256(code_verifier))`
2. App redirects user to Auth0:
   ```
   GET https://tenant.auth0.com/authorize?
     response_type=code
     &client_id=CLIENT_ID
     &redirect_uri=https://app.com/callback
     &scope=openid profile email
     &audience=https://api.myapp.com
     &code_challenge=CHALLENGE
     &code_challenge_method=S256
     &state=RANDOM_STATE
   ```
3. User authenticates at Auth0
4. Auth0 redirects back with `?code=AUTH_CODE&state=RANDOM_STATE`
5. App verifies `state` (CSRF protection), then exchanges code:
   ```
   POST /oauth/token
   {
     "grant_type": "authorization_code",
     "client_id": "CLIENT_ID",
     "code_verifier": "VERIFIER",
     "code": "AUTH_CODE",
     "redirect_uri": "https://app.com/callback"
   }
   ```
6. Auth0 returns access token, ID token, refresh token

### Client Credentials Flow

For M2M — no user involved. Your service authenticates directly as itself.

```
Service → Auth0 (client_id + client_secret) → Service (access token) → Protected API
```

```python
import httpx

async def get_m2m_token() -> str:
    response = await httpx.post(
        "https://your-tenant.auth0.com/oauth/token",
        json={
            "grant_type": "client_credentials",
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
            "audience": "https://api.myapp.com",
        }
    )
    return response.json()["access_token"]
```

This is the pattern you'd use for the integration platform middleware calling a protected internal service.

### Device Authorization Flow

For devices with limited input (CLIs, smart TVs). The device shows a short code, user goes to `device.auth0.com` on another device and enters it.

### Refresh Tokens

Access tokens are short-lived (default 24h in Auth0, often set to 15 minutes). A **refresh token** lets you get a new access token without the user re-authenticating.

```python
async def refresh_access_token(refresh_token: str) -> dict:
    response = await httpx.post(
        "https://your-tenant.auth0.com/oauth/token",
        json={
            "grant_type": "refresh_token",
            "client_id": CLIENT_ID,
            "refresh_token": refresh_token,
        }
    )
    return response.json()
```

**Refresh token rotation:** Auth0 can issue a new refresh token each time you use one and invalidate the old one. This limits the blast radius if a refresh token is stolen.

---

## 4. OpenID Connect (OIDC) Layer

OIDC is a thin **authentication** layer built on top of OAuth 2.0. It adds:
- The **ID Token** (a JWT containing user identity info)
- Standardized claims (`sub`, `name`, `email`, `picture`, etc.)
- The `/userinfo` endpoint
- Discovery document at `/.well-known/openid-configuration`

When you add `openid` to the `scope`, you get OIDC in addition to OAuth 2.0.

**Discovery document** (auto-configure your client from this):
```
GET https://your-tenant.auth0.com/.well-known/openid-configuration
```
Returns JWKS URI, supported scopes, token endpoint, etc.

---

## 5. Auth0 Extensibility

### Actions

**Actions** are JavaScript/TypeScript functions that execute during Auth0 flows. Replace the older "Rules" and "Hooks".

```
Login flow:
User authenticates → [Post Login Actions run] → Token issued
```

Common use cases:
- Enrich tokens with custom claims from a database
- Block login based on risk signals
- Send login events to analytics
- Multi-factor authentication logic

```javascript
// Action: add user's roles to the access token
exports.onExecutePostLogin = async (event, api) => {
  const roles = event.authorization?.roles ?? [];
  api.accessToken.setCustomClaim('https://myapp.com/roles', roles);
};
```

### Rules (Legacy — prefer Actions)

Same concept, older API. If you encounter them in an existing Auth0 tenant, understand they run in order and share a `context` object.

---

## 6. FastAPI Integration

A complete, production-ready Auth0 middleware for FastAPI:

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from jwt import PyJWKClient
from functools import lru_cache

AUTH0_DOMAIN = "your-tenant.auth0.com"
API_AUDIENCE = "https://api.myapp.com"
ALGORITHMS = ["RS256"]

security = HTTPBearer()

@lru_cache()
def get_jwks_client() -> PyJWKClient:
    return PyJWKClient(f"https://{AUTH0_DOMAIN}/.well-known/jwks.json")

def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)) -> dict:
    token = credentials.credentials
    try:
        signing_key = get_jwks_client().get_signing_key_from_jwt(token)
        payload = jwt.decode(
            token,
            signing_key.key,
            algorithms=ALGORITHMS,
            audience=API_AUDIENCE,
            issuer=f"https://{AUTH0_DOMAIN}/",
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token expired")
    except jwt.JWTClaimsError as e:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail=f"Invalid claims: {e}")
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")

def require_scope(required_scope: str):
    def checker(payload: dict = Depends(verify_token)):
        token_scopes = payload.get("scope", "").split()
        if required_scope not in token_scopes:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Missing required scope: {required_scope}"
            )
        return payload
    return checker

# Usage in routes:
from fastapi import FastAPI
app = FastAPI()

@app.get("/data")
def get_data(payload: dict = Depends(require_scope("read:data"))):
    return {"user": payload["sub"], "data": "..."}
```

---

## 7. Machine-to-Machine (M2M) Pattern

M2M is common in microservice architectures. Key design points:

**Token caching** — M2M tokens should be cached until just before expiry. Fetching a new one on every request is wasteful and will hit rate limits.

```python
import time
from dataclasses import dataclass

@dataclass
class TokenCache:
    token: str
    expires_at: float

_cache: TokenCache | None = None

async def get_cached_m2m_token() -> str:
    global _cache
    if _cache and time.time() < _cache.expires_at - 60:  # 60s buffer
        return _cache.token
    
    data = await fetch_new_m2m_token()  # calls /oauth/token
    _cache = TokenCache(
        token=data["access_token"],
        expires_at=time.time() + data["expires_in"]
    )
    return _cache.token
```

**Rate limits:** Auth0's free tier allows 1,000 M2M tokens/month. Production tiers are much higher, but still — cache aggressively.

---

## 8. Security Best Practices

| Practice | Why |
|----------|-----|
| Always verify `iss` and `aud` | Prevents token confusion attacks (a token for API A used on API B) |
| Use RS256, not HS256 | RS256 uses asymmetric keys; you don't need to share a secret to verify |
| Short access token lifetime | Limits blast radius if a token is stolen (can't revoke JWTs once issued) |
| Store tokens in memory, not localStorage | `localStorage` is accessible to XSS; use `httpOnly` cookies or in-memory |
| Validate `state` in the callback | Prevents CSRF in the auth flow |
| Use PKCE even for server-side apps | Defense in depth |
| Rotate refresh tokens | Auth0 "Refresh Token Rotation" setting |
| Scope minimization | Request only the scopes you need |

### The "Can't Revoke JWTs" Problem

JWTs are stateless. Once issued, you can't revoke them before expiry — the API doesn't check with Auth0 on every request. Solutions:
1. **Short expiry** (15 minutes) — limits the window
2. **Opaque tokens** — Auth0 can issue opaque tokens; your API introspects each one at Auth0 (slower but revocable)
3. **Token blocklist** — maintain a Redis set of revoked JTIs; check on each request (adds latency)

---

## 9. Auth0 vs. Other Providers

| | Auth0 | Okta | AWS Cognito | Keycloak |
|--|-------|------|-------------|----------|
| **Hosting** | Cloud (SaaS) | Cloud (SaaS) | Cloud (AWS) | Self-hosted |
| **Free tier** | 7,500 MAU | Limited | 50,000 MAU | Free (infra cost) |
| **PKCE support** | ✅ | ✅ | ✅ | ✅ |
| **Social logins** | 30+ | 30+ | Limited | Via plugins |
| **Actions/Rules** | Actions (JS) | Inline hooks | Lambda triggers | Scripts |
| **Enterprise SSO** | ✅ | ✅ | ✅ | ✅ |
| **Vendor lock-in** | Medium | High | High | Low |
| **Complexity** | Low | Medium | Medium | High |

Keycloak is the go-to for self-hosted/on-prem requirements. Auth0 wins on developer experience. Cognito wins if you're deep in AWS.

---

## 10. Key Concepts Summary

```
Identity Platform
├── Tenant (isolated namespace)
│   ├── Applications (clients that request tokens)
│   │   ├── Regular Web App
│   │   ├── SPA
│   │   ├── Native
│   │   └── M2M
│   ├── APIs (resource servers with audiences)
│   ├── Connections (identity sources)
│   │   ├── Database
│   │   ├── Social
│   │   └── Enterprise (SAML/AD)
│   └── Actions (extensibility hooks)
│
├── Token Types
│   ├── Access Token (JWT → authorize API calls)
│   ├── ID Token (JWT → authenticate user identity)
│   └── Refresh Token (opaque → get new access tokens)
│
└── Flows
    ├── Auth Code + PKCE (user + browser/mobile)
    ├── Client Credentials (M2M, no user)
    ├── Device Authorization (CLI/TV)
    └── Refresh Token (renew without re-auth)
```

---

## Quiz — 20 Questions

Test your understanding. Try to answer without looking back, then check the answers below.

---

### Questions

**1.** What is an Auth0 "tenant" and why would you create one per environment?

**2.** A JWT has three parts. What does each part contain?

**3.** What does the `aud` claim in a JWT represent, and what happens if it doesn't match when your API verifies the token?

**4.** What is the difference between `RS256` and `HS256` JWT signing algorithms? Which does Auth0 use by default and why?

**5.** What is PKCE and what attack does it prevent? Is it only needed for public clients?

**6.** You have a Python background job that needs to call a protected internal API. Which OAuth 2.0 flow should you use?

**7.** What is the difference between an Access Token and an ID Token? What is the most common mistake developers make with these two?

**8.** Why should JWTs (access tokens) have a short expiry (e.g., 15 minutes)? What is the trade-off?

**9.** What is refresh token rotation and why does it improve security?

**10.** Your API receives a JWT. List all the things you must verify before trusting it.

**11.** A developer stores the access token in `localStorage`. What is the security risk?

**12.** What is Auth0's JWKS endpoint used for, and why does it exist instead of a single static public key?

**13.** What is an Auth0 "Action" and in which flows can it run?

**14.** What does the `scope` claim in a JWT represent? How does it differ from the `permissions` claim in Auth0?

**15.** You need to add a user's internal database role to the JWT so your API can use it for authorization. How do you do this in Auth0?

**16.** A user logs out. Their access token (valid for 15 more minutes) is already issued. How can you prevent it from being used if you suspect it's compromised?

**17.** What is token introspection, and when would you use opaque tokens instead of JWTs?

**18.** What is the `state` parameter in the Authorization Code flow? What attack does it prevent?

**19.** Auth0 rate limits M2M token requests. What is the correct pattern for handling M2M tokens in a high-throughput service?

**20.** Compare Auth0 and Keycloak: what are the main reasons you might choose one over the other?

---

### Answers

??? note "Reveal Answers"

    **1.** A tenant is Auth0's top-level isolated namespace — it has its own domain, user database, applications, APIs, and configuration. You create one per environment (dev/staging/prod) to prevent test users, configurations, and tokens from bleeding across environments.

    **2.** Header (algorithm + key ID), Payload (claims: sub, iss, aud, exp, custom data), Signature (cryptographic proof of integrity signed by the issuer).

    **3.** The `aud` claim identifies the intended recipient (your API's audience URI). If it doesn't match, the token verification library will throw a `JWTClaimsError`. This prevents a token issued for API A from being used on API B.

    **4.** `RS256` uses asymmetric RSA keys — Auth0 signs with a private key; you verify with a public key from the JWKS endpoint. `HS256` uses a shared secret — both sides must know it, creating a distribution and trust problem. Auth0 defaults to RS256 so APIs can verify tokens without knowing a secret.

    **5.** PKCE (Proof Key for Code Exchange) is a mechanism where the client generates a random `code_verifier`, hashes it to `code_challenge`, and sends the hash with the auth request. When exchanging the code for tokens, it sends the original verifier. This prevents an attacker who intercepts the authorization code from exchanging it. Not just for public clients — recommended for all clients as defense in depth.

    **6.** Client Credentials flow. No user is involved; the background job authenticates as itself using its `client_id` and `client_secret` to get an access token.

    **7.** Access Token = proof of authorization to call an API. ID Token = proof of who the user is. The most common mistake is sending the ID Token to an API as if it were an access token — the API should only accept access tokens scoped to its audience.

    **8.** JWTs cannot be revoked once issued. A short expiry limits the window of exposure if a token is stolen. The trade-off is more frequent token refreshes (handled transparently by the client using refresh tokens).

    **9.** Refresh token rotation issues a new refresh token each time you use one and invalidates the old one. If a refresh token is stolen and used, the legitimate holder's next use will fail (the old token is gone), triggering a re-authentication — limiting the attacker's window.

    **10.** Verify: (1) signature using the correct public key from JWKS, (2) `iss` matches your Auth0 domain, (3) `aud` matches your API audience, (4) `exp` is in the future (not expired), (5) `nbf` is in the past if present, (6) `alg` is what you expect (RS256).

    **11.** Any JavaScript running on the page (including injected via XSS) can read `localStorage`. If the site has an XSS vulnerability, the attacker can steal the token. Prefer in-memory storage or `httpOnly` cookies (which JS cannot read).

    **12.** The JWKS endpoint (`/.well-known/jwks.json`) publishes Auth0's public keys in JSON format so your API can download and cache them for signature verification. It exists (vs. a single static key) because Auth0 periodically rotates its signing keys; the `kid` header in the JWT tells you which key to use for verification.

    **13.** Actions are JavaScript/TypeScript serverless functions that execute at specific points in Auth0 flows (Login, Pre User Registration, Post User Registration, etc.). Most commonly used in the Post Login flow to enrich tokens with custom claims.

    **14.** `scope` represents OAuth 2.0 permissions requested by the client (e.g., `read:data write:data`). `permissions` is an Auth0-specific claim populated when RBAC is enabled, reflecting the user's assigned permissions from Auth0's role management. `scope` is what the client requested; `permissions` is what the user is allowed based on their roles.

    **15.** Use an Auth0 Action (Post Login flow). Query your database for the user's role using their `sub`, then call `api.accessToken.setCustomClaim('https://myapp.com/role', role)`. Prefix the custom claim with a URL namespace to avoid colliding with standard claims.

    **16.** Options: (1) Maintain a JWT blocklist (Redis set of blocked `jti` values) checked on every API request, (2) Use opaque tokens with introspection (Auth0 checks validity server-side), (3) If using refresh tokens, revoke the refresh token immediately to prevent issuing new access tokens. The current access token with 15 minutes remaining is effectively unstoppable without a blocklist.

    **17.** Token introspection is when your API calls Auth0's `/introspect` endpoint to validate a token server-side on each request (rather than validating locally). Opaque tokens (non-JWT) require this because they have no verifiable structure. Use when you need immediate revocation capability, at the cost of a network round-trip per request.

    **18.** `state` is a random string generated by the client, included in the authorization request and returned unchanged in the callback. The client verifies it matches before proceeding. This prevents CSRF attacks where an attacker tricks a user's browser into completing an auth flow initiated by the attacker.

    **19.** Cache the token in memory (not per-request). Store the token and its expiry time. Before each use, check if expiry - (buffer of 60s) > now. Only fetch a new token when the cached one is about to expire. This turns thousands of token requests into a handful per day.

    **20.** Choose Auth0 for: developer experience, fast setup, no infrastructure to run, built-in social/enterprise connectors, managed service SLA. Choose Keycloak for: self-hosted (data sovereignty, on-prem requirements), no per-MAU cost at scale, full control over data, deep SAML/enterprise SSO customization. Main downside of Keycloak: you own the operations burden.
