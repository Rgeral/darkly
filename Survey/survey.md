# Darkly — README: Client-Side Validation Bypass in Form Input

## Summary
This document describes a vulnerability in a form that relies exclusively on client-side validation to restrict selectable values. The form presents a dropdown menu with numeric options ranging from 1 to 10. By modifying the submitted value (e.g., via browser developer tools or intercepting the request), it is possible to send values outside the permitted range. Submitting an out-of-range value triggers unintended server behavior and reveals a flag.

This write-up explains why the issue occurs, how it is exploited, and the proper server-side mitigations.

---

## Vulnerability overview
- **Type:** Client-side validation bypass / Insecure direct input handling
- **Affected component:** HTML `<select>` element with numeric `<option>` values
- **Impact:** Sending unauthorized values leads to flag disclosure
- **Attack vector:** Manipulation of form fields on the client side

---

## Proof of concept (high level)
1. The application provides a dropdown such as:
   ```html
   <select name="value">
     <option value="1">1</option>
     <option value="2">2</option>
     <option value="3">3</option>
     ...
     <option value="10">10</option>
   </select>
   ```

2. Although the user interface restricts selection to values 1–10, this restriction is **only enforced in the HTML**, not on the server.

3. Using browser developer tools, an attacker can modify the value before submission:
   ```html
   <option value="999">999</option>
   ```
   or manually edit the request payload.

4. Submitting a value greater than 10 returns a hidden response containing the flag.

This confirms that the backend trusts client-side constraints instead of applying its own validation.

---

## Root cause analysis
1. **Exclusive reliance on client-side validation.** HTML constraints (dropdown limits, input ranges) can be trivially modified by end users.

2. **Missing server-side validation.** The backend accepts the posted parameter without checking whether it lies in the allowed range.

3. **Implicit trust in the UI.** The server assumes that the client cannot alter form fields, leading to insecure design.

4. **Predictable functionality triggered by invalid input.** Submitting an out-of-range value triggers alternative code paths, exposing internal flags.

---

## Impact
- **Unauthorized access to restricted functionality.** Attackers can trigger code paths never intended for them.
- **Data exposure.** In this case, submitting out-of-range values reveals a flag.
- **Potential escalations.** Similar patterns can lead to privilege escalation, unintended server operations, or business logic abuse.

Severity classification: **Medium to High**, depending on what the backend reveals when out-of-range values are accepted.

---

## Remediation (recommended fixes)

### 1. Enforce server-side validation
The backend must verify that submitted values fall within the expected range:
```
if value < 1 or value > 10:
    reject_request()
```
Server-side checks must be considered authoritative.

### 2. Treat all client input as untrusted
- Never assume the client will respect HTML constraints.
- Validate type, range, and format for every parameter.

### 3. Avoid exposing sensitive logic based on raw parameter values
- Do not reveal flags or sensitive operations based solely on numeric input.
- Use proper authentication, authorization, and access control.

### 4. Consider duplicating restrictions client-side only for UX
- Client-side limits improve user experience, but should never replace backend checks.

### 5. Add logging for abnormal input
- Log unexpected values to detect tampering attempts.

---

## Detection and verification
- **Pre-fix reproduction:** Modify the `<option>` value or intercept the form submission to provide a value > 10; the flag appears.
- **Post-fix expectations:**
  - Values outside 1–10 must be rejected.
  - The backend must respond with an error (e.g., HTTP 400) or silently ignore the invalid input.
  - No sensitive information should be disclosed.

---

## Fix validation checklist
- [ ] Server-side range checks implemented.
- [ ] UI constraints reflect, but do not replace, backend rules.
- [ ] Invalid values produce no sensitive responses.
- [ ] Logging and monitoring enabled for abnormal inputs.

---

## Recommendations for further hardening
1. Apply strict schema validation on all inputs.
2. Use server frameworks that support automatic request validation.
3. Conduct regular review of form-handling code.
4. Integrate automated testing to ensure invalid inputs are consistently rejected.

---