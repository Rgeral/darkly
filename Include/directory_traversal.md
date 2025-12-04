# Darkly — README: Directory Traversal / Local File Inclusion (LFI) Vulnerability

## Summary
This document describes a directory traversal / local file inclusion vulnerability discovered in the Darkly application. The issue allows user-supplied path traversal through the `page` parameter which can cause arbitrary local files to be read by the web application. This README explains the root cause, impact, detection, and recommended remediation steps in a technical, fact-focused manner.

---

## Vulnerability overview
- **Type:** Directory Traversal / Local File Inclusion (LFI)
- **Affected parameter:** `page` (HTTP GET parameter)
- **Observed behavior:** Supplying a path containing `..` sequences causes the application to include or read files outside the intended document/template directory. In the evaluation the following request returned system files: `http://10.14.200.249/?page=../../../../../../../etc/passwd`.
- **Risk:** This vulnerability permits disclosure of arbitrary local files readable by the web server process and can lead to information disclosure, credential leakage, and — in chained attacks — remote code execution depending on the environment and other application weaknesses.

---

## Proof of concept (high level)
A request was made with a `page` parameter containing multiple `..` segments resolving to the system root; the request returned the contents of `/etc/passwd`. The presence of sensitive system file content demonstrates that user input is used directly (or insufficiently sanitized) in file path selection.

> Note: only the minimal reproduction context is recorded here. Do not use this document as an exploitation guide — it exists to help developers and maintainers understand and fix the weakness.

---

## Root cause analysis
1. **Unvalidated user-controlled file path.** The application constructs a filesystem path from the untrusted `page` parameter and performs a file include/read operation without canonicalizing and validating the resolved path.
2. **Missing allowlist / insufficient path restriction.** There is no strict allowlist of permitted page names or templates; or the application relies on blacklisting (e.g., filtering out `..`) which is brittle and error-prone.
3. **Insufficient privilege separation.** The web server process can read files outside the intended web root, increasing the blast radius when path traversal is possible.
4. **Lack of defense-in-depth.** Logging, monitoring, and file access restrictions are not adequate to prevent or detect large-scale scanning of the filesystem.

---

## Impact
- **Information disclosure:** Any file readable by the web application's process can be exfiltrated (configuration files, credentials, SSH keys, source code, etc.).
- **Privilege escalation / code execution (indirect):** If attacker-controlled files can be written to an includable location (through separate vulnerabilities), or if the environment interprets included content as code, this can chain into remote code execution.
- **Compliance and confidentiality risks:** Exposure of secrets and PII may violate policy and regulatory requirements.

---

## Severity (technical estimate)
Using standard risk considerations (ease of exploitation, impact of data disclosure, and potential to pivot), this flaw should be treated as **High** severity until mitigations are applied and verified.

---

## Remediation (recommended fixes)
Apply the following defenses, ordered from strongest to least fragile. Prefer multiple layers (allowlist + canonicalization + least privilege).

1. **Implement an explicit allowlist of pages/templates.**
   - Map logical page identifiers (for example: `home`, `about`, `contact`) to static template files on the server. Never accept raw file paths from user input.
   - Example approach: a switch/lookup table where the user sends a page *name*, and the server resolves it to a filename from the allowlist.

2. **Canonicalize and validate any filesystem paths before use.**
   - If allowing some user-supplied path fragments is unavoidable, compute the canonical absolute path (`realpath` or equivalent) and verify that it is inside the intended document root. Reject requests that resolve outside the permitted directory.

3. **Avoid direct file inclusion of user-controlled strings.**
   - Do not pass raw user input into `include`/`require` or file reading functions. Use safe abstractions and template engines when rendering user-facing pages.

4. **Reduce file read privileges of the web process.**
   - Run the webserver under a dedicated, low-privilege account with access only to the web application's content directory. Remove read access to sensitive system files where possible.

5. **Harden server configuration.**
   - Configure the web server and runtime to disallow serving files outside the document root. Disable directory listing and unnecessary file handlers.

6. **Input canonicalization over naive blacklists.**
   - Blacklisting patterns like removing `..` or replacing `../` are error-prone. Use canonicalization checks and allowlists instead.

7. **Logging and monitoring.**
   - Log attempted accesses to files outside normal templates and create alerts for unusual file access patterns.

8. **Use a secure templating engine.**
   - Prefer template systems that separate template identifiers from filesystem paths and which do not evaluate templates as arbitrary code.
