# Single Sign-On & Federation with Microsoft Entra ID
### SAML 2.0 + OpenID Connect · Conditional Access MFA · Centralized Audit Logging

Microsoft **Entra ID configured as an enterprise Identity Provider (IdP)**, with applications federated to it over **both** major protocols — **SAML 2.0** and **OpenID Connect (OIDC)** — then hardened with **Conditional Access MFA** and an auditable sign-in trail.

Built end-to-end in a live Entra tenant against a fictional healthcare network, **Meridian Health Partners (MHP)**, so every control maps to a real compliance driver (**HIPAA, NIST SP 800-53, ISO/IEC 27001**).

**Tech:** Microsoft Entra ID · SAML 2.0 · OpenID Connect / OAuth 2.0 · Conditional Access · Microsoft Entra SAML Toolkit · jwt.ms · SAML-tracer

![Federated SSO login succeeded](screenshots/11-sso-success.png)

---

## Table of Contents
1. [Business Context & Problem](#1-business-context--problem)
2. [Objectives](#2-objectives)
3. [Core Concepts](#3-core-concepts)
4. [Architecture](#4-architecture)
5. [Authentication Flows](#5-authentication-flows)
6. [Implementation — Visual Walkthrough](#6-implementation--visual-walkthrough)
7. [Validating the Proof Objects](#7-validating-the-proof-objects)
8. [Real-World Troubleshooting](#8-real-world-troubleshooting)
9. [Security Hardening](#9-security-hardening)
10. [Design Decisions & Rationale](#10-design-decisions--rationale)
11. [Compliance Mapping](#11-compliance-mapping)
12. [Outcomes](#12-outcomes)
13. [Résumé Bullets](#13-résumé-bullets)
14. [Skills Demonstrated](#14-skills-demonstrated)
15. [Repository Structure](#15-repository-structure)

---

## 1. Business Context & Problem

Meridian Health Partners (MHP) is a ~520-employee regional healthcare network across three facilities. Growth by acquisition left a fragmented app landscape — each system with its own login — creating four risks an IAM analyst owns:

| # | Problem | Risk |
|---|---------|------|
| 1 | Password sprawl | Credential reuse → the top healthcare breach vector |
| 2 | Inconsistent access control | Lingering access on role change / termination |
| 3 | No central audit trail | Can't answer "who accessed what, when" |
| 4 | Authentication friction | Lost clinician time at point of care |

**Thesis:** make Entra ID the single, authoritative IdP and federate apps to it — one identity, centralized, MFA-enforced, auditable.

---

## 2. Objectives

- Configure Entra ID as an IdP and federate a **SAML 2.0** service provider.
- Federate a second SP over **OpenID Connect** (both protocols proven).
- Enforce **MFA** on federated access via Conditional Access.
- Apply a **session-lifetime** control (sign-in frequency).
- Produce a **centralized audit trail**.
- Capture and **validate** both proof objects (SAML assertion, OIDC ID token).

---

## 3. Core Concepts

**SSO** is an outcome — one login, many apps — not a product. Every SSO relationship has two roles: the **Identity Provider (IdP)** authenticates and asserts identity (Entra ID); the **Service Provider (SP)** trusts the IdP and consumes its assertion. **Federation** is that trust when IdP and SP are separate systems, established by exchanging **metadata** (endpoints + a signing certificate).

**SAML 2.0 vs OIDC** carry the same trust in different encodings:

| | SAML 2.0 | OpenID Connect |
|---|----------|----------------|
| Format | XML assertion | JSON Web Token (JWT) |
| Built on | Standalone XML | OAuth 2.0 |
| Subject | `NameID` | `sub` / `preferred_username` |
| Audience | `<Audience>` | `aud` |
| Decoded with | SAML-tracer | jwt.ms |

**AD vs AD FS vs Entra ID:** Active Directory is the on-prem *directory*; AD FS was the on-prem *federation server* (legacy IdP); **Entra ID** is the cloud IdP that replaced it. *(Full conceptual detail in [`docs/concepts-primer.md`](docs/concepts-primer.md).)*

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

   Hardening: Group-based assignment (RBAC) · Conditional Access MFA · Sign-in frequency · Sign-in logs
```

Identities and role-based groups are reused from the earlier JML and RBAC projects, so a single identity flows across both protocols.

---

## 5. Authentication Flows

**SAML (SP-initiated):** user hits the SP → SP redirects to Entra with an `AuthnRequest` → Entra authenticates (+ MFA) → signed assertion POSTed to the SP's **ACS URL** → SP validates the signature, reads the `NameID`, grants access.

**OIDC:** browser → Entra `/authorize` with `client_id`, `redirect_uri`, `scope=openid …` → Entra authenticates → signed **ID token (JWT)** returned to the redirect URI → app validates and reads claims.

---

## 6. Implementation — Visual Walkthrough

### Track A — SAML SSO (Entra IdP → SAML Toolkit SP)

Starting in the Microsoft Entra admin center, added the SAML Toolkit as an enterprise application and selected SAML as the SSO method.

![Microsoft Entra admin center](screenshots/01-entra-admin-center-overview.png)
![SAML Toolkit added as an enterprise application](screenshots/02-SAML-toolkit-app-added.png)
![SAML single sign-on configuration page](screenshots/03-saml-sso-config-page.png)

Selected a clinical-staff test user (from the RBAC project); the user's email becomes the assertion's `NameID`.

![Test user selected](screenshots/04-test-user-selected.png)

Performed the bidirectional metadata exchange — IdP values from Entra, SP values from the Toolkit.

![Entra IdP metadata: login URL, issuer, certificate](screenshots/05-entra-idp-values.png)
![SP metadata: Entity ID, ACS URL, SP-initiated login URL](screenshots/06-toolkit-sp-values.png)

Configured Basic SAML Configuration with the SP's real values (must match exactly).

![Basic SAML Configuration with real values](screenshots/07-entra-basic-saml-config.png)

Confirmed `NameID = email` and the attribute claims, then assigned access by RBAC group.

![Attributes & claims, NameID = email](screenshots/08-attributes-claims.png)
![Group-based (RBAC) application assignment](screenshots/09-app-assignment.png)

Ran the SP-initiated login — the SP hands off to Entra, which authenticates and returns the user.

![SP-initiated login start](screenshots/10-sp-initiated-login-start.png)
![Entra sign-in prompt (the SP→IdP handoff)](screenshots/10a-entra-login-prompt.png)
![Federated login succeeded](screenshots/11-sso-success.png)

### Track B — OIDC SSO (Entra IdP → registered app)

Registered the app (App registrations) with `redirect_uri = https://jwt.ms`, enabled ID tokens, and mapped optional claims.

![OIDC app registration](screenshots/16-oidc-app-registration.png)
![ID token issuance enabled](screenshots/17-oidc-id-tokens-enabled.png)
![OIDC optional claims mapped](screenshots/18-oidc-optional-claims.png)

### Track C — Hardening (Conditional Access MFA + audit)

Created a Conditional Access policy requiring MFA (with a sign-in-frequency session control), enforced it, and confirmed it in the sign-in logs.

![Conditional Access policy: require MFA + sign-in frequency](screenshots/13-conditional-access-policy.png)
![MFA enforced at login](screenshots/14-mfa-required.png)
![Sign-in log: Conditional Access policy applied](screenshots/15-signin-log-ca-applied.png)
![Sign-in log: MFA authentication detail](screenshots/15a-signin-log-mfa-detail.png)

---

## 7. Validating the Proof Objects

Both proof objects were captured live and validated field by field.

**SAML assertion** (SAML-tracer): confirmed `<Issuer>` = Entra, `<Audience>` = the SP Entity ID, `<NameID>` = the user's email, and the `<Signature>`.

![Decoded SAML assertion](screenshots/12-decoded-saml-assertion.png)

**OIDC ID token** (jwt.ms): confirmed `iss` (tenant), `aud` (client_id), `sub`/`preferred_username` (the user), the mapped claims, `nonce` match (replay protection), and `alg: RS256`.

![Decoded OIDC ID token](screenshots/19-oidc-decoded-idtoken.png)

**Side by side:** `<Issuer>`↔`iss`, `<Audience>`↔`aud`, `<NameID>`↔`sub`, `<AttributeStatement>`↔claims, signed XML↔signed JWT. Same trust, two encodings.

---

## 8. Real-World Troubleshooting

Production SSO rarely works first try. Three real failures were diagnosed and fixed (full detail in [`docs/troubleshooting.md`](docs/troubleshooting.md)):

1. **Circular metadata dependency** — the IdP withheld its certificate until a relying party was defined, while the SP needed that certificate. Resolved with a **placeholder bootstrap**.
2. **Wrong principal authenticating** — the first login crashed the SP; **decoding the assertion** revealed the `NameID` was an *admin* account (a reused session), not the intended user. Re-ran in an isolated session as the correct user. (Using a privileged account for app SSO is the anti-pattern PAM addresses.)

   ![SP error — null reference, used to diagnose the wrong-principal issue](screenshots/10b-error.png)

3. **Account-matching failure** — the SP needs a local user matching the assertion's `NameID`; ensured the match.

---

## 9. Security Hardening

- **Require MFA** via Conditional Access — password-only access eliminated (NIST IA-2(1)).
- **Sign-in frequency** — periodic re-authentication, addressing HIPAA §164.312(a)(2)(iii) automatic logoff for shared clinical workstations.
- **Report-only first**, then enforced — avoids lockouts.
- **Centralized sign-in logs** — the authoritative audit record.

---

## 10. Design Decisions & Rationale

- **Entra as IdP with the SP trusting it** — avoids the public-DNS requirement of inbound external-IdP federation; works on a free tenant; reuses RBAC identities.
- **Both SAML and OIDC** — demonstrates the federation *concept*, not one vendor flow.
- **Group-based assignment** — access follows job role; lifecycle changes propagate automatically.
- **NameID = email** — a stable join key both protocols carry naturally.
- **Implicit flow for the OIDC test only** — so jwt.ms could display the token; production SPAs use authorization code + PKCE.

---

## 11. Compliance Mapping

| Framework | Control | Satisfied by |
|-----------|---------|--------------|
| HIPAA | §164.312(d) | Federated authentication via one IdP |
| HIPAA | §164.312(a)(2)(iii) | Sign-in-frequency (automatic logoff) |
| HIPAA | §164.312(b) | Centralized sign-in logs |
| NIST 800-53 | IA-2 / IA-8 | Identification & authentication |
| NIST 800-53 | IA-2(1) | MFA via Conditional Access |
| NIST 800-53 | AU-2 / AU-3 | Sign-in event logging |
| ISO 27001 | A.5.16 / A.8.5 | Identity management, secure authentication |

*Full detail in [`docs/compliance-mapping.md`](docs/compliance-mapping.md).*

---

## 12. Outcomes

- **2 apps federated** to one IdP — **1 SAML, 1 OIDC**.
- **Both protocols** validated end to end.
- **MFA enforced**; password-only login eliminated.
- **Periodic re-authentication** enforced.
- **Centralized audit trail** produced.
- **3 real failure modes** diagnosed and resolved.

---

## 13. Résumé Bullets

- Federated **Microsoft Entra ID** as the identity provider to both **SAML 2.0 and OpenID Connect** service providers — exchanging IdP/SP metadata, mapping `NameID`/claims, and decoding live SAML assertions and OIDC ID tokens (SAML-tracer / jwt.ms) to validate issuer, audience, and signed-claim integrity.
- Enforced **Conditional Access MFA** and a **sign-in-frequency** session control, eliminating password-only login and addressing automatic-logoff requirements (**HIPAA §164.312, NIST 800-53 IA-2(1), ISO 27001 A.8.5**).
- Assigned access by **RBAC job-role group** and authored a **federation troubleshooting record** (circular metadata dependency, `NameID` mismatch, session/principal errors).

---

## 14. Skills Demonstrated

SAML 2.0 · OpenID Connect / OAuth 2.0 · identity federation · IdP/SP trust & metadata exchange · claim mapping · Conditional Access · MFA · session management · access governance (RBAC) · identity audit logging · assertion & token validation · SSO troubleshooting · HIPAA / NIST 800-53 / ISO 27001 mapping

---

## 15. Repository Structure

```
entra-sso-saml-oidc/
├── README.md
├── docs/
│   ├── concepts-primer.md
│   ├── walkthrough.md
│   ├── troubleshooting.md
│   └── compliance-mapping.md
└── screenshots/      ← 01–19
```

---

*Part of a hands-on IAM portfolio. Scenario (Meridian Health Partners) is fictional; all data is synthetic.*
