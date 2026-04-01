# Admin Takeover via Broken Authorization (McDonald's case)

## 👑 From Low Privilege to Full Admin Control

### Overview

During testing of an employee management system, I identified multiple vulnerabilities that could be chained to achieve full administrative access. The issue involved weak authentication, user enumeration, and improper authorization controls.

---

### Recon & Discovery

* Observed inconsistent login behavior based on Employee ID
* Identified predictable numeric IDs
* Intercepted authentication requests using **Burp Suite**

---

### Vulnerabilities Identified

#### 1. User Enumeration

Different responses for valid and invalid IDs allowed identification of active accounts.

---

#### 2. Weak Authentication

* No rate limiting
* Weak password accepted
* Successful brute-force attack

---

#### 3. Broken Authorization (Privilege Escalation)

User roles were stored client-side and trusted without validation.

Original:

```json id="tqyk1f"
"permission": ["VIEW_ONLY"]
```

Modified:

```json id="y9rkl7"
"permission": ["VIEW_EDIT"]
```

The server accepted the modified role without verification.

---

### Exploitation Flow

1. Enumerated valid user IDs
2. Brute-forced credentials
3. Logged into a low-privileged account
4. Modified role in request
5. Gained administrative privileges

---

### Impact

* Full system compromise
* Unauthorized access to internal features
* Ability to modify user roles and data
* Potential abuse of internal tools and data

---

### Fix Recommendations

* Enforce strict server-side authorization
* Do not trust client-side role data
* Implement rate limiting
* Enforce strong password policies
* Add monitoring for privilege changes
