# Troubleshooting & Failure Modes

Real SSO rarely works on the first attempt. This documents the issues hit during the build (with root cause and fix) and a general reference for diagnosing federation failures.

---

## Issues encountered

### 1. Circular metadata dependency
- **Symptom:** Entra's signing-certificate downloads were greyed out; the SP couldn't generate its endpoints either.
- **Root cause:** the IdP withholds its certificate/metadata until a relying party (Entity ID + Reply URL) is defined, while the SP needs the IdP's certificate to build its config — a deadlock.
- **Diagnosis:** recognized that each side was waiting on the other.
- **Fix:** a **placeholder bootstrap** — seed Entra's Basic SAML Configuration with temporary Identifier/Reply URL values to unlock the certificate, harvest the IdP metadata, configure the SP, then replace the placeholders with the SP's real Entity ID and ACS URL.
- **Takeaway:** federation setup is a two-way metadata exchange; break circular dependencies by bootstrapping one side with temporary values, then correcting them.

### 2. Wrong principal authenticating
- **Symptom:** login succeeded at Entra, but the SP returned `Object reference not set to an instance of an object`.
- **Root cause:** **decoding the SAML assertion** showed the `NameID` was the **admin** account, not the intended user — a lingering browser session was silently reused, and the SP had no local user matching that identity.
- **Diagnosis:** read the assertion's `<NameID>` to see *who actually authenticated*.
- **Fix:** re-ran the flow in an isolated (incognito) session and authenticated as the correct user; the assertion then carried the right NameID and the SP matched it.
- **Takeaway:** the assertion is the source of truth for who authenticated; and a privileged admin account should never be used for routine app SSO — exactly the anti-pattern PAM addresses.

### 3. Account-matching failure
- **Symptom:** a valid, signed assertion, but no login.
- **Root cause:** the SP requires a local user whose identifier equals the assertion's `NameID`; none matched.
- **Fix:** ensured a matching user existed at the SP and that the NameID (email) lined up exactly.
- **Takeaway:** the `NameID` / `sub` is the join key between IdP identity and SP account — its format and value must match what the SP expects.

---

## General SSO troubleshooting reference

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Entra login OK, app errors | Entity ID ≠ SP Entity ID (trailing slash, case) | Match exactly, both sides |
| App can't accept the response | Reply URL ≠ ACS URL | Match exactly, incl. scheme/path |
| "Signature validation failed" | Wrong/expired certificate at the SP | Re-pull the IdP certificate / metadata |
| Authenticated but not matched | NameID format/value ≠ SP's expectation | Adjust the NameID claim (email vs UPN) |
| "Assertion expired" intermittently | Clock skew between IdP and SP | Sync clocks; check NotBefore/NotOnOrAfter |
| Cert/metadata greyed out in Entra | Basic SAML Configuration incomplete | Populate Identifier + Reply URL, then save |
| OIDC "redirect_uri mismatch" | Request URI ≠ registered URI | Make them identical |
| OIDC token not returned | ID tokens not enabled / no consent | Enable ID tokens; grant consent |

---

## How to read the proof object when debugging

- **SAML:** in SAML-tracer, find the POST to the ACS URL containing `SAMLResponse`, open the decoded **SAML** tab, and inspect `<Issuer>`, `<Audience>`, `<NameID>`, `<Conditions>`, and the presence of `<Signature>`.
- **OIDC:** paste/redirect the token to `jwt.ms` and inspect `iss`, `aud`, `sub`, `nonce`, validity (`iat/nbf/exp`), and the header's `alg`.

The fastest path from "it broke" to "here's why" is almost always reading the assertion or token directly.
