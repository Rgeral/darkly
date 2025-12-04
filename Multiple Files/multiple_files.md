# Darkly — README: Hidden Directory Enumeration via `robots.txt` and Recursive Retrieval

## Summary
This document describes a vulnerability where sensitive files are exposed through a deeply nested hidden directory referenced in `robots.txt`. Although intended to hide content from search engines, `robots.txt` publicly reveals the existence of the `.hidden` directory. By recursively enumerating its subdirectories, it is possible to retrieve multiple `README` files, one of which contains the flag.

This write‑up explains how the flaw is discovered, why it exists, and the proper remediation strategies.

---

## Vulnerability overview
- **Type:** Information Disclosure / Predictable Resource Enumeration
- **Affected component:** `.hidden` directory referenced in `robots.txt`
- **Attack vector:** Recursive crawling and file enumeration
- **Impact:** Exposure of internal documentation and a flag hidden in one `README` file

---

## Proof of concept (high level)

1. The application's `robots.txt` file contains:
   ```
   Disallow: /.hidden
   ```
   Although intended for search engines, this disclosure *invites* attackers to inspect the directory.

2. Navigating to:
   ```
   http://10.14.200.249/.hidden/
   ```
   reveals numerous nested subdirectories, each containing additional layers of directories and files.

3. To efficiently retrieve all files, a recursive download is performed:
   ```
   wget -r -np -nH --cut-dirs=0 -e robots=off -U "Mozilla/5.0" http://10.14.200.249/.hidden/
   ```
   Flags used:
   - `-r` recursive download
   - `-np` do not ascend to parent directory
   - `-nH` avoid creating hostname folder
   - `--cut-dirs=0` preserve directory structure
   - `-e robots=off` ignore crawler restrictions
   - `-U` set a realistic user-agent

4. Once downloaded, all nested `README` files can be inspected:
   ```
   cat */*/*/README | grep flag
   ```

5. One of the many README files contains the flag, allowing the attacker to retrieve it without solving additional logic.

This demonstrates that relying on directory obscurity is not a security mechanism.

---

## Root cause analysis

1. **Misuse of `robots.txt`.**
   - `robots.txt` is a *public* file.
   - Listing sensitive directories effectively advertises their existence.

2. **Presence of sensitive files in publicly accessible directories.**
   - Internal or challenge-related READMEs should never be left in web‑accessible paths.

3. **Lack of access control.**
   - No permissions restrict access to the `.hidden` directory or its contents.

4. **Predictable and enumerable directory structure.**
   - Attackers can use recursive crawlers to exhaustively retrieve subdirectories.

5. **No rate‑limiting or anomaly detection.**
   - Large recursive downloads go unnoticed.

---

## Impact
- **Exposure of sensitive documentation** stored in hidden directories.
- **Flag disclosure** via enumeration.
- **Scalability of attack:** A single crawler can explore thousands of directories quickly.

Severity classification: **Medium**, escalating to **High** if similar directories store real credentials or sensitive internal files.

---

## Remediation (recommended fixes)

### 1. Do not use `robots.txt` to hide sensitive directories
`robots.txt` is not a security control. Anything listed inside is publicly visible.

### 2. Implement proper server‑side access control
Restrict access using server rules:
- Deny access to `.hidden` entirely.
- Move sensitive files outside the document root.

### 3. Remove unnecessary files from production
Internal or debug documentation should not be exposed to users.

### 4. Add authentication for sensitive resources
If pages or files must remain online, require authentication or token-based access.

### 5. Monitor directory crawling attempts
- Detect recursive HTTP requests.
- Log and alert on abnormal traffic patterns.

### 6. Avoid predictable naming conventions
Attackers often locate resources by guessing or enumerating directory structures.

---

## Detection and verification
- **Before remediation:** Accessing `/.hidden/` reveals numerous subdirectories; recursive crawling retrieves all READMEs; one contains the flag.
- **After remediation:**
  - `http://<server>/.hidden/` must return 403 or 404.
  - READMEs must no longer be publicly accessible.
  - Recursive downloads should fail due to access controls.

---

## Fix validation checklist
- [ ] `.hidden` directory removed or protected.
- [ ] Sensitive files moved outside the web root.
- [ ] No sensitive paths referenced in `robots.txt`.
- [ ] Access control rules enforced at server level.
- [ ] Recursion attempts fail with proper HTTP errors.

---

## Recommendations for further hardening
1. Conduct a full audit for forgotten or hidden directories.
2. Use infrastructure automation to prevent accidental exposure of internal files.
3. Apply content‑security scanning tools to detect exposed documents.
4. Review crawler‑related logs regularly.

---
