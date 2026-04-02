---
layout: writeup
title: "Admin Takeover via Broken Authorization — McDonald's"
description: "Chained user enumeration, brute force, and a client-side trust issue to escalate from zero access to full admin control on an internal employee management system."
category: web
difficulty: medium
event: "Bug Bounty"
status: oos
outcome_note: "Reported responsibly. Marked out of scope after triage — the system turned out to be a decommissioned third-party application no longer under the program's control. The vulnerability chain was real; the asset just wasn't theirs to fix."
date: 2025-03-10
---

## Background

This was one of those findings where no single vulnerability is catastrophic on its own — but chaining three weak spots together turns into a full system compromise. The target was an internal employee management portal. I won't share the specific subdomain, but it was in scope under McDonald's disclosure program.

The application managed employee records, shift data, and internal roles. Interesting enough to dig into.

---

## Recon

The first thing I noticed was the login form. It asked for an **Employee ID** (numeric) and a password. When I entered a valid ID with a wrong password, the error was:

> *"Incorrect password."*

When I entered an ID that didn't exist:

> *"Employee not found."*

That's a textbook user enumeration issue. The application was leaking whether an account existed based on the error message alone. From here, I could confirm valid IDs by iterating through a numeric range and watching for the error to change.

I wrote a quick Python script to enumerate:

```python
import requests

base_url = "https://[redacted]/login"

for emp_id in range(1000, 1200):
    r = requests.post(base_url, data={"id": emp_id, "password": "test"})
    if "Incorrect password" in r.text:
        print(f"[+] Valid ID found: {emp_id}")
```

Within a few minutes I had a list of active employee IDs.

---

## Gaining a Foothold

With valid IDs in hand, I turned to the password policy. The application had no rate limiting on the login endpoint — no CAPTCHA, no lockout after failed attempts, nothing. I ran Burp Intruder against one of the confirmed IDs with a small list of weak passwords.

It logged in on the third attempt. The password was a simple numeric string — likely a default credential set during account creation and never changed.

At this point I had access to a low-privileged employee account. It could view some shift data but most features were locked behind "admin" roles.

---

## Privilege Escalation

Here's where it got interesting. I intercepted an authenticated API request using Burp Suite and noticed the request body included a `permission` field:

```json
{
  "employeeId": "1042",
  "permission": ["VIEW_ONLY"]
}
```

The server was accepting role data from the client and trusting it without any server-side validation. I modified the request to:

```json
{
  "employeeId": "1042",
  "permission": ["VIEW_EDIT"]
}
```

Forwarded it — and the application responded as if I were an admin. Full access: employee records, role management, internal tooling, everything.

---

## Full Attack Chain

| Step | Action | Vulnerability |
|------|--------|---------------|
| 1 | Enumerate valid Employee IDs | User Enumeration via error messages |
| 2 | Brute-force credentials | No rate limiting |
| 3 | Log in as low-privilege user | Weak password policy |
| 4 | Modify `permission` field in request | Client-side role trust |
| 5 | Full admin access | Broken Authorization |

---

## Impact

Once escalated, I could:

- View and modify any employee record
- Reassign roles across the system
- Access internal tooling restricted to HR/management
- Potentially exfiltrate shift and payroll-adjacent data

This wasn't a theoretical risk. Each step was reproducible in minutes.

---

## Fix Recommendations

**Authorization should live on the server, not the client.** The most critical fix is straightforward: never accept role or permission data from the client. The server should determine what a user is allowed to do based on their authenticated session — not on a field they can freely edit in a request.

Beyond that:

- Implement rate limiting and account lockout on the login endpoint
- Use uniform error messages regardless of whether an account exists
- Enforce a minimum password complexity policy
- Log and alert on unusual privilege-adjacent API calls
