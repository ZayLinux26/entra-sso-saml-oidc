# Concepts Primer — SSO, Federation, SAML, OIDC & the Microsoft Identity Stack

This document explains the concepts behind the project in depth. It is written so that reading it is enough to confidently discuss federated single sign-on in an interview.

---

## 1. Single Sign-On (SSO)

SSO is an **outcome**, not a product or protocol: one authentication grants access to many applications without re-entering credentials. Everything else in this document is *how* that outcome is achieved securely.

---

## 2. Identity Provider (IdP) vs Service Provider (SP)

Every SSO relationship has exactly two roles:

- **Identity Provider (IdP):** holds the user accounts and performs authentication. It issues a signed statement asserting who the user is. *In this project: Microsoft Entra ID.*
- **Service Provider (SP):** the application that **trusts** the IdP and consumes its assertion to grant access. *In this project: the SAML Toolkit (SAML) and a registered app (OIDC).*

The entire practice of SSO is establishing and maintaining the **trust relationship** between these two roles.

---

## 3. Federation

**Federation** is that trust relationship when the IdP and SP are **separate** systems or organizations. It is created by exchanging **metadata** (endpoints + a signing certificate) so each side knows how to talk to, and verify, the other. Federation is the *mechanism*; SSO is the *experience* it produces.

---

## 4. SAML 2.0 in depth

SAML (Security Assertion Markup Language) is an XML-based federation standard, common in enterprise and legacy SaaS.

**The objects exchanged:**
- **Entity ID** — a unique identifier for each party (SP and IdP each have one).
- **Assertion Consumer Service (ACS) URL** — the SP endpoint where the IdP POSTs the signed response.
- **SSO / Login URL** — the IdP endpoint the SP redirects users to for authentication.
- **Signing certificate** — the IdP signs the assertion; the SP verifies it with the IdP's public certificate.
- **Metadata** — an XML document bundling the above.

**The flow (SP-initiated):**
1. User hits the SP → SP redirects to the IdP with a SAML **`AuthnRequest`** (HTTP-Redirect binding).
2. IdP authenticates the user.
3. IdP returns a **SAML Response** containing a signed **`<Assertion>`** (HTTP-POST binding) to the SP's ACS URL.
4. SP validates the signature, reads the assertion, and creates a session.

**Inside an assertion:**
- `<Issuer>` — who issued it (the IdP).
- `<Subject><NameID>` — *who* authenticated (the primary identifier; often email).
- `<Conditions NotBefore / NotOnOrAfter>` — the validity window.
- `<AudienceRestriction><Audience>` — the intended recipient (must equal the SP's Entity ID).
- `<AttributeStatement>` — additional claims (email, given name, surname, group…).
- `<Signature>` — proves authenticity and integrity.

**SP-initiated vs IdP-initiated:** SP-initiated starts at the application and redirects to the IdP (preferred). IdP-initiated starts at the IdP portal and POSTs to the SP.

---

## 5. OpenID Connect (OIDC) in depth

OIDC is an **identity layer built on OAuth 2.0**, common for modern web, mobile, and single-page apps. Where OAuth 2.0 handles *authorization* (access to resources), OIDC adds *authentication* (proving who the user is) via the **ID token**.

**Key objects:**
- **client_id** — the app's unique identifier (the OIDC analog of a SAML Entity ID).
- **redirect_uri** — where the IdP returns the token (the analog of the ACS URL).
- **scope** — what's requested; `openid` is required to get an ID token, with `profile`, `email`, etc. for additional claims.
- **ID token (JWT)** — the signed proof of authentication.
- **access token** — a separate token for calling APIs (not used for login itself).
- **nonce** — a value the app sends and expects echoed back in the token, preventing replay.

**The JWT structure:** three Base64URL parts — `header.payload.signature`.
- *Header:* `alg` (e.g. `RS256`), `kid` (which signing key).
- *Payload (claims):* `iss` (issuer), `aud` (audience = client_id), `sub` (subject), `iat/nbf/exp` (validity), plus identity claims (`email`, `name`, `given_name`…), and `nonce`.
- *Signature:* signs the header+payload so tampering is detectable.

**Flows:**
- **Authorization Code + PKCE** — the modern, secure standard for web/mobile/SPA; tokens are exchanged server-side / with a proof key, never exposed in the URL.
- **Implicit** — returns the token directly in the URL fragment; simple for testing (used here so `jwt.ms` could display the token) but **legacy** because tokens can leak via history/referrer.
- **Hybrid** — a mix; less common.

---

## 6. SAML vs OIDC — the same trust, two encodings

| | SAML 2.0 | OpenID Connect |
|---|----------|----------------|
| Era / base | Standalone XML | OAuth 2.0 (JSON) |
| Proof object | `<Assertion>` (XML) | ID token (JWT) |
| Subject | `NameID` | `sub` / `preferred_username` |
| Audience | `<Audience>` | `aud` |
| Issuer | `<Issuer>` | `iss` |
| Claims | `<AttributeStatement>` | token claims |
| Best fit | enterprise / legacy SaaS | modern web / mobile / SPA |

A SAML assertion and an OIDC ID token play the **same role**: a signed statement from the IdP vouching for the user, carrying claims. Understand one and you understand both.

---

## 7. Active Directory vs AD FS vs Entra ID

- **Active Directory (AD):** the on-premises **directory** — the database of accounts. A store, not an IdP.
- **AD FS (Active Directory Federation Services):** Microsoft's on-premises **federation server** — the legacy way to do SAML/WS-Fed SSO. You stood up AD FS as your IdP.
- **Entra ID (formerly Azure AD):** Microsoft's **cloud identity platform** and modern IdP, which has largely **replaced AD FS**. On-prem AD can sync to Entra via Entra Connect.

*One line:* AD is the directory, AD FS was the on-prem federation server, Entra ID is its cloud successor — and SAML/OIDC are the protocols they speak.

---

## 8. Conditional Access (concept)

Conditional Access is Entra's **policy engine** — an "if *signals*, then *control*" layer evaluated at sign-in.
- **Signals:** user/group, application, device state, location, sign-in risk.
- **Grant controls** ("can you get in?"): block, allow, **require MFA**, require compliant device, etc.
- **Session controls** ("what governs the session?"): **sign-in frequency**, persistent browser, app-enforced restrictions.

It's how zero-trust is operationalized: demand stronger authentication exactly where risk warrants it, without forcing friction everywhere.

---

## 9. Glossary

- **Assertion / ID token** — the IdP's signed statement of who authenticated.
- **NameID / sub** — the primary subject identifier; the join key to the SP's local account.
- **ACS URL / redirect_uri** — where the proof object is delivered.
- **Entity ID / client_id** — the unique identifier of a party.
- **Metadata** — bundled endpoints + certificate exchanged to establish trust.
- **Audience (aud)** — the intended recipient of the token.
- **Account matching** — the SP linking an incoming identity to a local user by NameID/sub.
- **Conditional Access** — Entra's sign-in policy engine.
- **MFA** — a second authentication factor beyond a password.

---

## 10. Common questions, answered

- **SSO vs federation?** SSO is the outcome — one login, many apps. Federation is the trust relationship between an IdP and SP that makes it possible across systems.
- **Walk me through the SAML flow.** App redirects to IdP with an AuthnRequest → IdP authenticates → signed assertion POSTed to the SP's ACS URL → SP validates signature, reads NameID → access granted.
- **SAML vs OIDC — which/when?** SAML (XML) for enterprise/legacy SaaS; OIDC (JWT, on OAuth 2.0) for modern web/mobile/SPA. Same trust idea, different encoding.
- **IdP vs SP?** IdP authenticates and asserts identity; SP trusts the IdP and consumes the assertion/token.
- **AD vs AD FS vs Entra ID?** AD is the on-prem directory; AD FS was the on-prem federation server; Entra ID is the cloud IdP that replaced it.
- **What's in a SAML assertion?** Issuer, Subject (NameID), validity conditions, audience, attribute claims, and a signature.
- **What's exchanged during setup?** Metadata: the SP's Entity ID + ACS URL, and the IdP's login URL, issuer, and signing certificate.
- **SP-initiated vs IdP-initiated?** SP-initiated begins at the app (preferred); IdP-initiated begins at the IdP portal.
- **Common SSO failures?** Entity ID/ACS mismatch, NameID format/value mismatch, bad/expired certificate, clock skew, redirect_uri mismatch (OIDC).
- **What is Conditional Access?** Entra's policy engine that allows/blocks/steps-up access based on signals — e.g., require MFA for a sensitive app.
- **Why is implicit flow discouraged?** It exposes tokens in the URL; the authorization code flow with PKCE is the modern, secure standard.
