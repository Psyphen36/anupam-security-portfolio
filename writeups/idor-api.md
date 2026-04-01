---
layout: writeup
title: "IDOR in Production API — Comcast"
description: "A predictable numeric user ID exposed through an authenticated endpoint made it trivial to pull any user's account data — name, email, job title, roles, and company structure — with no authorization checks."
category: web
difficulty: easy
event: "Bug Bounty"
date: 2024-03-22
---

## How It Started

I was exploring a Comcast internal API during a recon pass — the kind where you're not looking for anything specific, just following the data. I authenticated normally and started mapping endpoints through Burp's proxy.

One endpoint caught my eye:

```
GET /api/v1/loggedInUsers
```

It returned my own account information: name, email, job title, assigned roles, and company/org details. Useful for an app, normal enough. But buried in the JSON response was a numeric `userId`:

```json
{
  "userId": 10482,
  "name": "...",
  "email": "...",
  "jobTitle": "...",
  "roles": ["..."],
  "company": "..."
}
```

Sequential numeric IDs in a production API are always worth a second look.

---

## The Vulnerability

I tried something simple. Instead of hitting `/api/v1/loggedInUsers`, I constructed a direct resource URL:

```
GET /api/v1/10483
```

The server responded with a `200 OK` and returned the full profile of another user — someone I had no business accessing. No authorization check. No ownership validation. Just a raw lookup by ID.

This is a classic **Insecure Direct Object Reference (IDOR)**. The application was treating the user ID as both an identifier *and* an access token, which it isn't.

---

## Exploitation

The steps were almost embarrassingly simple:

1. Log in normally and capture the authenticated session
2. Pull my own `userId` from the `/loggedInUsers` response
3. Increment or decrement the ID in the direct endpoint
4. Receive another user's full profile in response

Because the IDs were sequential integers, iterating through a range would expose every user in the system. I stopped after confirming the issue with two or three IDs — no need to pull a full dump to prove impact.

A basic loop would look like:

```python
import requests

session = requests.Session()
# ... authenticate here ...

for user_id in range(10000, 10020):
    r = session.get(f"https://[redacted]/api/v1/{user_id}")
    if r.status_code == 200:
        print(f"[+] {user_id}: {r.json().get('email')}")
```

---

## What Was Exposed

Each unauthorized profile response included:

- Full name and email address
- Job title and department
- Assigned roles within the system
- Company and organizational hierarchy details

For a large organization, that's not just a privacy issue — it's a social engineering gold mine. An attacker mapping internal org structure and email formats is already halfway through a phishing or pretexting campaign.

---

## Impact

| Risk | Detail |
|------|--------|
| PII exposure | Name, email, job title accessible for all users |
| Org mapping | Full company structure enumerable |
| Phishing risk | Email formats + roles = targeted social engineering |
| Compliance | Likely reportable under applicable data regulations |

---

## Fix Recommendations

The fix here is a one-liner in principle, but needs to be applied consistently across every endpoint that touches user resources.

**The server must verify that the requesting user owns — or has explicit permission to access — the requested resource.** The check should look something like:

```python
if requested_user_id != current_user.id and not current_user.is_admin:
    return 403
```

Beyond that:

- Replace sequential integer IDs with UUIDs or other non-guessable identifiers — this doesn't fix the authorization gap, but it removes the ability to enumerate
- Apply rate limiting on user-lookup endpoints
- Audit all API routes that accept object identifiers as path parameters
