# PART 1 — Software Engineering
## Module 7: Authentication & Authorization

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Implement JWT-based authentication in FastAPI correctly, understanding
  precisely what a JWT does and doesn't guarantee.
- Design API-key-based authentication for programmatic/agent access — the
  dominant access pattern for AI systems (model provider APIs themselves
  use API keys, and your own AI services will too).
- Implement authorization (RBAC — role-based access control) cleanly,
  separated from authentication, using FastAPI dependencies (Module 3).
- Apply per-user/per-API-key rate limiting — an AI-specific authorization
  concern, since LLM calls are expensive and abuse/runaway usage is a real
  cost risk, not just a traffic-shaping nicety.
- Recognize and avoid the most common, most damaging authentication/
  authorization security mistakes.

### 2. Prerequisites
Modules 1–6, and Part 0 Module 6 (Redis, for rate limiting).

### 3. Estimated Study Time
10–12 hours over 5–6 days.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — you know auth conceptually from Spring Security; the
depth here is JWT internals, API-key patterns specific to AI/agent access,
and getting authorization cleanly separated from authentication.)

### 5. Why This Matters
Every AI SaaS product (Part 9/11) needs real authentication, and nearly
every AI service you build needs to support **API-key-based** access for
programmatic clients (other services, agents, scripts) in addition to or
instead of typical username/password human login. Getting rate limiting
tied to auth right is also a direct cost-control mechanism — unrestricted
API access to an LLM-backed endpoint is a genuine, realistic way to
accidentally run up a large bill.

---

### 6. Theory

**Authentication vs. Authorization — precisely, since these are often
conflated:**
**Authentication** answers "who are you?" (verifying identity).
**Authorization** answers "what are you allowed to do?" (given a verified
identity, checking permissions). Keep these as two clearly separate
concerns in your code — a common mistake is tangling identity verification
and permission checks into one function/middleware.

**JWT (JSON Web Tokens) — what they actually are and guarantee:**
A JWT is a signed (not necessarily encrypted) token: `header.payload.signature`,
base64-encoded. The signature (using a secret key, or a public/private key
pair) lets the server verify the payload hasn't been tampered with —
**but the payload itself is readable by anyone** (it's base64, not
encryption) unless you specifically encrypt it (JWE, less common). Never
put secrets in a JWT payload — only non-sensitive claims (user ID, roles,
expiry).

```python
import jwt

payload = {"sub": user_id, "roles": ["user"], "exp": expiry_timestamp}
token = jwt.encode(payload, secret_key, algorithm="HS256")

# Verifying (in a FastAPI dependency):
def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    try:
        payload = jwt.decode(token, secret_key, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(401, "Invalid token")
    return User(id=payload["sub"], roles=payload["roles"])
```
**Critical JWT precision points:**
- **Always set and check expiry (`exp`)** — a JWT with no expiry is a
  permanent credential if leaked.
- **Stateless by design** — the server doesn't need to look anything up
  to validate a JWT (just verify the signature), which is great for
  scalability but means **you can't easily revoke a single JWT before its
  expiry** without additional infrastructure (a token blocklist, or short
  expiries + refresh tokens). This is a genuine trade-off, not a flaw to
  "just fix" — short-lived access tokens (minutes) plus longer-lived
  refresh tokens (which *can* be revoked, since refresh is a stateful
  lookup) is the standard mitigation.

**API keys — the dominant pattern for AI/agent access:**
Unlike JWTs (typically for human user sessions with a login flow), API
keys are simpler, longer-lived credentials for programmatic access —
exactly how Anthropic's and OpenAI's own APIs authenticate you. Design
considerations:
```python
# Store a HASH of the API key, never the raw key (same principle as
# password hashing — if your DB leaks, raw keys leaking is catastrophic,
# hashed keys leaking is not immediately exploitable)
import hashlib
def hash_api_key(raw_key: str) -> str:
    return hashlib.sha256(raw_key.encode()).hexdigest()

# On issuance: show the raw key to the user EXACTLY ONCE, store only the hash
# On each request: hash the provided key, look up by hash, never by raw value
```
API keys should be **scoped** (which permissions/resources this specific
key can access — directly reusing Module 1's Interface Segregation
thinking, applied to permissions) and **revocable independently** (each
key has its own row/record that can be individually disabled without
affecting other keys for the same account).

**RBAC (Role-Based Access Control), cleanly separated via FastAPI
dependencies:**
```python
def require_role(required_role: str):
    def dependency(user: User = Depends(get_current_user)) -> User:
        if required_role not in user.roles:
            raise HTTPException(403, "Insufficient permissions")
        return user
    return dependency

@router.delete("/documents/{id}")
async def delete_document(
    id: str,
    user: User = Depends(require_role("admin")),
):
    ...
```
This keeps authentication (`get_current_user`, verifying identity) and
authorization (`require_role`, checking permission) as separate,
composable dependencies — directly applying Module 1's Single
Responsibility Principle to auth code specifically, a place where it's
especially easy (and especially costly) to let concerns blur together.

**Rate limiting tied to authentication — the AI-specific cost-control
concern:**
Reuse Part 0 Module 6's Redis-based rate limiter, but key it by **user ID
or API key**, not just IP address (IP-based limiting is easily
circumvented and doesn't map to your actual cost driver — a specific
authenticated user/key making expensive LLM calls). Design tiers
deliberately: a free tier might get 10 requests/day; a paid tier, 10,000 —
this is a genuine product/business decision, not just a technical default,
and it's how nearly every real AI API product (including Anthropic's own)
actually works.

---

### 7. Mental Models

**Model 1 — "A JWT is signed, not secret — never put sensitive data in
its payload, and always set a short expiry since you can't easily revoke
it early."**

**Model 2 — "Authentication answers 'who,' authorization answers 'what
are they allowed to do' — keep them as two separate, composable
dependencies, never one tangled function."**

**Model 3 — "Store hashed API keys, exactly like hashed passwords — the
raw key exists only at issuance time, shown once, never persisted."**

**Model 4 — "Rate limiting is a cost-control authorization concern in AI
systems, not just a traffic-shaping nicety — key it by authenticated
identity, and design tiers as a deliberate product decision."**

---

### 8. Visual Explanation (described)

**Diagram: "Auth vs. authz as separate dependency layers"**
```
Request with a Bearer token or API key
   |
   v
[ Authentication dependency: verify signature/hash, resolve to a User ]
   |  (answers: "who is this?")
   v
[ Authorization dependency: check User.roles / User.permissions against
  what this specific endpoint requires ]
   |  (answers: "are they allowed to do THIS specific thing?")
   v
[ Rate-limit dependency: check Redis counter keyed by User.id/api_key ]
   |  (answers: "have they exceeded their allowed usage?")
   v
Route handler executes
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **FastAPI official docs — "Security" section in full**, especially
   "OAuth2 with Password (and hashing), Bearer with JWT" — the
   authoritative, precise reference for the exact patterns above.
2. **jwt.io's "Introduction" page** — the clearest visual explanation of
   JWT structure (header/payload/signature) and what signing does and
   doesn't guarantee.
3. **OWASP API Security Top 10** (owasp.org) — read this as your security
   baseline for auth-adjacent mistakes; several of the top entries are
   directly authentication/authorization-related (broken object-level
   authorization, broken authentication).
4. **Stripe's API documentation on API keys** — as a concrete, well-
   designed real-world example of API-key scoping, rotation, and
   restriction patterns.

**Official documentation:** fastapi.tiangolo.com/tutorial/security/,
jwt.io, owasp.org/API-Security.

**GitHub repos:** `tiangolo/full-stack-fastapi-template` (revisit again —
its auth implementation is a clean reference), any well-regarded FastAPI
+ API-key-auth example repo.

---

### 17. Exercises

1. Implement JWT-based login (issue a token on valid credentials, verify
   it on protected endpoints) in `convo-api`, with a short access-token
   expiry and a separate, longer-lived, revocable refresh token.
2. Implement API-key issuance and verification: generate a raw key, hash
   it before storage, show it to the "user" exactly once, and verify
   incoming requests by hashing and looking up.
3. Build a `require_role` dependency and demonstrate that a user without
   the required role receives a 403, cleanly separated from the
   authentication dependency that resolves identity.
4. Implement per-API-key rate limiting (reusing Part 0 Module 6's Redis
   rate limiter, now keyed by API key instead of IP), with two
   configurable tiers.

### 18. Mini-Project
Add full JWT-based user authentication to `convo-api`, including
registration, login, protected endpoints via `get_current_user`, and a
`require_role`-based admin-only endpoint (e.g., deleting any user's
conversation, not just your own).

### 19. Production Project
**Build:** `auth-gateway` — extend `convo-api` with production-grade
auth, portfolio-quality:
- JWT-based authentication for human users (short-lived access token,
  revocable refresh token)
- API-key-based authentication for programmatic access, with hashed
  storage, scoped permissions, and independent revocation per key
- Clean separation of authentication and authorization dependencies
  throughout
- Per-identity (user or API key) rate limiting with at least two tiers,
  using Redis and reusing Module 4's observability to log/track rate-limit
  hits
- Full test suite covering: successful auth, expired/invalid tokens,
  insufficient-role rejection, rate-limit enforcement, and API key
  revocation actually taking effect immediately
- A security-focused README section explicitly walking through: how a
  leaked JWT is mitigated (short expiry), how a leaked API key is
  mitigated (hashed storage, independent revocation), and referencing
  specific OWASP API Security Top 10 items you've addressed

This becomes the auth layer every subsequent capstone project reuses —
and the security-focused README section is genuinely strong interview and
freelance-credibility material.

---

### 20–21. Practice & Interview Questions

1. Explain what a JWT's signature does and doesn't guarantee, and why you
   should never put sensitive data in its payload.
2. Why can't you easily revoke a single JWT before its expiry, and what's
   the standard mitigation?
3. Design an API key system: how do you store keys, how do you handle
   issuance/display, and how do you support independent revocation per
   key?
4. Explain the clean separation between authentication and authorization
   dependencies, and why conflating them is a design smell.
5. Why is per-user/per-API-key rate limiting especially important for
   AI-backed APIs specifically, compared to typical CRUD APIs, and how
   would you design tiered limits as a product decision?

---

### 22. Common Mistakes
- Storing raw API keys or passwords instead of hashed values.
- Putting sensitive data in a JWT payload, forgetting it's readable by
  anyone who has the token (signed, not encrypted).
- Issuing JWTs with no expiry, creating effectively permanent credentials
  if leaked.
- Conflating authentication and authorization into one function, making
  permission logic hard to reason about and easy to get subtly wrong.
- Rate limiting only by IP address for an authenticated API, missing the
  actual cost driver (a specific user/key) and being trivially
  circumvented.

### 23. Debugging Exercise
Given a reported incident where a revoked API key still worked for
several minutes after revocation, diagnose that the rate-limit/auth check
was reading from a cache with a stale TTL instead of checking the
authoritative revocation status on every request (or checking it too
infrequently) — fix by ensuring revocation status is checked with
appropriately low staleness tolerance, balancing correctness against the
performance cost of checking on every single request.

---

### 24. Checklist
- [ ] I can explain precisely what a JWT's signature guarantees and
      doesn't
- [ ] I store only hashed API keys, never raw values, past issuance time
- [ ] I keep authentication and authorization as clearly separate,
      composable dependencies
- [ ] I've implemented per-identity rate limiting with tiered limits
- [ ] I've completed `auth-gateway` with a security-focused README section

### 25. Summary
JWTs are signed but not encrypted — never put sensitive data in the
payload, and use short expiries with revocable refresh tokens to mitigate
the "can't easily revoke early" trade-off. API keys are the dominant
pattern for programmatic/agent access, stored hashed and independently
revocable, exactly like Anthropic's and OpenAI's own APIs. Rate limiting
tied to authenticated identity is a genuine cost-control mechanism for
AI-backed APIs, not just traffic shaping. `auth-gateway` becomes the auth
layer every later capstone project reuses.

### 26. Next Steps
Module 8: **CI/CD & GitHub Actions** — automating the testing, linting,
and deployment of everything built so far, turning your manual "run tests
locally" habit into a real, enforced pipeline — the direct prerequisite
for Part 8's production deployment work.

---

**Reply "continue" for Module 8, or flag anything to go deeper on.**
