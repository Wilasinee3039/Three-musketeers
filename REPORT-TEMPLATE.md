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

### Finding F-01 — 
| Field | Value |
|---|---|
| CWE | CWE-___ |
| OWASP 2025 | A__ / API__ / LLM__ |
| Severity | Critical / High / Medium / Low |
| Location | file:line or endpoint |
| Reproduction | step-by-step (payload/command) |
| Impact | what an attacker gains |
| Evidence | screenshot / request-response |


### Finding F-01 — Insecure Direct Object Reference (IDOR/BOLA)

| Field | Value |
|---|---|
| CWE | CWE-639 — Authorization Bypass Through User-Controlled Key |
| OWASP 2025 | A01:2025 — Broken Access Control |
| Severity | High |
| Location | `app.py` — `GET /api/notes/<nid>` |
| Reproduction | 1. Start the NoteVault application and log in as Alice (`alice` / `alicepw`). 2. Visit `http://localhost:8080/api/notes/1` to access Alice's own note. 3. Change the note ID to `2` by visiting `http://localhost:8080/api/notes/2`. 4. Before the fix, the application returns the Admin's private note even though Alice is not its owner. |
| Impact | An authenticated attacker can access private notes belonging to other users by manipulating note IDs, resulting in unauthorized disclosure of sensitive information. |
| Evidence | [before-own-note](evidence/F-01-IDOR/before-own-note.png) and [before-admin-note](evidence/F-01-IDOR/before-admin-note.png) |

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



### Fix for F-01 — Insecure Direct Object Reference (IDOR/BOLA)

**Before/After Code:**

```diff
- if not current_user():
+ user = current_user()
+ if not user:
      return jsonify(error="auth required"), 401

  con = db()
  r = con.execute(
-     "SELECT id,owner,title,body FROM notes WHERE id = ?",
-     (nid,)
+     "SELECT id,owner,title,body FROM notes WHERE id = ? AND owner = ?",
+     (nid, user)
  ).fetchone()
  con.close()

- return (jsonify(dict(r)) if r else (jsonify(error="not found"), 404))
+ if not r:
+     return jsonify(error="not found"), 404
+
+ return jsonify(dict(r))
```

**Why this fixes it:**

The original implementation only checked whether the user was authenticated, allowing any logged-in user to access another user's note by changing the note ID.

The fix adds an ownership check directly to the SQL query using `WHERE id = ? AND owner = ?`. This ensures that only the owner of a note can retrieve it. Unauthorized requests return `404 Not Found` without exposing private information.

**Commit:** `8188e9f` — `fix(F-01): prevent unauthorized note access (IDOR)`

**Proof the exploit now fails:**

After applying the fix and rebuilding the application, the endpoint was tested again using Alice's account.

- `/api/notes/1` → Alice can still access her own note.
- `/api/notes/2` → Returns `{"error":"not found"}` instead of exposing the Admin's private note.

**Evidence:**
- [after-own-note](evidence/F-01-IDOR/after-own-note.png)
- [after-admin-note](evidence/F-01-IDOR/after-admin-note.png)

**Result:** The IDOR vulnerability was successfully remediated, and unauthorized note access was prevented.

---

## 6. Supply-Chain Hardening  *(15 pts)*

### 6.1 SBOM (CycloneDX)

The SBOM was generated from the built container image with Syft 1.55.0 (CycloneDX 1.7 JSON): [`sbom.json`](./sbom.json).

```bash
docker build -t notevault:latest .
SYFT_FILE_METADATA_SELECTION=none syft notevault:latest -o cyclonedx-json=sbom.json
jq '.components | length' sbom.json   # 490
```

| Component type | Count |
|---|---|
| library (Debian OS packages + Python packages) | 482 |
| application | 7 |
| operating-system | 1 |
| **Total** | **490** |

Of these, 15 are Python packages, including the six declared in `requirements.txt` (Flask 2.0.1, Werkzeug 2.0.1, Jinja2 3.0.1, PyJWT 1.7.1, requests 2.25.1, urllib3 1.26.4). The large OS-package count comes from the unpinned, full `python:3.12` base image, which is most of the image's attack surface.

### 6.2 Image signed & verified with Cosign

The image was pushed to a local OCI registry and signed by digest with a Cosign key pair (Cosign v3.1.4). The public key is committed as [`cosign.pub`](./cosign.pub); the private key is git-ignored.

```bash
cosign generate-key-pair
cosign sign --key cosign.key localhost:5001/notevault@sha256:0911be0a565e420a8440eaa911ae5372421f9b20039a912dcf3f89da17b8f27e
cosign verify --key cosign.pub --insecure-ignore-tlog=true localhost:5001/notevault@sha256:0911be0a565e420a8440eaa911ae5372421f9b20039a912dcf3f89da17b8f27e
```

`cosign verify` output ([evidence](./evidence/supply-chain/cosign-verify.txt)):

```
Verification for localhost:5001/notevault@sha256:0911be0a565e420a8440eaa911ae5372421f9b20039a912dcf3f89da17b8f27e --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key

[{"critical":{"identity":{"docker-reference":"localhost:5001/notevault@sha256:0911be0a565e420a8440eaa911ae5372421f9b20039a912dcf3f89da17b8f27e"},"image":{"docker-manifest-digest":"sha256:0911be0a565e420a8440eaa911ae5372421f9b20039a912dcf3f89da17b8f27e"},"type":"https://sigstore.dev/cosign/sign/v1"},"optional":{}}]
```

Negative check: verifying the same image with an unrelated public key fails ([evidence](./evidence/supply-chain/cosign-verify-wrong-key.txt)):

```
Error: no matching attestations: failed to verify signature: could not verify envelope: accepted signatures do not match threshold, Found: 0, Expected 1
```

Because the registry was local and offline, signing skipped the public Rekor transparency log (`--tlog-upload=false`), so verification uses `--insecure-ignore-tlog=true`.

### 6.3 SLSA self-assessment

We assess NoteVault at **SLSA Build Level 1** at best (effectively L0–L1). The build is fully scripted in a Dockerfile, and we publish a CycloneDX SBOM and a Cosign signature, so a consumer can check what the image contains and that it has not been changed since we signed it. We do not reach Level 2: the image is built and signed on a developer workstation rather than a hosted build platform, there is no signed provenance attestation produced by the build service (the signature proves *who* signed, not *how* the image was built), and signing used a long-lived local key without a transparency-log entry. The base image is also unpinned (`python:3.12`, not a digest), so rebuilds are not reproducible. Moving the build into GitHub Actions with keyless (OIDC) Cosign signing, a Rekor entry, and `actions/attest-build-provenance` would reach **Level 2**. Using the isolated `slsa-github-generator` reusable workflow would reach **Level 3**.

---

## 7. Security CI Pipeline  *(10 pts)*

### 7.1 Workflow

The pipeline is defined in [`.github/workflows/security.yml`](./.github/workflows/security.yml).
After pushing the file, runs are available under [GitHub Actions — Security CI](https://github.com/Wilasinee3039/Three-musketeers/actions/workflows/security.yml).
It runs on pushes, pull requests, and manual dispatches. Three independent jobs run the following checks:

| Check | Tool | Failure condition |
|---|---|---|
| SAST | Bandit 1.8.3 | At least one HIGH severity finding, at any confidence level. Bandit has no CRITICAL severity category. |
| SCA | pip-audit 2.9.0 | Any known dependency vulnerability, including HIGH/CRITICAL; this is stricter than a HIGH-only gate. Dependency collection errors also fail the audit through `--strict`. |
| Secret scanning | Gitleaks 8.24.2 | Any secret detected by the default rules in Git history. Output is redacted. Pattern scanning does not guarantee detection of every hardcoded password. |

Each scanner returns a non-zero exit code on a finding that meets its gate.
No scan uses `continue-on-error` or `|| true`, so a failed job makes the workflow fail.
These checks do not block merging automatically: repository branch protection or a ruleset must separately require the three job checks.

### 7.2 Failed-build evidence

![GitHub Actions Security CI failing on HIGH severity Bandit findings](./img/security-ci-failed.png)

**Figure 7.1:** GitHub Actions ran the **SAST - Bandit (HIGH gate)** job and reported **6 HIGH severity findings**. The screenshot shows B201 (`flask_debug_true`) at `app.py:209`, where `debug=True` enables the Flask debugger. The scan ended with **Process completed with exit code 1**, causing the job to fail. This demonstrates that the HIGH severity security gate works as configured.

The job list also shows that the SCA job failed and the Gitleaks secret-scanning job passed. The SCA failure reason must be checked in its own log; this screenshot only shows the detailed SAST findings.

- **Failed run URL:** (https://github.com/Wilasinee3039/Three-musketeers/actions/runs/39925428245/job/120320512106)
- **Screenshot:** [`img/security-ci-failed.png`](./img/security-ci-failed.png)

Local verification on October 23, 2026 with Python 3.12.2 and Bandit 1.8.3 also returned exit code **1** and reported **6 HIGH findings**: four B324 weak-MD5 findings, B602 shell execution at `app.py:202`, and B201 Flask debug mode at `app.py:209`.

The local pip-audit 2.9.0 run returned exit code **1**, reporting **47 known vulnerabilities in 7 packages**, including transitive dependencies. Counts may change as the advisory database changes.

After remediation, rerun the same workflow and retain the passing run as additional evidence.

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
