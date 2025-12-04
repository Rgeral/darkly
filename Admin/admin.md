# Darkly — README: Sensitive File Exposure via `robots.txt` and Weak Credential Storage

## Summary
This document analyzes a vulnerability chain in which sensitive authentication material is indirectly exposed through the `robots.txt` file. Search engine exclusion directives point to a hidden directory, which contains an `.htpasswd` file storing an MD5-hashed password. When the hash is cracked, it allows unauthorized access to the `/admin` page, ultimately revealing a flag.

The goal of this README is to clearly explain how the vulnerability occurs, how it can be reproduced, and how it must be remediated to prevent similar issues.

---

## Vulnerability overview
- **Type:** Information Disclosure, Weak Credential Storage, Poor Access Control
- **Entry point:** `robots.txt`
- **Affected paths:** `/robots.txt`, `/whatever`, `/admin`, `.htpasswd`
- **Underlying issue:** Sensitive resources are exposed through predictable locations, and credentials are stored using weak cryptographic hashing.

---

## Proof of concept (high level)
1. Navigate to:
   ```
   http://10.14.200.249/robots.txt
   ```
   The file includes a directive:
   ```
   Disallow: /whatever
   ```

2. Visiting this directory reveals an `.htpasswd` file containing:
   ```
   root:437394baff5aa33daa618be47b75cb49
   ```

3. The hash uses **MD5**, a deprecated and easily crackable hashing function.
   Once cracked, it yields a valid password.

4. Using this recovered password on:
   ```
   http://10.14.200.249/admin
   ```
   grants access to the administrator panel, which exposes a flag.

This chain demonstrates how seemingly harmless metadata (`robots.txt`) can reveal sensitive resources when not designed or protected correctly.

---

## Root cause analysis
1. **Inappropriate use of `robots.txt` for hiding content.**
   - The `robots.txt` file is intended for search engine crawlers, not as a security mechanism.
   - Directories listed under `Disallow` are publicly visible to all clients and often attract attackers.

2. **Exposure of authentication-related files.**
   - The `.htpasswd` file is accessible from the web root.
   - No access control prevents direct download.

3. **Weak cryptographic hashing (MD5).**
   - MD5 is vulnerable to brute force attacks and widely considered insecure.
   - Storing passwords with MD5 enables trivial offline cracking.

4. **Lack of server-side protection for sensitive directories.**
   - Sensitive areas under `/whatever` lack configuration rules such as `Deny from all`, authorization requirements, or removal from the public document root.

5. **No rate-limiting or monitoring.**
   - Access to `.htpasswd` is not flagged.
   - Attack patterns originating from enumeration attempts are not detected.

---

## Impact
- **Unauthorized administrative access.** Cracked credentials allow login to the admin interface.
- **Confidentiality breach.** Exposure of hashed passwords compromises the integrity of authentication.
- **Chainable attack surface.** With administrative credentials, an attacker may modify content, escalate further, or plant malicious payloads.
- **False sense of security due to misuse of `robots.txt`.** Developers may incorrectly assume hidden paths are protected.

Severity assessment: **High**, due to direct administrative access resulting from publicly accessible authentication data.

---

## Remediation (recommended fixes)
Implement the following mitigations to eliminate the vulnerability and prevent recurrence.

### 1. Remove sensitive directories from public web root
- Authentication files like `.htpasswd` must never be directly accessible.
- Store them outside the document root or ensure strong access control rules.

### 2. Do not rely on `robots.txt` for security
- `robots.txt` only communicates crawling preferences; it provides no protection.
- Avoid listing sensitive directories in `robots.txt` entirely.
- If a directory must not be indexed, combine this with proper access restrictions.

### 3. Enforce strict server-side access control
- Apply directory-level restrictions using server configuration (e.g., `.htaccess`, Nginx location rules).
- Disallow direct access to configuration and authentication files.

### 4. Use strong hashing for stored credentials
- Replace MD5 with slow, salted password hashing algorithms such as:
  - `bcrypt`
  - `Argon2`
  - `PBKDF2`
- Rotate all credentials exposed during evaluation.

### 5. Harden the admin interface
- Add a rate limiter, enforce strong passwords, and implement lockout thresholds.
- Consider two-factor authentication for sensitive panels.

### 6. Improve monitoring and logging
- Log failed login attempts.
- Detect suspicious enumeration patterns.
- Alert when abnormal access to sensitive paths occurs.

---

## Secure configuration pattern (examples)

### Move `.htpasswd` outside the document root
```
/var/www/darkly/               (public)
/etc/darkly/auth/.htpasswd     (private)
```
Web server config should reference the private path explicitly.

### Deny direct access with server rules (Apache example)
```
<FilesMatch "^(\.htpasswd|\.htaccess)$">
  Require all denied
</FilesMatch>
```

### Example of strong hashing (pseudo-code)
```
stored_hash = bcrypt.hash(password_input)
verify(password_input, stored_hash)
```

---

## Detection and verification
- **Pre-fix detection:** Confirm accessibility of `/whatever` and `.htpasswd` through direct URLs.
- **Post-fix verification:** Attempting to access these resources must now return `403 Forbidden` or `404 Not Found`.
- **Credential testing:** MD5-hashed passwords should no longer be present in the system.
- **Admin panel test:** The `/admin` page must require valid credentials and should not be accessible through any leaked or legacy passwords.

---

## Fix validation checklist
- [ ] `.htpasswd` is removed from the public directory.
- [ ] Direct access to sensitive files is denied.
- [ ] Admin password hashing upgraded to a modern, slow KDF.
- [ ] `robots.txt` no longer exposes sensitive directory names.
- [ ] Logging and alerts enabled for suspicious access.
- [ ] Re-penetration test performed to confirm mitigation.

---

## Recommendations for further hardening
1. Conduct a full audit of the filesystem to ensure no other sensitive files are web-accessible.
2. Implement regular password rotation and enforce stricter complexity requirements.
3. Add automated scanning tools to detect weak hashing or exposed files.
4. Perform periodic security reviews after feature updates.

---

