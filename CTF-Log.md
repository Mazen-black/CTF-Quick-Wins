# CTF Log — Quick Wins

Short-form entries for boxes solved without a full write-up. Each one documents the exploitation chain and key technique, not every step.

---

## TryHackMe — Support (Web Application Vulnerabilities II — final box)

**Goal:** Pentest the Support Ops platform → achieve RCE
**Chain:** Username Guessing → Brute Force → Cookie/Role Tampering → Path Traversal → IDOR → Command Injection → RCE

### Summary

Started with full enumeration (`gobuster`, source code review) — standard wordlists didn't surface the admin path directly, but `gobuster` did flag a `config` file that returned no direct access.

Guessed the support account's username (`help@support.thm`) from the page branding, then used **Hydra** to brute-force the password successfully.

After logging in, found an `admin=false` flag in the session **cookie** (MD5-hashed). Flipped it to `true` via DevTools and refreshed — privilege escalated to an "IT" role.

The `config` file `gobuster` had found earlier wasn't directly reachable, but a **path traversal** (`../config`) got around that restriction and leaked the **admin password** — without a matching username.

Found the missing username via **parameter manipulation** on `/api.php?id=`: dropping the ID value returned the full user list instead of a single record, revealing the admin account.

Logged in as admin → **Flag 1 captured.**

Spotted a `sys` parameter in a request further down the application. Recognized it as a likely **Command Injection** point and used `;` as a command separator (confirmed the technique via a public write-up after an initial failed attempt) to run:
```
cat /ubuntu/user.flag
```
→ **RCE confirmed, final flag captured.**

### Key takeaway

No single vulnerability here was severe on its own — the real risk was **chaining** six separate weak points (recon leak → weak credentials → client-side trust → path traversal → IDOR → injection) into full remote code execution. Same lesson as Recruit: a proper review has to look at how findings combine, not just rate each one in isolation.
