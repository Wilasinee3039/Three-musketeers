# Term Project Report — Secure Build

> Fill in every section. Keep the headings. Total **100 pts** (see rubric at the end).
> This is **team work** (2–3). Do not share your report with other teams.

## Cover

| Field | Value |
|---|---|
| **Team name** | Three Musketeers (Team 11) |
| **Members (name + ID)** | Sai Seng Main — 6631503085 |
| | Wilasinee Mangkorn — 6631503039 |
| | Pirisa Kitichai — 6631503031 |
| **Target application** | NoteVault |
| **Repository URL** | [https://github.com/Wilasinee3039/Three-musketeers] |
| **Date** | October 23, 2026 |

## AI-tool usage disclosure
*Academic integrity: state how you used any AI tools (ChatGPT, Copilot, etc.) — e.g. searching, coding, translating, drafting. "None" is a valid answer.*

| Tool | How it was used |
|---|---|
| | |

---

## 1. Executive Summary
*(½ page) What the app is, how many findings by severity, and the overall risk before vs. after your fixes.*

---

## 2. System Scope & Function

### 2.1 Application Overview

NoteVault is a web-based note management and sharing application developed using Python Flask. It allows users to register, log in, create and view personal notes, and search their notes. The application also provides an administrative endpoint for listing users and an export functionality.

### 2.2 System Components

- **Client:** A web browser used to interact with the application.
- **Backend:** A Python Flask application that handles HTTP requests, authentication, note management, search, and administrative functions.
- **Database:** SQLite, which stores user accounts and note information in two tables: `users` and `notes`.
- **Authentication:** JWT-based authentication using a session cookie.
- **Operating Environment:** A locally hosted application intended for security testing.

### 2.3 In-Scope Components

The security assessment covers:

1. User registration and login.
2. Session and authentication management.
3. Note creation, retrieval, and access control.
4. Note searching.
5. Administrative access.
6. Note export functionality.
7. Application dependencies, Docker configuration, and the build/release pipeline.

### 2.4 Data-Flow Diagram

The Data-Flow Diagram illustrates how user requests are processed by the Flask backend, how information is stored and retrieved from the SQLite database, and where trust boundaries exist between the client, application, database, and operating system.

The primary trust boundaries are between the user's browser and the application server, and between application-controlled inputs and sensitive backend resources.

![NoteVault Data Flow Diagram](./img/Data%20Flow%20Diagram.png)

---

## 3. Threat Model — STRIDE  *(20 pts)*

| Element / Data Flow | S | T | R | I | D | E | Top Threat & Mitigation |
|---|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1.0 Authentication / JWT | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | Hardcoded JWT secret, SQL injection, and weak MD5 password hashing. Use environment variables for secrets, parameterized queries, and Argon2id. |
| 2.0 Manage Notes / API | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | IDOR and stored XSS. Enforce note ownership checks and escape user-controlled HTML content. |
| 3.0 Search Notes | — | ✓ | — | ✓ | ✓ | — | SQL injection through search input. Use parameterized queries. |
| 4.0 Admin Functions | ✓ | ✓ | ✓ | ✓ | — | ✓ | Privilege escalation through user-controlled role assignment. Enforce server-side role assignment and authorization checks. |
| 5.0 Export / OS Shell | — | ✓ | — | ✓ | ✓ | ✓ | OS command injection through the export format parameter. Remove shell execution and validate allowed formats. |
| D1: Users / SQLite | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | Weak password hashing and sensitive account data exposure. Use Argon2id and restrict access to sensitive records. |
| D2: Notes / SQLite | — | ✓ | ✓ | ✓ | ✓ | — | Unauthorized note access. Enforce object-level authorization and ownership validation. |

**Note:** ✓ indicates a potential threat identified through source-code review, not a confirmed exploit.

### Top 5 Risks Ranked by Likelihood × Impact

1. **R-01: Remote Command Execution via /export (Risk Score: 25)**
   - Likelihood: 5, Impact: 5
   - Untrusted input passed directly to subprocess.run(..., shell=True) gives attackers total control of the host.
   - Mitigation: Remove shell execution; enforce a strict whitelist of export formats.

2. **R-02: SQL Injection in Authentication & Search (Risk Score: 25)**
   - Likelihood: 5, Impact: 5
   - Direct string formatting in SQL queries allows login bypass and full database dumping.
   - Mitigation: Use parameterized queries (prepared statements).

3. **R-03: Privilege Escalation via Mass Assignment on Registration (Risk Score: 20)**
   - Likelihood: 4, Impact: 5
   - /register accepts a role parameter without validation, letting attackers create admin accounts.
   - Mitigation: Hardcode newly registered accounts to role `"user"`.

4. **R-04: Broken Object-Level Authorization (BOLA/IDOR) on /api/notes/<nid> (Risk Score: 20)**
   - Likelihood: 5, Impact: 4
   - Any logged-in user can access any note ID without an ownership check, exposing admin secrets.
   - Mitigation: Verify owner == current_user before returning note contents.

5. **R-05: Stored Cross-Site Scripting (XSS) in Note Rendering (Risk Score: 12)**
   - Likelihood: 4, Impact: 3
   - Note body rendered with |safe allows JavaScript execution in victims' browsers.
   - Mitigation: Enable auto-escaping in templates and sanitize HTML.
---

## 4. Vulnerability Findings  *(25 pts)*

One block per finding (copy as needed):

### Finding F-01 — <title>
| Field | Value |
|---|---|
| CWE | CWE-___ |
| OWASP 2025 | A__ / API__ / LLM__ |
| Severity | Critical / High / Medium / Low |
| Location | file:line or endpoint |
| Reproduction | step-by-step (payload/command) |
| Impact | what an attacker gains |
| Evidence | screenshot / request-response |

---

## 5. Remediation  *(25 pts)*

Per finding: the fix, **before/after** code, and the commit that implements it.

### Fix for F-01
```diff
- vulnerable line
+ fixed line
```
- **Why this fixes it:** …
- **Commit:** `<hash>`
- **Proof the exploit now fails:** (screenshot)

---

## 6. Supply-Chain Hardening  *(15 pts)*
- **SBOM** (CycloneDX) generated — attach `sbom.json`, note component count.
- **Artifact signed & verified** with Cosign — paste the `cosign verify` output.
- One-paragraph **SLSA** self-assessment (which level, why).

---

## 7. Security CI Pipeline  *(10 pts)*
- Link to the GitHub Actions workflow (SAST + SCA + secret scanning).
- Screenshot of a build **failing** on a HIGH/CRITICAL finding.
- How the gate is configured (what fails the build).

---

## 8. Conclusion & Reflection  *(5 pts — clarity)*
- Residual risks and what you'd harden next.
- Lessons learned.

---

## Appendix
- Tools used, references (OWASP/CWE links), and the team's task split.

---

## Grading rubric (100)

| Criterion | Pts |
|---|---|
| Threat model quality & completeness (§3) | 20 |
| Vulnerability findings — correctness, CWE/OWASP mapping, depth (§4) | 25 |
| Remediation quality — correct, minimal, well-explained (§5) | 25 |
| Supply-chain hardening — SBOM + signing + provenance (§6) | 15 |
| CI pipeline — works, fails build appropriately (§7) | 10 |
| Presentation & report clarity (§1, §8) | 5 |

*All work stays within the [ethics policy](../ETHICS.md).*
