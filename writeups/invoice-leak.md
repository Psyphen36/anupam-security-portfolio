# Unauthenticated Invoice Exposure (Taxify/Bolt)

## 💳 Public Access to Sensitive Financial Documents

### Overview

I identified an issue where customer invoices were accessible without authentication through a predictable parameter. The application failed to enforce access control on sensitive financial documents.

---

### Discovery

While exploring the invoice system, I came across URLs structured as:

```id="t0kpfi"
/?s=<uuid>
```

Accessing these links directly returned invoice PDFs.

---

### Vulnerability

No authentication or authorization was required to access invoices.

By modifying the UUID parameter, different users’ invoices could be accessed.

---

### Exploitation

1. Accessed invoice URL without login
2. Observed valid invoice data returned
3. Replaced UUID with another value
4. Retrieved different customer invoices

---

### Data Exposed

* Customer names
* Business details
* Trip data
* Payment amounts

---

### Impact

* Unauthorized access to financial records
* Privacy violations
* Exposure of sensitive customer data
* Potential regulatory risk (GDPR implications)

---

### Fix Recommendations

* Require authentication for invoice access
* Validate ownership of requested resource
* Use signed, time-limited URLs
* Monitor abnormal access patterns
