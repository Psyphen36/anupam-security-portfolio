# IDOR in Production API (Comcast case)

## 🔓 Breaking Access Control via ID Enumeration

### Overview

While testing a production API, I identified an Insecure Direct Object Reference (IDOR) that allowed unauthorized access to user data by modifying object identifiers. The issue stemmed from missing authorization checks on direct resource access.

---

### Recon & Discovery

During API testing using **Burp Suite**, I observed the endpoint:

```id="q3v6q2"
/api/v1/loggedInUsers
```

This returned details of the authenticated user, including:

* Name
* Email
* Job title
* Roles
* Company details

The response included a numeric user ID, which became the pivot point.

---

### Vulnerability

By modifying the endpoint to:

```id="s0n9pa"
/api/v1/{userId}
```

I was able to retrieve data belonging to other users.

No authorization checks were enforced to verify ownership of the requested resource.

---

### Exploitation

1. Captured authenticated request
2. Identified user ID in response
3. Replaced endpoint path with another ID
4. Received another user’s data

Repeating this allowed enumeration of multiple users.

---

### Impact

* Exposure of sensitive user information
* User enumeration due to predictable IDs
* Increased risk of targeted phishing and social engineering
* Potential mapping of internal organizational structure

---

### Fix Recommendations

* Enforce server-side authorization checks
* Ensure users can only access their own data
* Replace sequential IDs with UUIDs
* Implement rate limiting on API requests
