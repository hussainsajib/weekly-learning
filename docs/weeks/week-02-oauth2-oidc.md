# Week 2 — OAuth 2.0 & OIDC Deep Dive

**Week of:** June 15, 2026
**Estimated study time:** ~2 hours
**Tags:** `auth` `security` `oauth`

---

## Overview

OAuth 2.0 and OpenID Connect (OIDC) are the foundational protocols behind nearly every modern authentication and authorization system — from "Sign in with Google" to machine-to-machine API access in enterprise platforms. Week 1 introduced Auth0 as a concrete implementation. This week you go one level deeper: you're reading the spec, understanding the flows mechanically, and learning to reason about security trade-offs rather than just copy-pasting SDK code.

For a senior/staff engineer, protocol-level understanding matters because you'll inevitably hit edge cases that the SDK hides from you: a client that can't do PKCE, a resource server that needs to validate tokens without calling Auth0, a Salesforce connected app that uses a non-standard flow. When something breaks in production, you need to be able to read a raw HTTP exchange and diagnose it.

Your CRM-EHR Integration Platform stack makes this immediately relevant. The middleware (`crm-middleware`) exposes REST APIs that the ETL, Salesforce, and various internal tools consume. These callers need to authenticate. The middleware also calls the EHR system's BDE backend, which requires its own credentials. Understanding Client Credentials flow, token caching, and introspection lets you implement that authentication layer correctly — and audit the existing implementation for vulnerabilities.

By the end of this week you'll be able to: implement every major OAuth 2.0 grant type from scratch in Python, distinguish ID tokens from access tokens with precision, explain PKCE's role and why Implicit flow was deprecated, validate JWTs locally in FastAPI, compare provider trade-offs with specifics, and identify common token storage mistakes in web and server contexts.

---

## 1. OAuth 2.0 — What the Protocol Actually Is

OAuth 2.0 (RFC 6749) is an *authorization* framework, not an authentication protocol. This distinction is the most commonly misunderstood fact in the space: OAuth 2.0 answers "can this client access this resource on behalf of this user?" It does *not* answer "who is this user?"

The core abstraction is the **authorization grant**: a credential representing the resource owner's (user's) permission. The client exchanges this grant for an **access token**, which it uses to call the protected resource (your API).

### The Four Roles

```
┌─────────────────────────────────────────────────────────┐
│  Resource Owner   →  the user (or machine) granting     │
│                      access                             │
│                                                         │
│  Client           →  the application wanting access     │
│                      (your FastAPI service, a           │
│                      Salesforce connected app)          │
│                                                         │
│  Authorization    →  the server that issues tokens      │
│  Server           →  (Auth0, Okta, Cognito)             │
│                                                         │
│  Resource Server  →  the API being protected            │
│                      (crm-middleware)                   │
└─────────────────────────────────────────────────────────┘
```

The Resource Owner is typically a human user, but in machine-to-machine (M2M) flows it's the client application itself. The Authorization Server and Resource Server are often the same product (e.g., your middleware both issues and validates tokens) but logically they're separate — and keeping them conceptually separate helps you reason about security boundaries.

**Common mistake:** Treating `client_id` as a secret. `client_id` is a public identifier. `client_secret` is the secret. In public clients (browser SPAs, mobile apps), there *is* no client secret — which is exactly why PKCE exists.

---

## 2. The Grant Types — A Complete Walkthrough

### 2.1 Authorization Code + PKCE (The Right Way for User-Facing Apps)

This is the grant you should use for any flow where a human is present. The PKCE extension (RFC 7636) makes it safe for public clients. Here's the full exchange:

```
User clicks "Login"
        │
        ▼
Client generates:
  code_verifier  = random 43–128 char string
  code_challenge = BASE64URL(SHA256(code_verifier))

        │
        ▼
Client redirects browser to Authorization Server:
  GET /authorize?
    response_type=code
    &client_id=abc123
    &redirect_uri=https://app.example.com/callback
    &scope=openid profile email
    &state=random_csrf_token
    &code_challenge=<BASE64URL_SHA256>
    &code_challenge_method=S256

        │  (user logs in, grants consent)
        ▼
Auth Server redirects back:
  GET /callback?code=AUTH_CODE&state=random_csrf_token

        │  (client verifies state matches)
        ▼
Client POSTs to /token:
  grant_type=authorization_code
  &code=AUTH_CODE
  &redirect_uri=https://app.example.com/callback
  &client_id=abc123
  &code_verifier=ORIGINAL_VERIFIER

        │
        ▼
Auth Server validates: SHA256(code_verifier) == code_challenge
Returns:
  {
    "access_token": "...",
    "id_token": "...",       ← only if scope included "openid"
    "refresh_token": "...",
    "expires_in": 3600,
    "token_type": "Bearer"
  }
```

The `state` parameter prevents CSRF: a malicious site can't inject a code into your callback because they can't predict your state value. PKCE prevents authorization code interception: even if an attacker intercepts the code (via a malicious app on a device), they can't exchange it without the `code_verifier` that never left the client.

```python
import hashlib
import base64
import secrets
import httpx

def generate_pkce_pair() -> tuple[str, str]:
    verifier = secrets.token_urlsafe(64)
    digest = hashlib.sha256(verifier.encode()).digest()
    challenge = base64.urlsafe_b64encode(digest).rstrip(b"=").decode()
    return verifier, challenge

verifier, challenge = generate_pkce_pair()
print(f"verifier={verifier[:20]}...")
print(f"challenge={challenge[:20]}...")
```

### 2.2 Client Credentials (M2M — Most Relevant to Your Stack)

Used when there's no human involved. The client authenticates directly with the Authorization Server using its own credentials.

```
Client (crm-middleware) wants to call EHR BDE API:

  POST /oauth/token
  Content-Type: application/x-www-form-urlencoded

  grant_type=client_credentials
  &client_id=MIDDLEWARE_CLIENT_ID
  &client_secret=MIDDLEWARE_CLIENT_SECRET
  &audience=https://api.ehr-bde.internal

  Response:
  {
    "access_token": "eyJ...",
    "expires_in": 86400,
    "token_type": "Bearer"
  }
```

This is the flow for **your** integration platform middleware calling downstream services. The key engineering challenge is **token caching**: you must not request a new token on every API call (that's one extra round-trip + rate limit risk). You should cache the token and refresh it only when it's near expiry.

```python
import time
import httpx
from dataclasses import dataclass, field

@dataclass
class TokenCache:
    _token: str = ""
    _expires_at: float = 0.0

    def is_valid(self, buffer_seconds: int = 60) -> bool:
        return bool(self._token) and time.time() < (self._expires_at - buffer_seconds)

    def store(self, token: str, expires_in: int) -> None:
        self._token = token
        self._expires_at = time.time() + expires_in

    @property
    def token(self) -> str:
        return self._token

_cache = TokenCache()

async def get_client_credentials_token(
    token_url: str,
    client_id: str,
    client_secret: str,
    audience: str,
) -> str:
    if _cache.is_valid():
        return _cache.token

    async with httpx.AsyncClient() as client:
        resp = await client.post(
            token_url,
            data={
                "grant_type": "client_credentials",
                "client_id": client_id,
                "client_secret": client_secret,
                "audience": audience,
            },
        )
        resp.raise_for_status()
        payload = resp.json()

    _cache.store(payload["access_token"], payload["expires_in"])
    return _cache.token
```

**Common mistake:** Storing the token as a module-level string without expiry tracking. On long-running services, a 24h token cached at startup will fail 24 hours later with a cryptic 401.

### 2.3 Device Authorization Grant (RFC 8628)

Designed for devices without a browser (smart TVs, CLI tools). The device polls for a token while the user completes login on a separate device.

```
Device:   POST /device/code   →  { device_code, user_code, verification_uri, interval }
Device:   shows user_code + verification_uri to user
User:     visits verification_uri, enters user_code, authenticates
Device:   polls POST /token with device_code every `interval` seconds
Auth:     returns "authorization_pending" until user grants access, then returns tokens
```

Relevant for the integration platform if you ever build a CLI tool for internal admin operations.

### 2.4 Refresh Token Grant

Access tokens are short-lived (typically 1 hour). Refresh tokens are long-lived credentials that let the client get new access tokens without re-authenticating the user.

```
POST /oauth/token
  grant_type=refresh_token
  &refresh_token=REFRESH_TOKEN
  &client_id=CLIENT_ID
  (client_secret if confidential client)
```

Refresh tokens are high-value credentials — losing one is similar to losing a password. Auth servers should implement **refresh token rotation**: each use invalidates the old refresh token and issues a new one.

### 2.5 Implicit Flow — Deprecated, Know Why

The Implicit flow returned tokens directly in the URL fragment (no authorization code step). It was designed for SPAs before CORS existed. It's deprecated in OAuth 2.1 because:
- Tokens appear in browser history and referrer headers
- No opportunity to bind tokens to the client via PKCE
- The "optimization" (fewer round trips) is minimal on modern networks

**Authorization Code + PKCE** is the correct replacement for SPAs.

---

## 3. OpenID Connect — Adding Authentication to OAuth 2.0

OIDC (OpenID Connect Core 1.0) is a thin identity layer on top of OAuth 2.0. It adds:
- The **ID Token** (a JWT containing user identity claims)
- The **`openid` scope** (signals "I want identity, not just authorization")
- The **UserInfo endpoint** (returns claims about the authenticated user)
- Standardized claims (`sub`, `iss`, `aud`, `exp`, `iat`, `email`, `name`)

### ID Token vs. Access Token

| | ID Token | Access Token |
|---|---|---|
| **Audience** (`aud`) | The client application | The resource server (API) |
| **Purpose** | Prove user identity to the *client* | Authorize calls to the *API* |
| **Format** | Always a JWT | JWT or opaque (provider-dependent) |
| **Should client validate?** | Yes — always | Only if the client is also the RS |
| **Should you send to your API?** | **No** | Yes, in Authorization header |

This is the most commonly confused pair. **Never send an ID token to your API.** The API is not the intended audience. Send the *access token*. The ID token is for the client to learn who the user is.

```python
import jwt  # pip install PyJWT

def decode_id_token(id_token: str, jwks_uri: str, audience: str) -> dict:
    """Validate and decode an OIDC ID token."""
    import httpx, json
    from jwt.algorithms import RSAAlgorithm

    # Fetch JWKS (in production: cache this)
    jwks = httpx.get(jwks_uri).json()
    header = jwt.get_unverified_header(id_token)
    kid = header["kid"]

    # Find the matching key
    for key_data in jwks["keys"]:
        if key_data["kid"] == kid:
            public_key = RSAAlgorithm.from_jwk(json.dumps(key_data))
            break
    else:
        raise ValueError(f"No key found for kid={kid}")

    return jwt.decode(
        id_token,
        public_key,
        algorithms=["RS256"],
        audience=audience,
    )
```

### The Discovery Document

OIDC providers expose a `/.well-known/openid-configuration` endpoint that returns all the URLs you need:

```python
import httpx

async def get_oidc_config(issuer: str) -> dict:
    url = f"{issuer}/.well-known/openid-configuration"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
        resp.raise_for_status()
        return resp.json()

# Returns: authorization_endpoint, token_endpoint, jwks_uri,
#          userinfo_endpoint, introspection_endpoint, ...
```

**Common mistake:** Hardcoding token endpoint URLs instead of reading them from the discovery document. Provider URLs can change during migrations.

---

## 4. Scopes and Claims in Depth

**Scopes** are requested by the client at authorization time. They represent what the client wants to do. They're strings — conventions vary.

```
openid         → request an ID token (OIDC)
profile        → name, picture, locale claims
email          → email, email_verified claims
offline_access → request a refresh token
read:patients  → custom scope for your API
write:policies → custom scope for your API
```

**Claims** are key-value pairs inside a JWT. Standard claims:

| Claim | Meaning |
|-------|---------|
| `sub` | Subject — unique user identifier (stable, use as primary key) |
| `iss` | Issuer — the Authorization Server URL |
| `aud` | Audience — who the token is intended for |
| `exp` | Expiry — Unix timestamp; token invalid after this |
| `iat` | Issued at — Unix timestamp |
| `nbf` | Not before — token not valid before this |
| `jti` | JWT ID — unique token identifier (useful for revocation) |
| `email` | User's email (from OIDC `email` scope) |
| `name` | Display name (from OIDC `profile` scope) |

You can add **custom claims** to tokens in Auth0 via Actions. In Auth0, custom claims on access tokens must be namespaced (a full URL):

```javascript
// Auth0 Action (Node.js runtime)
exports.onExecutePostLogin = async (event, api) => {
  const namespace = 'https://api.crm-platform.internal/';
  api.accessToken.setCustomClaim(`${namespace}roles`, event.user.app_metadata.roles);
  api.accessToken.setCustomClaim(`${namespace}org_id`, event.user.app_metadata.org_id);
};
```

Then in your FastAPI middleware:

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)) -> dict:
    token = credentials.credentials
    try:
        payload = decode_access_token(token)  # your JWT validation function
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

    roles = payload.get("https://api.crm-platform.internal/roles", [])
    return {"sub": payload["sub"], "roles": roles}
```

---

## 5. Token Introspection (RFC 7662)

Introspection lets a Resource Server ask the Authorization Server: "is this token still valid?"

```
POST /oauth/introspect
Authorization: Basic <RS_CLIENT_ID:RS_CLIENT_SECRET>
Content-Type: application/x-www-form-urlencoded

token=ACCESS_TOKEN

Response:
{
  "active": true,
  "scope": "read:policies",
  "client_id": "...",
  "username": "hussain.sajib@example.com",
  "exp": 1750000000
}
```

**When to use introspection vs. local JWT validation:**

| | Local JWT Validation | Introspection |
|---|---|---|
| **Performance** | Fast — no network call | Slow — one HTTP round-trip per request |
| **Revocation** | Doesn't detect revoked tokens until expiry | Knows immediately if token is revoked |
| **Offline support** | Works; just need JWKS cached | Requires network to Auth Server |
| **Use case** | Most APIs; tokens are short-lived | When revocation is critical (security events) |

For the integration platform middleware, local JWT validation is usually correct — tokens expire in 1 hour, and the security window for a revoked token is acceptable. Use introspection only if you need immediate revocation (e.g., an admin-triggered session termination feature).

```python
import httpx
from functools import lru_cache

async def introspect_token(token: str, introspection_url: str, rs_client_id: str, rs_client_secret: str) -> dict:
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            introspection_url,
            data={"token": token},
            auth=(rs_client_id, rs_client_secret),
        )
        resp.raise_for_status()
        return resp.json()

# Usage in a FastAPI dependency:
async def require_active_token(token: str) -> dict:
    result = await introspect_token(token, ...)
    if not result.get("active"):
        raise HTTPException(status_code=401, detail="Token is not active")
    return result
```

---

## 6. Refresh Flows and Token Lifetime Strategy

Designing token lifetimes is a security and UX trade-off:

```
┌─────────────────────────────────────────────────────────┐
│               Token Lifetime Trade-offs                 │
│                                                         │
│  Short access token (5–15 min)                          │
│    PRO: Stolen token has minimal blast radius           │
│    CON: More refresh calls, higher load on Auth Server  │
│                                                         │
│  Long access token (1–24 hours)                         │
│    PRO: Fewer refreshes, better performance             │
│    CON: Revoked tokens stay valid until expiry          │
│                                                         │
│  Short refresh token (1–7 days)                         │
│    PRO: Forces re-authentication, limits long-term risk │
│    CON: Users get logged out more often                 │
│                                                         │
│  Long refresh token + rotation (30–90 days)             │
│    PRO: Good UX, rotation detects theft                 │
│    CON: Complex implementation, rotation race conditions│
└─────────────────────────────────────────────────────────┘
```

**Refresh token rotation** is the modern best practice: when you use a refresh token, you get a *new* refresh token back and the old one is invalidated. If an attacker steals the refresh token and uses it, the next time your legitimate client tries to refresh, it'll get a "refresh token already used" error — which Auth0 treats as a token theft signal and can revoke the entire session.

```python
async def refresh_access_token(
    token_url: str,
    client_id: str,
    refresh_token: str,
    client_secret: str | None = None,
) -> dict:
    data = {
        "grant_type": "refresh_token",
        "client_id": client_id,
        "refresh_token": refresh_token,
    }
    if client_secret:
        data["client_secret"] = client_secret

    async with httpx.AsyncClient() as client:
        resp = await client.post(token_url, data=data)
        resp.raise_for_status()
        return resp.json()  # new access_token, possibly new refresh_token
```

---

## 7. Token Storage Security

Where you store tokens is as important as how you issue them.

### Browser / SPA Context

| Storage | XSS Risk | CSRF Risk | Notes |
|---------|----------|-----------|-------|
| `localStorage` | **High** — any script can read it | None — not auto-sent | **Avoid for tokens** |
| `sessionStorage` | **High** — same origin scripts | None | Slightly better; gone on tab close |
| **HttpOnly Cookie** | **None** — JS cannot read it | Moderate | Best practice; add `SameSite=Strict` or `Lax` |
| Memory (JS variable) | Low — lost on refresh | None | For short-lived access tokens; pair with HttpOnly refresh cookie |

The modern recommended pattern for SPAs: store the **access token in memory** (a JS variable or closure), store the **refresh token in an HttpOnly, SameSite=Strict cookie**. The access token is gone on page refresh but can be silently renewed via the refresh token. This gives you XSS resistance for the refresh token while keeping the access token short-lived.

### Server / API Context (Your Middleware)

For the `crm-middleware` calling downstream APIs:
- Store `client_id` in environment variables or Kubernetes Secrets
- Store `client_secret` in **Vault** (already in your stack) — never in code or plain env vars
- Keep the access token in memory (module-level cache) with expiry tracking — never write it to disk or logs
- Never log full token values — log the first 8 characters + `...` for correlation

```python
import logging

log = logging.getLogger(__name__)

def log_token_event(event: str, token: str) -> None:
    # Truncate for log safety
    preview = token[:8] + "..." if len(token) > 8 else "***"
    log.info("token_event", extra={"event": event, "token_preview": preview})
```

**Common mistake:** Logging the full Authorization header in request logs (e.g., in httpx middleware or FastAPI request loggers). One misconfigured log shipper and you've leaked all active tokens.

---

## 8. Provider Comparison: Auth0 vs. Okta vs. Cognito vs. Keycloak

| | Auth0 | Okta | AWS Cognito | Keycloak |
|---|---|---|---|---|
| **Hosting** | SaaS | SaaS | SaaS (AWS) | Self-hosted |
| **Dev experience** | Excellent — M2 console | Good — more enterprise-focused | Poor — verbose, AWS-specific | Good — powerful, complex UI |
| **Pricing model** | Per MAU | Per user | Per MAU, free tier generous | Open source (hosting cost only) |
| **Custom domain** | Yes (paid) | Yes | Yes | Yes |
| **M2M support** | First-class | First-class | Limited — use Cognito App Clients | First-class |
| **Actions / Rules** | Actions (Node.js) | Event Hooks + inline hooks | Lambda triggers | Providers/SPI (Java) |
| **SAML/SCIM** | Enterprise tier | Excellent — born in enterprise | Limited | Excellent |
| **Multi-tenancy** | Organizations feature | Built-in | User Pools per tenant | Realms |
| **Salesforce integration** | Connected App docs | Connected App docs | Manual | Manual |
| **Platform fit** | **Current choice** — Auth0 | Good alternative | Viable if AWS-only | Good if you need self-hosted |

**Keycloak** is the right choice if you have compliance requirements that prohibit SaaS token issuance (e.g., tokens can never leave your data center). It's Java-based and runs on Kubernetes — fully compatible with your GCP/K8s stack.

**Cognito** is easiest if your entire stack is AWS and you want minimal operational overhead. The developer experience is rough compared to Auth0 — token claims customization requires Lambda functions.

**Auth0 vs. Okta:** Auth0 (acquired by Okta in 2021) is developer-first; Okta is IT/enterprise-first. For a product that developers interact with frequently, Auth0 is the right call. For workforce identity (employee SSO), Okta has deeper integrations.

---

## 9. Implementing JWT Validation in FastAPI

A complete, production-ready JWT validation dependency:

```python
import json
import time
from typing import Annotated
from functools import lru_cache

import httpx
import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from jwt.algorithms import RSAAlgorithm
from pydantic import BaseModel

bearer_scheme = HTTPBearer()


class TokenPayload(BaseModel):
    sub: str
    iss: str
    aud: str | list[str]
    exp: int
    iat: int
    scope: str = ""


@lru_cache(maxsize=1)
def get_jwks(jwks_uri: str) -> dict:
    """Cache JWKS in memory. In production, refresh on 401."""
    return httpx.get(jwks_uri).json()


def validate_token(token: str, jwks_uri: str, audience: str, issuer: str) -> dict:
    jwks = get_jwks(jwks_uri)
    header = jwt.get_unverified_header(token)
    kid = header.get("kid")

    for key_data in jwks["keys"]:
        if key_data.get("kid") == kid:
            public_key = RSAAlgorithm.from_jwk(json.dumps(key_data))
            break
    else:
        # Refresh JWKS cache and retry once (key rotation)
        get_jwks.cache_clear()
        jwks = get_jwks(jwks_uri)
        for key_data in jwks["keys"]:
            if key_data.get("kid") == kid:
                public_key = RSAAlgorithm.from_jwk(json.dumps(key_data))
                break
        else:
            raise ValueError("Unknown signing key")

    return jwt.decode(
        token,
        public_key,
        algorithms=["RS256"],
        audience=audience,
        issuer=issuer,
        options={"verify_exp": True},
    )


def require_auth(
    credentials: Annotated[HTTPAuthorizationCredentials, Depends(bearer_scheme)],
    settings = Depends(get_settings),  # your app settings
) -> dict:
    try:
        payload = validate_token(
            credentials.credentials,
            jwks_uri=settings.AUTH0_JWKS_URI,
            audience=settings.AUTH0_AUDIENCE,
            issuer=settings.AUTH0_ISSUER,
        )
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token expired")
    except jwt.InvalidTokenError as e:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail=str(e))
    return payload


# Usage in a route:
@app.get("/api/v2/policies")
async def list_policies(user: Annotated[dict, Depends(require_auth)]):
    ...
```

**Key details in this implementation:**
- **JWKS caching with rotation handling:** The `lru_cache` avoids fetching JWKS on every request. When a kid isn't found (key rotation), it clears the cache and retries once.
- **`verify_exp=True`:** This is the default in PyJWT but explicit is safer — a library upgrade won't silently disable expiry checking.
- **Separate error handling for expiry vs. invalid:** Lets you return more specific error messages and log the right metrics.

---

## 10. Key Concepts Summary

```
OAuth 2.0 & OIDC Mental Model
══════════════════════════════

OAuth 2.0 (Authorization Framework)
├── Roles
│   ├── Resource Owner (user)
│   ├── Client (your app)
│   ├── Authorization Server (Auth0)
│   └── Resource Server (your API)
│
├── Grant Types
│   ├── Authorization Code + PKCE  ← humans, web/mobile apps
│   ├── Client Credentials          ← M2M, service accounts
│   ├── Device Authorization        ← CLI tools, IoT
│   └── Refresh Token               ← renew without re-auth
│   (Implicit: deprecated)
│
└── Tokens
    ├── Access Token — bearer credential for APIs (short-lived)
    └── Refresh Token — renews access tokens (long-lived, rotated)

OpenID Connect (Identity Layer on top of OAuth 2.0)
├── Added by "openid" scope
├── ID Token (JWT) — who is the user? for the CLIENT
├── UserInfo endpoint — fetch more claims
└── Discovery document — /.well-known/openid-configuration

JWT Structure
├── Header: {"alg": "RS256", "kid": "key-id"}
├── Payload: {"sub": "...", "iss": "...", "aud": "...", "exp": ...}
└── Signature: RS256(base64(header) + "." + base64(payload), private_key)

Security Trade-offs
├── Local validation: fast, no revocation; use for most APIs
├── Introspection: slow, immediate revocation; use for high-security flows
├── Store access tokens in memory (SPAs) or in-memory cache (servers)
└── Store refresh tokens in HttpOnly cookies (SPAs) or Vault (servers)
```

---

## Quiz — 20 Questions

Test your understanding. Try to answer without looking back, then check the answers below.

---

### Questions

**1.** OAuth 2.0 is described as an *authorization* framework, not an *authentication* protocol. What specific thing does it authorize, and what question does it *not* answer?

**2.** What are the four roles defined in OAuth 2.0 RFC 6749? Give an example of each from the integration platform stack.

**3.** In Authorization Code + PKCE, what is the `code_verifier`, what is the `code_challenge`, and why does PKCE exist?

**4.** What is the `state` parameter in the Authorization Code flow, and what attack does it prevent?

**5.** Why was the Implicit grant deprecated in OAuth 2.1? What should you use instead?

**6.** When would you use the Client Credentials grant in your integration platform middleware? Describe a concrete example.

**7.** What is the difference between an ID Token and an access token? Which one should you send in the `Authorization` header when calling `crm-middleware`?

**8.** What HTTP endpoint does OIDC add to enable discovery of an Authorization Server's capabilities?

**9.** A JWT has three parts. What is in each part, and how are they signed?

**10.** You receive an access token with `"exp": 1750000000`. It's currently Unix time 1750003600. Is the token valid? What should your middleware return?

**11.** What is token introspection (RFC 7662), and when would you choose it over local JWT validation?

**12.** Your FastAPI service caches a client credentials token in memory without tracking expiry. What goes wrong after 24 hours in production?

**13.** Explain refresh token rotation. What security property does it provide?

**14.** A developer on your team stores the access token in `localStorage` in a browser SPA. What specific attack does this enable, and what's the safer alternative?

**15.** What is the difference between `scope` and `claims` in OAuth 2.0 / OIDC?

**16.** Your Auth0 action adds a custom claim `org_id` to the access token without a namespace (just `"org_id"`). What problem does this cause?

**17.** Compare Auth0 and Keycloak on two dimensions most relevant to a team with on-premises compliance requirements.

**18.** In your JWKS validation code, why do you need to handle the case where a `kid` is not found in the cached JWKS — and what's the correct recovery action?

**19.** What does the `aud` claim in a JWT represent, and what happens in PyJWT if you call `jwt.decode()` with the wrong audience value?

**20.** You're designing the auth layer for a new internal tool: a Python CLI that engineers run to trigger data migrations on `crm-middleware`. The CLI has no browser. Which OAuth 2.0 flow would you use, and why?

---

### Answers

??? note "Reveal Answers"

    **1.** OAuth 2.0 authorizes a *client application* to access a *resource on behalf of a resource owner* (user). It answers "does this client have permission to access this resource?" It does *not* answer "who is this user?" — that's OIDC's job. You can complete an OAuth 2.0 flow and still not know the identity of the authorizing user.

    **2.** Resource Owner: the human user or machine granting access (e.g., an integration platform admin user authorizing Salesforce access). Client: the application requesting access (e.g., `crm-middleware`, a Salesforce connected app). Authorization Server: issues tokens and authenticates the resource owner (e.g., Auth0 tenant). Resource Server: the protected API (e.g., `crm-middleware`'s `/api/v2/` endpoints, or the EHR system BDE API).

    **3.** The `code_verifier` is a cryptographically random string generated by the client before the authorization request. The `code_challenge` is `BASE64URL(SHA256(code_verifier))` and is sent to the authorization server. PKCE exists because public clients (SPAs, mobile apps) can't keep a `client_secret` confidential — if an attacker intercepts the authorization code, they can't exchange it without knowing the original `code_verifier`, which never left the client.

    **4.** The `state` parameter is an opaque random value generated by the client before redirecting to the authorization server. After the redirect back, the client verifies the `state` matches what it generated. This prevents CSRF attacks: a malicious site can't trick your callback into processing a fraudulent authorization code because they can't predict or control the `state` value.

    **5.** Implicit flow was deprecated because it returned tokens in the URL fragment, which appeared in browser history, server access logs, and `Referer` headers — all uncontrolled surfaces. There was also no mechanism to bind the token to the client (no `code_verifier`). The replacement is Authorization Code + PKCE, which keeps tokens out of the URL entirely and adds cryptographic client binding.

    **6.** Client Credentials is appropriate for any service-to-service call with no human in the loop. Concrete integration platform examples: (a) `crm-middleware` authenticating to the EHR system BDE API — the middleware is a "machine" requesting access to EHR system resources; (b) the ETL pipeline authenticating to the middleware's admin endpoints. In both cases, you present `client_id` + `client_secret` directly to the Auth Server and receive an access token, with no user redirect involved.

    **7.** An ID Token is a JWT issued by the Authorization Server to the *client* proving user identity — it's for the application to learn who logged in. An access token is a credential issued to the client to present to the *Resource Server* (your API) to prove authorization. You should send the **access token** in the `Authorization: Bearer` header when calling `crm-middleware`. Sending an ID token to an API is wrong because the API isn't the intended audience of that token — it may have different validation requirements and it leaks user PII unnecessarily.

    **8.** OIDC providers expose `/.well-known/openid-configuration` (the Discovery Document). This JSON endpoint contains the `authorization_endpoint`, `token_endpoint`, `jwks_uri`, `userinfo_endpoint`, `introspection_endpoint`, and all supported scopes, claims, and algorithms. You should read this at startup rather than hardcoding individual URLs so your code survives provider endpoint changes.

    **9.** A JWT has three Base64URL-encoded parts separated by dots. The **Header** contains `{"alg": "RS256", "typ": "JWT", "kid": "key-id"}` — the signing algorithm and key identifier. The **Payload** contains the claims: `sub`, `iss`, `aud`, `exp`, `iat`, and any custom claims. The **Signature** is computed as `RS256(BASE64URL(header) + "." + BASE64URL(payload), private_key)` — the Authorization Server signs it with its private key, and Resource Servers verify with the matching public key from JWKS.

    **10.** The token is expired. `1750003600 > 1750000000`, meaning the current time is 3,600 seconds (1 hour) past the `exp` timestamp. Your middleware should return `HTTP 401 Unauthorized` with a `WWW-Authenticate: Bearer error="invalid_token", error_description="Token has expired"` header. You should not attempt to use the token or introspect it.

    **11.** Token introspection (RFC 7662) lets a Resource Server ask the Authorization Server whether a token is currently active by calling `POST /introspect` with the token. It provides immediate knowledge of revocation — if an admin invalidates a session, the next introspection call returns `{"active": false}`. You'd choose introspection over local JWT validation when revocation must take effect immediately (e.g., a "log out everywhere" feature or when a user's account is suspended). The cost is one HTTP round-trip per authenticated request.

    **12.** The service fetches a token at startup and caches it in a plain string with no expiry tracking. After 24 hours (or whatever the token's `expires_in`), the token is expired, and all downstream API calls start failing with 401. Because there's no expiry check, the service never knows to refresh. In production this typically manifests as a service working fine overnight and then all requests failing with cryptic auth errors the next morning. Fix: store both the token and its expiry timestamp; check before each use.

    **13.** Refresh token rotation means every time a refresh token is used to get a new access token, the Authorization Server issues a *new* refresh token and immediately invalidates the old one. If an attacker steals the refresh token and uses it, the legitimate client's next refresh attempt will fail (token already used), which Auth0 can detect as a theft signal and revoke the entire session. Rotation limits the window during which a stolen refresh token is useful.

    **14.** Storing tokens in `localStorage` enables XSS (Cross-Site Scripting) attacks: any JavaScript injected into the page (via a vulnerable dependency, CDN compromise, or input injection) can call `localStorage.getItem('access_token')` and exfiltrate the token. The safer alternative is storing access tokens in a JavaScript variable in memory (they're lost on page refresh but kept for the session) and storing refresh tokens in an `HttpOnly` cookie, which JavaScript cannot access at all.

    **15.** **Scopes** are requested by the client during authorization — they're coarse-grained permissions indicating what access is needed (`read:policies`, `openid`, `offline_access`). The Authorization Server may grant all or a subset. **Claims** are the actual key-value pairs *inside* the resulting token: `sub`, `email`, `roles`, `org_id`. Scopes determine *which claims* appear in the token and *what the token allows the client to do* on the Resource Server.

    **16.** OIDC and the Auth0 documentation require custom claims on access tokens to use a namespace (a URL you control, like `https://api.crm-platform.internal/`). Without a namespace, a claim like `"org_id"` could conflict with a future standard claim or another vendor's claim. Auth0 actually silently drops non-namespaced custom claims from access tokens — so the claim will simply not appear in the token, and any code reading `payload.get("org_id")` will get `None`. Always namespace custom claims.

    **17.** **Compliance/data residency:** Keycloak is self-hosted — tokens are issued and stored entirely within your infrastructure, satisfying regulations that prohibit sending authentication traffic to external SaaS providers. Auth0 tokens are issued from Okta's cloud infrastructure. **Operational overhead:** Auth0 requires zero infrastructure management; Keycloak requires you to run, scale, backup, and upgrade a Java application (typically on Kubernetes). For teams with strict on-premises requirements, Keycloak's operational burden is the trade-off they accept for compliance.

    **18.** The JWKS endpoint returns the *current* set of signing keys. Authorization Servers rotate signing keys periodically (key rotation) — old tokens signed with the previous key are still valid until they expire, but new tokens use the new key. If a `kid` in an incoming token isn't in your cached JWKS, it means your cache is stale (the key was rotated after you last fetched). The correct recovery is: clear the JWKS cache, re-fetch from the `jwks_uri`, and retry validation once. If the `kid` still doesn't exist after the retry, the token is genuinely invalid.

    **19.** The `aud` (audience) claim specifies the intended recipient(s) of the token — typically the API's identifier (e.g., `"https://api.crm-platform.internal"`). When you call `jwt.decode()` in PyJWT with an `audience` parameter, it validates that the token's `aud` claim matches. If it doesn't match, PyJWT raises `jwt.InvalidAudienceError` — a subclass of `jwt.InvalidTokenError`. This is a critical validation: without it, a token issued for a different API (but signed by the same Auth Server) could be accepted by your API.

    **20.** Use the **Device Authorization Grant** (RFC 8628). The CLI has no embedded browser, so it can't redirect the user to a browser-hosted login page. With Device Authorization, the CLI displays a URL and a user code, the engineer opens that URL in their own browser and authenticates with Auth0, and the CLI polls the token endpoint until the user completes login. This is preferable to Client Credentials because it authenticates the *individual engineer* (audit trail per person) rather than issuing a shared service credential — important for an admin tool triggering data migrations.
