# Darkly — README: SQL Injection Leading to Credential Disclosure

## Summary
This document describes an SQL Injection vulnerability affecting the user search functionality. By injecting crafted SQL fragments into the search parameter, it is possible to enumerate database structure, extract sensitive fields, recover hashed credentials, and derive the flag.

This write‑up explains how the flaw works, why it exists, and how to remediate it properly.

---

## Vulnerability overview
- **Type:** SQL Injection (Union-based)
- **Affected feature:** User search form
- **Attack vector:** Unsanitized input incorporated directly into an SQL query
- **Impact:** Full read access to database tables; credential disclosure

---

## Proof of concept (high level)
1. The search bar allows direct SQL injection. Entering:
   ```
   1 UNION SELECT table_name, column_name FROM information_schema.columns
   ```
   forces the application to merge attacker-controlled results with legitimate query output.

2. This exposes the database schema, including the presence of a table named `users` and its columns such as `countersign` and `Commentaire`.

3. Using that knowledge, the following payload extracts sensitive data from the `users` table:
   ```
   1 UNION SELECT countersign, Commentaire FROM users
   ```

4. The application then displays:
   ```
   First name: 5ff9d0165b4f92b14994e5c685cdce28
   Surname: Decrypt this password -> then lower all the char. Sh256 on it and it's good !
   ```

5. The hash (`5ff9d0165b4f92b14994e5c685cdce28`) corresponds to the plaintext `FortyTwo`.

6. Converting the recovered password to lowercase and hashing it with SHA‑256 yields the final flag:
   ```
   10a16d834f9b1e4068b25c4c46fe0284e99e44dceaf08098fc83925ba6310ff5
   ```

This confirms that user input is concatenated directly into SQL queries without parameterization.

---

## Root cause analysis
1. **Dynamic SQL construction using user input.** The application likely builds queries such as:
   ```sql
   SELECT first_name, last_name FROM users WHERE id = '<user_input>';
   ```
   Without escaping or parameter binding, injected SQL fragments execute as part of the query.

2. **Lack of server‑side validation.** Inputs are not validated, sanitized, or restricted to numeric values.

3. **Use of `UNION` without restrictions.** Database responses are directly rendered to the page, enabling data exfiltration.

4. **Exposure of internal schema.** The database server allows unrestricted access to the `information_schema` meta‑tables.

5. **Weak separation between database privileges.** The application’s database user likely has broad `SELECT` permissions across tables.

6. **Lack of output filtering.** Displaying database fields directly in the UI makes data extraction straightforward.

---

## Impact
- **Extraction of database schema.** Attacker learns table structures, aiding deeper exploitation.
- **Credential disclosure.** Sensitive fields including password hashes can be retrieved.
- **Offline password cracking.** Hashes can be brute-forced or cracked if using weak algorithms.
- **Privilege escalation.** Compromised credentials may grant administrative access or enable further attack chains.
- **Full data leakage.** SQL injection enables reading any data the database user has access to.

Severity classification: **Critical**, due to full database compromise.

---

## Remediation (recommended fixes)

### 1. Use prepared statements / parameterized queries
Replace dynamic SQL building with secure parameter binding. Example pattern:
```
SELECT * FROM users WHERE id = ?
```
This eliminates the attack surface entirely.

### 2. Enforce strict input validation
- Restrict search parameters to expected types (e.g., numeric IDs).
- Reject or sanitize malformed input before reaching the database layer.

### 3. Implement least‑privilege SQL permissions
- The web application’s DB user should not have access to system schemas or unrelated tables.
- Restrict access to only the fields required for normal operation.

### 4. Hide internal database errors
- Avoid returning raw SQL output or failure messages to the client.
- Use generic error messages.

### 5. Remove sensitive hints from database fields
- Storing guide‑text such as “Decrypt this password...” exposes internal logic.
- Replace with proper authentication handling.

### 6. Strengthen password storage
- Replace weak hashing with modern, slow KDFs such as Argon2, bcrypt, or PBKDF2.
- Ensure passwords include salts to prevent rainbow‑table attacks.

### 7. Add monitoring and rate‑limiting
- Detect repeated failed queries.
- Alert on suspicious patterns consistent with enumeration.

---

## Detection and verification
- **Before remediation:** Inject test payloads containing `'` or `UNION SELECT` to confirm vulnerability.
- **After remediation:**
  - The application must reject malformed input.
  - `UNION` payloads must no longer execute.
  - Database errors must not be visible.
  - Schema enumeration (`information_schema`) must fail.

---

## Fix validation checklist
- [ ] All queries use parameterized statements.
- [ ] Numeric fields validated server‑side.
- [ ] Database permissions restricted.
- [ ] No raw database output is shown to users.
- [ ] Password storage upgraded to secure hashing.
- [ ] Tests confirm SQL injection no longer possible.

---

## Recommendations for further hardening
1. Perform a full audit for all dynamic database queries.
2. Implement Web Application Firewall (WAF) rules that detect SQL injection patterns.
3. Conduct periodic SQL injection testing using automated tools.
4. Apply secure software development practices and code reviews.

---

