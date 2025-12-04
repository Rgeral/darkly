# Darkly — README: SQL Injection in Image Search Function

## Summary
This document describes a second SQL Injection vulnerability affecting the image search functionality of the Darkly application. As with the previous SQLi flaw, unsanitized user input is incorporated into database queries. This allows attackers to enumerate database structure, extract arbitrary fields, recover sensitive values, and ultimately derive a flag.

The attack is similar in technique but targets a different table (`list_images`) and exposes a different credential-like value.

---

## Vulnerability overview
- **Type:** SQL Injection (Union-based)
- **Affected feature:** Image search bar
- **Attack vector:** Direct concatenation of user input into SQL queries
- **Impact:** Disclosure of image metadata, embedded hints, and credential-like hashes leading to flag retrieval

---

## Proof of concept (high level)

1. Inject the following payload into the image search field:
   ```
   1 UNION SELECT table_name, column_name FROM information_schema.columns
   ```
   This reveals database schema information, including the table `list_images` and its columns, such as `comment`.

2. Using this information, extract data from the target table:
   ```
   1 UNION SELECT comment, comment FROM list_images
   ```

3. The application displays:
   ```
   Title: If you read this just use this md5 decode lowercase then sha256 to win this flag ! : 1928e8083cf461a51303633093573c46
   ```

4. The MD5 hash `1928e8083cf461a51303633093573c46` corresponds to the plaintext `albatroz`.

5. Converting it to lowercase (already lowercase) and hashing with SHA-256 produces the flag:
   ```
   f2a29020ef3132e01dd61df97fd33ec8d7fcd1388cc9601e7db691d17d4d6188
   ```

This confirms full read access to table contents through SQL injection.

---

## Root cause analysis

1. **Dynamic SQL construction.** User-controlled input is inserted directly into a query used to fetch image data.

2. **Lack of prepared statements.** The backend does not use parameterized queries to separate code from data.

3. **Direct reflection of database values in the UI.** Combined with SQL Injection, this makes exfiltration trivial.

4. **Broader schema exposure.** `information_schema` is accessible, enabling attackers to map the database before querying specific tables.

5. **Weak or inappropriate data stored in the `comment` field.** Embedding hints and MD5 hashes in comments suggests the application mixes metadata with sensitive data.

---

## Impact
- **Arbitrary data extraction.** Any table or field the DB user can access is retrievable.
- **Hash disclosure.** Allows offline cracking and further access.
- **Chaining with other vulnerabilities.** Combined SQLi and weak hashing produce easy privilege escalation.
- **Loss of confidentiality.** Exposes internal application logic and data flows.

Severity classification: **Critical**, as full database read access is possible.

---

## Remediation (recommended fixes)

### 1. Use parameterized SQL queries
All database interactions must use prepared statements, e.g.:
```
SELECT title, comment FROM list_images WHERE id = ?
```
This prevents attacker-controlled fragments from altering the query structure.

### 2. Validate and sanitize input
- Restrict the search field to expected patterns (e.g. alphanumeric identifiers).
- Reject or neutralize SQL meta-characters.

### 3. Restrict database permissions
- The application's DB user should not read `information_schema`.
- Limit access to only the tables and fields needed.

### 4. Remove sensitive hints from metadata
Fields intended for comments or descriptions should never contain operational secrets or password-like values.

### 5. Strengthen password hashing
Do not use MD5 for any authentication or security purpose. Replace with:
- Argon2
- bcrypt
- PBKDF2

### 6. Improve error handling
Ensure SQL errors are not reflected directly in the response.

### 7. Add monitoring and intrusion detection
- Track repeated suspicious queries.
- Log unexpected use of `UNION` or attempts to access unusual columns.

---

## Detection and verification
- **Before remediation:** Test payloads containing `'`, `UNION SELECT`, and attempts to read from `information_schema` should work, confirming the vulnerability.
- **After remediation:**
  - These payloads must be rejected or treated as plain text.
  - The database must not execute attacker-supplied SQL fragments.
  - Sensitive fields should not be retrievable via manipulated queries.

---

## Fix validation checklist
- [ ] All search functionality uses parameterized queries.
- [ ] Input validation enforced.
- [ ] DB user privileges limited.
- [ ] No sensitive data stored in comments or metadata.
- [ ] MD5 hashes fully removed from the system.
- [ ] SQLi payloads no longer function.

---

## Recommendations for further hardening
1. Conduct a full audit of all user-facing input fields.
2. Add automated SQL injection scanning to CI.
3. Apply secure coding training for developers.
4. Consider deploying a WAF to block known SQLi patterns.

---

