# Darkly — README: Insecure Password Reset Mechanism / Manipulable Hidden Field

## Summary
This document describes a vulnerability in the password recovery feature. The form sends a POST request containing a hidden `mail` field. Since the value is entirely controlled on the client side and not validated server-side, an attacker can modify the email to any arbitrary address. Submitting a modified email triggers an unintended server response, revealing a flag.

This README explains the root cause, attack path, and correct remediation strategies.

---

## Vulnerability overview
- **Type:** Insecure Password Reset Mechanism / Client-Side Parameter Manipulation
- **Affected component:** Password reset form
- **Attack vector:** Editing hidden form field `mail`
- **Impact:** Unauthorized password reset behavior and flag disclosure

---

## Proof of concept (high level)
1. The password reset page contains the following form structure:
   ```html
   <form method="POST" action="/lost_password">
     <input type="hidden" name="mail" value="webmaster@borntosec.com" maxlength="15" />
     <button type="submit">Recover password</button>
   </form>
   ```

2. The application assumes the user cannot change the hidden field.

3. Using developer tools or by intercepting the request, the attacker modifies the hidden input:
   ```html
   <input type="hidden" name="mail" value="attacker@example.com" />
   ```

4. Submitting this request causes the backend to treat the arbitrary address as legitimate.

5. The server returns a response containing the flag.

This shows that the backend incorrectly trusts client-side values for critical authentication functionality.

---

## Root cause analysis
1. **Use of hidden fields to store security-sensitive parameters.** Hidden inputs are easily modifiable and provide no protection.

2. **Absence of server-side validation.** The backend does not verify that the submitted `mail` value corresponds to the authenticated user or a valid account.

3. **Lack of authentication or challenge before password recovery.** Any email value is accepted as-is.

4. **No rate-limiting or identity verification.** Password reset endpoints must verify ownership of the email address before issuing sensitive responses.

5. **Weak business logic.** Sensitive flows (password recovery, identity confirmation) rely on the client to enforce constraints.

---

## Impact
- **Unauthorized password reset behavior.** Attackers can initiate password recovery flows for arbitrary accounts.
- **Data disclosure.** The server reveals a flag or other sensitive information.
- **Potential account compromise in real systems.** In production scenarios, this could allow full account takeover.

Severity classification: **High**, since authentication and identity mechanisms are compromised.

---

## Remediation (recommended fixes)

### 1. Never trust hidden fields for critical data
Hidden inputs should contain non-sensitive UI metadata only. They must not be used to identify accounts or enforce restrictions.

### 2. Verify identity server-side
The server must check that the email submitted:
- belongs to an existing user, and
- corresponds to the authenticated user (if applicable), or
- is validated through a challenge process.

### 3. Implement a secure password reset workflow
A proper reset flow includes:
1. User submits an email.
2. If it exists, send a time-limited token to that email.
3. User clicks the token link.
4. Server verifies the token and allows password update.

### 4. Do not reveal whether an email exists
Error messages should not distinguish between existing and non-existing accounts.

### 5. Enforce rate limiting
Block automated attempts and repeated requests.

### 6. Audit and secure all forms
Any form containing hidden fields must not be trusted for authentication or identity-related logic.

---

## Detection and verification
- **Before remediation:** Modify the hidden field and submit the form. Any altered email value is accepted, and the flag is returned.
- **After remediation:**
  - Arbitrary email addresses must not trigger sensitive server behavior.
  - Reset flow must require email verification via token.
  - No flag or sensitive response must be disclosed based solely on form manipulation.

---

## Fix validation checklist
- [ ] Hidden fields removed from security logic.
- [ ] Server-side checks implemented for email ownership.
- [ ] Token-based password reset workflow added.
- [ ] No sensitive information exposed in responses.
- [ ] Rate-limiting and logging enabled.

---

## Recommendations for further hardening
1. Add CAPTCHA or equivalent anti-automation challenge.
2. Store password reset tokens using secure hashing.
3. Review all authentication-related endpoints for insecure direct object references.
4. Conduct regular security assessments focused on business logic flaws.

---