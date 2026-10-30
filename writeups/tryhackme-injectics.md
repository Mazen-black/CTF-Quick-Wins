# TryHackMe — Injectics CTF Write-up

> **Type:** Web application penetration testing  
> **Target:** Injectics  
> **Platform:** TryHackMe  
> **Scope:** Authorized training lab only  
> **Status:** Solved — both challenge flags retrieved

## Executive summary

Injectics was solved by chaining information disclosure, two SQL injection issues, an automated database-recovery mechanism, server-side template injection (SSTI), and remote command execution (RCE).

The initial foothold came from an exposed `mail.log` file. It disclosed developer and Super Admin credentials and described an automated service that repopulated the `users` table with default accounts. After obtaining Developer access through a login SQL injection, a second SQL injection in leaderboard editing deleted that table. The recovery service restored it, allowing the disclosed Super Admin account to be used.

After retrieving the first flag, testing the profile first-name field confirmed SSTI. Twig-related callback behavior then demonstrated command execution as `www-data`, which was sufficient to enumerate the application directory and read the second flag.

## 1. Reconnaissance and information disclosure

The application exposed several useful resources:

- `login.php`
- `adminLogin007.php`
- `script.js`
- `mail.log`
- `composer.json`
- `composer.lock`
- `edit_leaderboard.php`
- `update_profile.php`

The most useful discovery was `mail.log`, which contained an internal message describing **InjecticsService**. The message stated that the service monitored the database and restored default credentials if the `users` table was deleted or corrupted. It was scheduled to run every minute.

The log also disclosed the default Developer and Super Admin credentials. Those credentials were used only within this authorized lab.

**Key takeaway:** exposed logs can reveal both credentials and operational details that shape an attack path.

## 2. Initial access — login SQL injection

The login functionality was vulnerable to SQL injection. The following input was used during the lab:

```sql
'||1=1#;
```

This manipulated the login condition and bypassed the normal authentication check, providing Developer-level access.

**Finding:** SQL injection in authentication logic.

**Impact:** authentication bypass and unauthorized application access.

## 3. Leaderboard SQL injection — deleting the users table

After logging in as the Developer, the leaderboard editing functionality in `edit_leaderboard.php` was examined. The rank input was vulnerable to SQL injection.

The payload used was:

```sql
3;DROP TABLE users;--
```

### Why this payload worked in the lab

- `3` supplied the expected numeric rank value in the original query context.
- `;` terminated the first SQL statement.
- `DROP TABLE users;` issued a second statement that deleted the users table.
- `--` commented out any remaining SQL text in the constructed query (comment syntax and whitespace requirements vary by database).

The important condition is **stacked-query execution**: the database driver and application code must allow more than one SQL statement to be executed from the same request. A semicolon by itself does not guarantee that a second statement will run. The field's numeric context also helps explain why this payload could work where quote-based payloads did not.

The lab's observed behavior confirmed that the `users` table was deleted. InjecticsService then restored the table and its default accounts.

> This is destructive SQL. The payload is documented as evidence from the isolated TryHackMe lab and must never be tested against systems without explicit authorization.

**Finding:** SQL injection in leaderboard editing.

**Impact:** unauthorized database modification and deletion of the application's user table.

## 4. Automated recovery and Super Admin access

The recovery behavior described in `mail.log` was used as part of the attack chain:

1. Delete the `users` table through the vulnerable leaderboard input.
2. Allow the scheduled recovery service to run.
3. The service recreates or repopulates the table with the disclosed default accounts.
4. Authenticate through the administrator login page using the restored Super Admin account.

This resulted in administrative access and retrieval of the first flag.

## 5. SSTI detection and template identification

The profile update functionality was tested, specifically the **First Name** field.

Detection payload:

```twig
{{7*7}}
```

Observed response:

```text
Welcome, 49
```

The server evaluated the expression rather than displaying it literally, confirming server-side template injection.

The expression below was then used as an additional template-engine clue:

```twig
{{_self}}
```

The response contained a value resembling `__string_template__...`. Together with the Composer dependency declaration, this supported identifying Twig as the template engine. The application's Composer configuration declared Twig `2.14.0`; the installed runtime version should be verified independently using `composer.lock` or installed package metadata.

## 6. Remote command execution

The following proof of concept demonstrated that the Twig sort filter could invoke PHP's `passthru` function in the lab:

```twig
{{ ['id','']|sort('passthru') }}
```

Observed output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirmed command execution in the context of the web-server account, `www-data` — **not root**.

### CVE note

CVE-2022-23614 is relevant to the observed Twig sort-callback behavior and the affected-version discussion. However, the assessment did not independently verify whether Twig Sandbox mode was enabled or confirm the installed runtime version. Therefore, RCE was demonstrated, but attribution specifically to this CVE remains unverified.

## 7. Post-exploitation enumeration and second flag

The command-execution context was used for limited filesystem enumeration:

```bash
pwd
ls
ls -la /var/www/html/flags
```

The working directory was:

```text
/var/www/html
```

A flag file was found under `/var/www/html/flags` and read from the lab environment.

The assessment did not demonstrate root-level privileges, persistence, or access to unrelated systems.

## 8. Attack-chain summary

| Stage | Action | Result |
|---|---|---|
| 1 | Application reconnaissance | Identified exposed application resources |
| 2 | Read `mail.log` | Obtained credentials and recovery-service intelligence |
| 3 | Login SQL injection | Bypassed authentication and obtained Developer access |
| 4 | Developer account access | Reached leaderboard functionality |
| 5 | Leaderboard SQL injection | Deleted the `users` table |
| 6 | Automated recovery | Restored default user accounts |
| 7 | Super Admin login | Obtained administrative access |
| 8 | Admin dashboard | Retrieved first flag |
| 9 | Profile update testing | Confirmed SSTI |
| 10 | Twig callback abuse | Achieved RCE through `passthru` |
| 11 | Filesystem enumeration | Confirmed `www-data` and located flag file |
| 12 | Flag retrieval | Retrieved second flag |

## 9. Findings and remediation

| Finding | Root cause | Recommended remediation |
|---|---|---|
| Information disclosure | Sensitive logs were publicly accessible | Keep logs outside the web root and enforce access controls |
| Credential exposure | Default credentials were stored in plaintext in an exposed log | Rotate exposed credentials; never store passwords in plaintext |
| Login SQL injection | User input was incorporated unsafely into SQL | Use parameterized queries and prepared statements |
| Leaderboard SQL injection | User-controlled rank data was processed unsafely | Parameterize all database queries; reject unexpected input server-side |
| Unsafe recovery design | Recovery restored predictable default credentials | Use secure, unique recovery secrets and tightly restrict recovery operations |
| SSTI | User input was evaluated as template source | Never compile or evaluate user-controlled input as template source |
| RCE | Unsafe template behavior exposed callable functionality | Upgrade and securely configure Twig; restrict callable functions |
| Excessive impact potential | Web application had filesystem access as `www-data` | Apply least-privilege OS and filesystem permissions |

Additional recommendations:

- Review Twig Sandbox configuration and callback restrictions.
- Confirm the exact installed Twig version from runtime package metadata before making a CVE determination.
- Store sensitive files outside the publicly accessible web root.
- Add regression tests for SQL injection, SSTI, authorization, and recovery behavior.

## 10. Flags

<details>
<summary>Show challenge flags (spoilers)</summary>

**Flag 1**

```text
THM{INJECTICS_ADMIN_PANEL_007}
```

**Flag 2**

```text
THM{5735172b6c147f4dd649872f73e0fdea}
```

</details>

## Lessons learned

- Read exposed files carefully: they may reveal architecture and operational behavior, not just credentials.
- Understand the SQL context before choosing a payload. Numeric input, statement separators, comment syntax, and driver behavior all matter.
- A successful SQL injection payload can have destructive impact; stacked queries are not supported in every database API.
- Confirm SSTI with a harmless expression before attempting to assess impact.
- Distinguish demonstrated facts from hypotheses. In this lab, RCE was observed, while the exact CVE attribution still required verification.
- Record the execution identity and actual scope of access; command execution as `www-data` is not root access.

---

*Educational write-up based on an authorized TryHackMe lab. All testing described here was performed in the challenge environment. Do not run destructive payloads or security tests against systems without explicit permission.*
