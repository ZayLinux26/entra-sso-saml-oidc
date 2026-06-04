# Build Walkthrough — Entra ID SSO & Federation

The full build as executed, in order, with the reasoning behind each step and the evidence captured. Screenshot numbers map to `screenshots/`.

---

## Prerequisites

- A Microsoft Entra ID tenant with users and role-based groups (reused from the JML and RBAC projects).
- An account with Cloud Application Administrator (and Conditional Access Administrator for the MFA step).
- SAML-tracer browser extension for capturing the assertion.
- A scratch file for the values exchanged between the IdP and SPs.

---

## Track A — SAML SSO (Entra IdP → SAML Toolkit SP)

1. **Add the SP.** Entra admin center → Enterprise applications → New application → added **Microsoft Entra SAML Toolkit** from the gallery. *(Screenshot 02–03.)* An "enterprise application" is how Entra represents a service provider it federates with.
2. **Select SAML** as the single sign-on method — true federation (a signed token), as opposed to password-based vaulting or a plain linked tile.
3. **Pick the test user.** A clinical-staff user from the RBAC project; copied the exact **UPN (email)**, which becomes the assertion's `NameID`. *(Screenshot 04.)* Credential reset performed where needed (standard identity operations).
4. **Metadata exchange.** Captured the SP's **Entity ID**, **ACS URL**, and **SP-initiated login URL** from the Toolkit, and the IdP's **Login URL**, **Identifier**, and **signing certificate** from Entra. *(Screenshots 05–06.)*
5. **Configure Basic SAML Configuration** in Entra with the SP's real Identifier, Reply URL (ACS), and Sign-on URL. *(Screenshot 07.)* These must match the SP's values **exactly** — the #1 failure mode.
6. **Claims / NameID.** Confirmed `NameID = user.userprincipalname` (the email), plus `email`/`givenname`/`surname` claims. *(Screenshot 08.)* The NameID is the join key to the SP's local account.
7. **Assign by group.** Granted access via the **RBAC role-based group**, not per user. *(Screenshot 09.)* Access follows job role; lifecycle changes propagate through membership.
8. **Test (SP-initiated).** Triggered the SP-initiated login URL in a clean session and authenticated. *(Screenshots 10 / 10a; success 11.)*
9. **Validate.** Decoded the live assertion in SAML-tracer and confirmed Issuer, Audience, NameID, and signature. *(Screenshot 12.)*

> The first test failed and was debugged — see `troubleshooting.md`.

---

## Track B — OIDC SSO (Entra IdP → registered app)

1. **Register the app.** App registrations → New registration → `redirect_uri = https://jwt.ms`, single tenant. Captured the Application (client) ID and tenant ID. *(Screenshot 16.)* App registrations is the developer-side surface — distinct from enterprise applications.
2. **Enable ID tokens.** Authentication → enabled ID-token issuance for the implicit/hybrid flow, **for testing only** (production SPAs use authorization code + PKCE). *(Screenshot 17.)*
3. **Map optional claims.** Token configuration → added `email`, `given_name`, `family_name` to the ID token — the OIDC equivalent of a SAML attribute statement. *(Screenshot 18.)*
4. **Fire the flow.** Built an `/authorize` URL (`response_type=id_token`, `scope=openid profile email`, a `nonce`), authenticated as the test user, and landed on `jwt.ms`. *(Screenshot 19.)*
5. **Validate.** Confirmed `iss` (tenant), `aud` (client_id), `sub`/`preferred_username` (the user), the mapped claims, `nonce` match (replay protection), and `alg: RS256` (signed).

---

## Track C — Hardening

1. **Conditional Access — Require MFA.** New policy targeting the RBAC group and the federated app, Grant control = Require MFA, started in **report-only** then enforced. *(Screenshot 13.)*
2. **Sign-in frequency (session control).** Added periodic re-authentication, addressing HIPAA automatic-logoff for shared clinical workstations.
3. **Enforce & test.** Flipped the policy to On and re-ran the login; MFA was registered and required live. *(Screenshot 14.)*
4. **Audit trail.** Reviewed Sign-in logs showing the federated login, the applied CA policy, and the satisfied MFA. *(Screenshots 15 / 15a.)*

---

## Validation summary

| Object | Tool | Confirmed |
|--------|------|-----------|
| SAML assertion | SAML-tracer | Issuer, Audience, NameID, signature, validity window |
| OIDC ID token | jwt.ms | iss, aud, sub, claims, nonce, RS256 signature |
| MFA enforcement | Entra sign-in logs | CA policy applied, MFA method satisfied |

The result: one Entra identity, two protocols, MFA-enforced and fully auditable.
