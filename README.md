# Single Sign-On & Federation with Microsoft Entra ID
### SAML 2.0 + OpenID Connect · Conditional Access MFA · Centralized Audit Logging

A hands-on identity & access management project that configures **Microsoft Entra ID as an enterprise Identity Provider (IdP)** and federates applications to it over **both** major federation protocols — **SAML 2.0** and **OpenID Connect (OIDC)** — then hardens access with **Conditional Access MFA** and proves an auditable sign-in trail.

Built end-to-end in a live Entra tenant against a fictional regional healthcare network, **Meridian Health Partners (MHP)**, so every control is grounded in real compliance drivers (**HIPAA Security Rule, NIST SP 800-53, ISO/IEC 27001**).

**Tech:** Microsoft Entra ID · SAML 2.0 · OpenID Connect / OAuth 2.0 · Conditional Access · Microsoft Entra SAML Toolkit · jwt.ms · SAML-tracer

---

## Table of Contents
1. [Business Context & Problem](#1-business-context--problem)
2. [Objectives & Success Criteria](#2-objectives--success-criteria)
3. [Core Concepts (the federation model)](#3-core-concepts-the-federation-model)
4. [Architecture](#4-architecture)
5. [How It Works — Authentication Flows](#5-how-it-works--authentication-flows)
6. [Implementation](#6-implementation)
7. [Validating the Proof Objects](#7-validating-the-proof-objects)
8. [Real-World Troubleshooting](#8-real-world-troubleshooting)
9. [Security Hardening](#9-security-hardening)
10. [Design Decisions & Rationale](#10-design-decisions--rationale)
11. [Evidence Index](#11-evidence-index)
12. [Compliance Mapping](#12-compliance-mapping)
13. [Outcomes](#13-outcomes)
14. [Résumé Bullets](#14-résumé-bullets)
15. [Skills Demonstrated](#15-skills-demonstrated)
16. [Repository Structure](#16-repository-structure)
17. [References](#17-references)

---

## 1. Business Context & Problem

Meridian Health Partners (MHP) is a ~520-employee regional healthcare network spanning three facilities. Like most organizations that grew through acquisition, MHP accumulated a fragmented application landscape — clinical portals, scheduling tools, and back-office systems, each with its own login. That fragmentation creates four problems an IAM analyst is directly responsible for solving:

| # | Problem | Risk | Why it matters in healthcare |
|---|---------|------|------------------------------|
| 1 | **Password sprawl** | Credential reuse, weak passwords | Stolen/phished credentials are the leading cause of reportable healthcare breaches |
| 2 | **Inconsistent access control** | Lingering access on role change/termination | Orphaned access to systems holding PHI is a HIPAA finding |
| 3 | **No central audit trail** | Can't answer "who accessed what, when" | Auditors require demonstrable access records |
| 4 | **Authentication friction** | Lost clinician time at point of care | Slow logins push staff toward insecure workarounds |

**Project thesis:** Establish Microsoft Entra ID as MHP's single, authoritative Identity Provider and federate representative applications to it. One identity then governs access everywhere — centralized, MFA-enforced, and fully auditable — directly resolving all four problems.

---

## 2. Objectives & Success Criteria

**Objectives**
- Configure Entra ID as an IdP and federate a **SAML 2.0** service provider.
- Federate a second service provider over **OpenID Connect** to demonstrate both protocols.
- Enforce **multi-factor authentication** on federated access via Conditional Access.
- Apply a **session-lifetime** control (sign-in frequency) for shared-device hygiene.
- Produce a **centralized audit trail** of federated sign-ins.
- Capture and **validate** both proof objects (SAML assertion, OIDC ID token) field by field.

**Success criteria (all met)**
- [x] A SAML SP authenticates a real user via Entra, SP-initiated, end to end.
- [x] An OIDC SP returns a valid, decoded ID token for the same identity.
- [x] Application access is granted by **role-based group**, not per user.
- [x] A Conditional Access policy **requires MFA** for the federated app.
- [x] Sign-in logs evidence the federated login **and** that MFA + the policy applied.
- [x] Every failure encountered was diagnosed, fixed, and documented.

---

## 3. Core Concepts (the federation model)

> This section exists to demonstrate conceptual fluency, not just click-paths.

**Single Sign-On (SSO)** is an *outcome*: one authentication grants access to many applications. It is not a product or protocol — it's the result of establishing trust correctly.

**Identity Provider (IdP) vs Service Provider (SP).** Every SSO relationship has exactly two roles. The **IdP** holds the accounts and authenticates the user (here, Entra ID). The **SP** is the application that trusts the IdP and consumes its verdict. The entire discipline of SSO is establishing and maintaining trust between these two roles.

**Federation** is that trust relationship when the IdP and SP are *separate* systems or organizations. Federation is the mechanism; SSO is the experience it produces.

**SAML 2.0 vs OpenID Connect.** Both carry the trust; they differ in age and shape:

| | SAML 2.0 | OpenID Connect |
|---|----------|----------------|
| Format | XML **assertion** | JSON Web Token (**JWT**) |
| Built on | Standalone XML standard | OAuth 2.0 |
| Typical use | Enterprise / legacy SaaS | Modern web, mobile, SPA |
| Proof object | Signed `<Assertion>` posted to the **ACS URL** | Signed **ID token** returned to the **redirect URI** |
| Primary subject | `NameID` | `sub` |
| Decoded with | SAML-tracer | jwt.ms |

The key insight, proven in this project: a **SAML assertion** and an **OIDC ID token** are the *same idea* — a signed statement from the IdP vouching for the user, carrying claims. Understand one and you understand both.

**Active Directory vs AD FS vs Entra ID** (a commonly confused trio):
- **Active Directory (AD):** the on-premises *directory* — the database of user accounts. A store, not an IdP.
- **AD FS (Active Directory Federation Services):** Microsoft's *on-premises federation server* — the legacy way to do SAML/WS-Fed SSO before the cloud. You stood up AD FS as your IdP.
- **Entra ID (formerly Azure AD):** Microsoft's *cloud identity platform* and the modern IdP that has largely **replaced AD FS**. On-prem AD can synchronize into Entra via Entra Connect.

In one line: *AD is the directory, AD FS was the on-prem federation server, Entra ID is its cloud successor — and SAML/OIDC are the protocols they speak to do SSO.*

---

## 4. Architecture

```
   MHP users / RBAC groups ─►        Microsoft Entra ID          (Identity Provider)
        (from JML + RBAC)      authenticates + issues signed claims
                          SAML │                            │ OIDC
              ┌────────────────▼───────┐      ┌─────────────▼─────────────┐
              │  Entra SAML Toolkit     │      │  Registered OIDC app       │   (Service Providers)
              │  (SAML 2.0 SP)          │      │  redirect_uri → jwt.ms     │
              └─────────────────────────┘      └────────────────────────────┘

   Cross-cutting hardening:
   • Group-based assignment (RBAC)   • Conditional Access — Require MFA
   • Sign-in frequency (re-auth)     • Sign-in logs (centralized audit)
```

The users and role-based groups reused here were provisioned in the earlier **User Lifecycle (JML)** and **RBAC** projects — so access is granted by job-role group, and a single identity flows across protocols. This continuity is intentional: it demonstrates lifecycle → RBAC → SSO as one coherent identity program.

---

## 5. How It Works — Authentication Flows

**SAML 2.0 (SP-initiated)**
1. The user opens the SP (the SAML Toolkit); the app has no session.
2. The SP redirects the browser to Entra with a SAML `AuthnRequest`.
3. Entra authenticates the user and applies Conditional Access (MFA).
4. Entra returns a **signed SAML Response** containing the assertion.
5. The browser POSTs it to the SP's **Assertion Consumer Service (ACS) URL**.
6. The SP validates the signature against Entra's certificate, reads the `NameID`, matches a local user, and grants access.

**OpenID Connect (authorization endpoint, ID token)**
1. The browser is sent to Entra's `/authorize` endpoint with the app's `client_id`, `redirect_uri`, and `scope=openid …`.
2. Entra authenticates the user.
3. Entra redirects back to the **redirect URI** with a signed **ID token (JWT)**.
4. The app (here, jwt.ms) validates the token and reads its claims (`iss`, `aud`, `sub`, `email`, …).

Same trust story, two encodings.

---

## 6. Implementation

**Track A — SAML SSO (Entra IdP → SAML Toolkit SP)**
- Added the Microsoft Entra SAML Toolkit as an **enterprise application** and selected **SAML** as the SSO method.
- Performed a **bidirectional metadata exchange**: the SP's Entity ID, ACS URL, and SP-initiated login URL into Entra; Entra's Login URL, Issuer, and signing certificate into the SP.
- Mapped the **NameID** to the user's email for account matching, and configured `email` / `givenname` / `surname` claims.
- Assigned access by **RBAC group**.
- Tested SP-initiated login and **decoded the live assertion** to validate Issuer, Audience, NameID, and signature.

**Track B — OIDC SSO (Entra IdP → registered app)**
- Registered an application (**App registration**, the developer-side surface — distinct from enterprise applications) with `redirect_uri = https://jwt.ms`.
- Enabled ID-token issuance and mapped optional claims (`email`, `given_name`, `family_name`).
- Initiated the OIDC flow and **decoded the live ID token** at jwt.ms, validating `iss`, `aud`, `sub`, claims, `nonce`, and the `RS256` signature.

**Track C — Hardening**
- Created a **Conditional Access** policy requiring **MFA** for the federated app (started in report-only, then enforced).
- Added a **sign-in frequency** session control for periodic re-authentication.
- Reviewed **sign-in logs** confirming the federated login, the applied CA policy, and the satisfied MFA.

---

## 7. Validating the Proof Objects

Validation — not just "it logged in" — is the core deliverable.

**SAML assertion (decoded in SAML-tracer):** confirmed `<Issuer>` = Entra, `<Audience>` = the SP Entity ID, `<NameID>` = the user's email, the presence of a `<Signature>`, and the `NotBefore`/`NotOnOrAfter` validity window.

**OIDC ID token (decoded at jwt.ms):** confirmed `iss` = the tenant endpoint, `aud` = the app's `client_id`, `sub` / `preferred_username` = the user, the mapped claims (`given_name`, `family_name`, `name`), `nonce` matching the request (replay protection), and `alg: RS256` (signed).

**The two proof objects, side by side:**

| SAML assertion (XML) | OIDC ID token (JWT) | Meaning |
|---|---|---|
| `<Issuer>` | `iss` | issued by the IdP |
| `<Audience>` | `aud` | intended for this app |
| `<NameID>` | `sub` / `preferred_username` | who authenticated |
| `<AttributeStatement>` | `email`, `given_name`, `family_name` | identity claims |
| `<Signature>` | `alg: RS256` + signature | tamper-evident |

---

## 8. Real-World Troubleshooting

Production SSO rarely works on the first attempt. Three real failures were encountered, diagnosed, and resolved — the ability to do this is what separates operating SSO from reading about it.

**1. Circular metadata dependency**
- *Symptom:* Entra's signing certificate downloads were greyed out; the SP couldn't produce its endpoints either.
- *Root cause:* the IdP withholds its certificate/metadata until a relying party (Entity ID + Reply URL) is defined, while the SP needs the IdP's certificate to generate its config — a deadlock.
- *Resolution:* a **placeholder bootstrap** — seed Entra with temporary Identifier/Reply URL values to unlock the certificate, harvest the IdP metadata, configure the SP, then replace the placeholders with the SP's real Entity ID and ACS URL.
- *Takeaway:* federation setup is a two-way metadata exchange; circular dependencies are broken by bootstrapping one side.

**2. Wrong principal authenticating**
- *Symptom:* Entra login succeeded, but the SP crashed with `Object reference not set to an instance of an object`.
- *Root cause:* **decoding the SAML assertion** revealed the `NameID` was the *admin* account, not the intended user — a lingering browser session had been silently reused, and the SP had no local user matching that identity.
- *Resolution:* re-ran the flow in an isolated (incognito) session and authenticated as the correct principal; the assertion then carried the right `NameID` and the SP matched it.
- *Takeaway:* read the assertion to see *who actually authenticated*; and never use a privileged admin account for routine app SSO — that is precisely the anti-pattern Privileged Access Management addresses.

**3. Account-matching failure**
- *Symptom:* valid, signed assertion, but no login.
- *Root cause:* the SP requires a local user whose identifier equals the assertion's `NameID` — the classic federation failure mode.
- *Resolution:* ensured a matching user existed at the SP and that the `NameID` claim (email) lined up exactly.
- *Takeaway:* the `NameID`/`sub` is the join key between IdP identity and SP account; format and value must match.

---

## 9. Security Hardening

- **Conditional Access — Require MFA:** MFA registered and enforced live on the federated app. Password-only access is eliminated, directly mitigating credential theft and phishing — the top healthcare breach vector. (NIST 800-53 IA-2(1).)
- **Sign-in frequency (session control):** forces periodic re-authentication, addressing **HIPAA §164.312(a)(2)(iii) automatic logoff** — essential for shared clinical workstations.
- **Report-only first:** the CA policy was validated in report-only mode before enforcement, a standard practice to avoid lockouts.
- **Centralized sign-in logs:** provide the authoritative audit record (user, app, source, MFA result, applied policy).

---

## 10. Design Decisions & Rationale

- **Entra as IdP with the SP trusting it (vs the reverse).** Chosen because it avoids the public-DNS-domain requirement of inbound external-IdP federation, works on a free tenant, and lets existing RBAC identities flow into the SP unchanged.
- **Both SAML and OIDC, deliberately.** Proving both demonstrates the *concept* of federation rather than one vendor flow, and backs the "SAML/OpenID Connect" skill claim with real artifacts.
- **Group-based assignment over per-user.** Access follows job role, so lifecycle changes propagate through group membership — least privilege and governance, not manual cleanup.
- **NameID = email.** Email is a stable, human-meaningful join key that both protocols carry naturally, simplifying account matching across SPs.
- **Implicit flow for the OIDC test only.** Used solely so jwt.ms could display the token in one hop; production SPAs use the **authorization code flow with PKCE** to keep tokens out of the URL.

---

## 11. Evidence Index

All screenshots are redacted (tenant GUIDs, source IPs, signatures, and secrets removed).

| # | File | Demonstrates |
|---|------|--------------|
| 01 | entra-admin-center-overview | Tenant / Entra admin console |
| 02 | saml-toolkit-app-added | SP added as an enterprise application |
| 03 | saml-sso-config-page | SAML SSO configuration surface |
| 04 | test-user-selected | RBAC clinical-staff test user |
| 05 | entra-idp-values | IdP metadata (login URL, issuer, certificate) |
| 06 | toolkit-sp-values | SP metadata (Entity ID, ACS, SP-initiated URL) |
| 07 | entra-basic-saml-config | Real Identifier / Reply URL / Sign-on URL |
| 08 | attributes-claims | NameID = email + claim mapping |
| 09 | app-assignment | Group-based (RBAC) assignment |
| 10 / 10a | sp-initiated-login-start / entra-login-prompt | SP → IdP federation handoff |
| 10b | error | Real failure (null reference) — troubleshooting evidence |
| 11 | sso-success | Federated login succeeded |
| 12 | decoded-saml-assertion | Validated assertion (Issuer/Audience/NameID/signature) |
| 13 | conditional-access-policy | MFA policy + sign-in-frequency |
| 14 | mfa-required | MFA enforced live |
| 15 / 15a | signin-log-ca-applied / mfa-detail | Audit trail: CA policy + MFA satisfied |
| 16 | oidc-app-registration | OIDC app registered |
| 17 | oidc-id-tokens-enabled | ID-token issuance enabled |
| 18 | oidc-optional-claims | OIDC claim mapping |
| 19 | oidc-decoded-idtoken | Validated ID token (iss/aud/sub/claims/nonce/RS256) |

---

## 12. Compliance Mapping

| Framework | Control | Satisfied by |
|-----------|---------|--------------|
| HIPAA | §164.312(d) Person/entity authentication | Federated authentication via a single trusted IdP |
| HIPAA | §164.312(b) Audit controls | Centralized sign-in logs |
| HIPAA | §164.312(a)(2)(iii) Automatic logoff | Sign-in-frequency session control |
| HIPAA | §164.308(a)(5)(ii)(D) Password management | SSO removes per-app passwords |
| NIST 800-53 | IA-2 / IA-8 | Identification & authentication (org + federated identities) |
| NIST 800-53 | IA-2(1) | MFA via Conditional Access |
| NIST 800-53 | AC-3 / AC-17 | Access enforced through federated trust + group assignment |
| NIST 800-53 | AU-2 / AU-3 | Sign-in event logging |
| ISO/IEC 27001:2022 | A.5.16 / A.5.17 / A.8.5 | Identity management, authentication information, secure authentication |
| ISO/IEC 27001:2022 | A.5.15 | Access control |

---

## 13. Outcomes

- **2 applications federated** to one IdP — **1 SAML, 1 OIDC**.
- **Both protocols** demonstrated and validated end to end.
- **MFA enforced** on federated access; password-only login eliminated.
- **Periodic re-authentication** enforced (automatic-logoff control).
- **Centralized audit trail** produced for federated sign-ins.
- **3 real failure modes** diagnosed and resolved, with documented fixes.

---

## 14. Résumé Bullets

- Federated **Microsoft Entra ID** as the identity provider to both **SAML 2.0 and OpenID Connect** service providers for a regional healthcare scenario — exchanging IdP/SP metadata, mapping `NameID`/claims, and decoding live SAML assertions and OIDC ID tokens (SAML-tracer / jwt.ms) to validate issuer, audience, and signed-claim integrity.
- Enforced **Conditional Access MFA** and a **sign-in-frequency** session control on federated access, eliminating password-only login and addressing automatic-logoff requirements, aligned to **HIPAA §164.312, NIST SP 800-53 IA-2(1), and ISO 27001 A.8.5**.
- Assigned application access by **RBAC job-role group** and authored a **federation troubleshooting record** covering circular metadata dependencies, account-matching/`NameID` mismatches, and session/principal errors — reducing onboarding friction for future integrations.

---

## 15. Skills Demonstrated

SAML 2.0 · OpenID Connect / OAuth 2.0 · identity federation · IdP/SP trust & metadata exchange · claim & attribute mapping · Conditional Access · MFA enforcement · session management · access governance (RBAC) · identity audit logging · assertion & token validation · SSO troubleshooting & root-cause analysis · HIPAA / NIST SP 800-53 / ISO 27001 control mapping

---

## 16. Repository Structure

```
entra-sso-saml-oidc/
├── README.md                       ← this file
├── docs/
│   ├── concepts-primer.md           ← SSO, federation, SAML vs OIDC, AD FS vs Entra
│   ├── walkthrough.md               ← full step-by-step build (SAML + OIDC + hardening)
│   ├── troubleshooting.md           ← failure modes, root causes, fixes
│   └── compliance-mapping.md        ← HIPAA / NIST / ISO control detail
└── screenshots/
    └── 01 … 19                      ← redacted evidence
```

---

## 17. References

- Microsoft Learn — Tutorial: Microsoft Entra SSO integration with the SAML Toolkit
- Microsoft Learn — Enable single sign-on for an enterprise application
- Microsoft Learn — Debug SAML-based single sign-on
- Microsoft Learn — Conditional Access policies and session controls
- Microsoft identity platform — ID tokens and OpenID Connect
- jwt.ms — token decoder

---

*Part of a hands-on IAM portfolio. Scenario (Meridian Health Partners) is fictional; all data is synthetic.*
