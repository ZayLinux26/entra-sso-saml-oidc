# Compliance Control Mapping

How each control implemented in this project maps to the relevant regulatory and standards frameworks, with the reasoning for each mapping. The scenario (Meridian Health Partners) is a healthcare network, so HIPAA is the primary driver, supported by NIST SP 800-53 and ISO/IEC 27001.

---

## HIPAA Security Rule

| Control | Requirement | How this project satisfies it |
|---------|-------------|-------------------------------|
| §164.312(d) | **Person or entity authentication** — verify identity before access | Federated authentication routes every login through a single trusted IdP (Entra), with verified, signed assertions/tokens |
| §164.312(a)(2)(i) | **Unique user identification** | One authoritative identity per user, carried across SPs via `NameID` / `sub` |
| §164.312(a)(2)(iii) | **Automatic logoff** | Conditional Access **sign-in frequency** forces periodic re-authentication — essential for shared clinical workstations |
| §164.312(b) | **Audit controls** | Centralized Entra sign-in logs record who accessed which app, when, from where, and whether MFA/policy applied |
| §164.308(a)(5)(ii)(D) | **Password management** | SSO removes per-app passwords, eliminating reuse and sprawl |
| §164.308(a)(4) | **Information access management** | Access granted by role-based group, aligned to job function |

---

## NIST SP 800-53

| Control | Title | How satisfied |
|---------|-------|---------------|
| IA-2 | Identification and authentication (organizational users) | Entra authenticates all federated logins |
| IA-2(1) | MFA to privileged/network accounts | Conditional Access requires MFA on federated access |
| IA-8 | Identification and authentication (non-organizational users) | Federated identity handling via standard protocols |
| IA-5 | Authenticator management | Credentials centralized at the IdP; MFA registered/enforced |
| AC-3 | Access enforcement | Access mediated by federated trust and group assignment |
| AC-17 | Remote access | Cloud-based access governed by Conditional Access |
| AU-2 | Event logging | Sign-in events captured centrally |
| AU-3 | Content of audit records | Logs include user, app, result, applied policy, MFA method |

---

## ISO/IEC 27001:2022 (Annex A)

| Control | Title | How satisfied |
|---------|-------|---------------|
| A.5.15 | Access control | Group/role-based application assignment |
| A.5.16 | Identity management | Single managed identity across applications |
| A.5.17 | Authentication information | Centralized, signed-assertion/token authentication |
| A.8.5 | Secure authentication | MFA + federation + session controls |

---

## Control coverage summary

| Capability built | HIPAA | NIST 800-53 | ISO 27001 |
|------------------|-------|-------------|-----------|
| Federated authentication (SSO) | §164.312(d) | IA-2, IA-8 | A.5.16, A.5.17 |
| MFA enforcement | — | IA-2(1) | A.8.5 |
| Session re-authentication | §164.312(a)(2)(iii) | AC-17 | A.8.5 |
| Group-based access | §164.308(a)(4) | AC-3 | A.5.15 |
| Centralized audit logging | §164.312(b) | AU-2, AU-3 | A.5.16 |
| Password reduction (SSO) | §164.308(a)(5)(ii)(D) | IA-5 | A.5.17 |

---

*Mappings illustrate how technical controls support compliance objectives; they are not a substitute for a formal risk assessment or audit.*
