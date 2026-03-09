# Lab 5 — SAST and DAST Security Analysis of OWASP Juice Shop

## Task 1 — SAST with Semgrep

For static analysis I used Semgrep in a Docker container against the Juice Shop v19.0.0 source code checked out into `labs/lab5/semgrep/juice-shop`. The scan ran with the `p/security-audit` and `p/owasp-top-ten` rule packs and produced both JSON and text reports.

Semgrep covered the main application codebase: server‑side routes, API handlers, business logic, and a large portion of the frontend. In total it reported a few dozen findings spread across several categories: injection patterns, insecure use of crypto and JWTs, potentially unsafe redirects, and places where user input is not clearly validated or sanitised. The majority of findings were low or medium severity, but there were also several high and critical issues worth calling out.

### 1.1 SAST tool effectiveness

Semgrep is a good fit for this project for a few reasons:

- It understands common JavaScript/TypeScript idioms used in Juice Shop and can match real code patterns rather than just doing regex searches.
- The rule packs used align with the OWASP Top 10 and general security best practices, so the findings map nicely to familiar vulnerability categories.
- The JSON output format is structured and easy to post‑process or feed into other tooling, while the text report is convenient for quick manual review.

In practice the scan surface felt “wide enough”: it flagged risky patterns in authentication, session handling, request processing, and some of the challenge‑related code paths. As usual with SAST, there are some false positives and noisy rules, but as a first security gate in CI this would already catch a lot of obvious mistakes before they ever hit a running environment.

### 1.2 Five most critical Semgrep findings

Below are five of the most important issues highlighted by Semgrep, grouped by type and with enough context to act on them:

1. **Potential SQL injection in product search**
   - **Type:** SQL Injection (tainted user input flowing into a query)
   - **Location:** search implementation for `/rest/products/search` (server‑side route handling the `q` parameter)
   - **Severity:** High / Critical
   - **Why it matters:** User‑controlled input from the search box influences how products are fetched. If the query is not normalised or parameterised correctly, an attacker can pivot from a harmless search into extracting or modifying data in the backing database.

2. **Insecure JWT handling for login**
   - **Type:** Insecure authentication / JWT configuration
   - **Location:** logic around `/rest/user/login` where JWTs are created and verified
   - **Severity:** High
   - **Why it matters:** Juice Shop intentionally contains weak JWT configuration for training purposes. Semgrep rules targeting JWT usage flag places where tokens are signed with weak or hard‑coded secrets, or where algorithms are not pinned strictly enough, which can lead to token forgery and account takeover.

3. **Hard‑coded secrets and API keys**
   - **Type:** Hard‑coded credential / secret in source code
   - **Location:** configuration and challenge‑related files (e.g. tokens, sample API keys, or passwords embedded directly in code)
   - **Severity:** High
   - **Why it matters:** Even in a demo project, hard‑coded secrets are a realistic anti‑pattern. In a real system, committing keys or passwords means they are now effectively public and hard to rotate. Semgrep’s rules for “hardcoded-password” and “generic-secret” are valuable here because they catch this early in the development cycle.

4. **Insecure cryptography usage**
   - **Type:** Weak or misused cryptography
   - **Location:** utility functions dealing with hashing, token generation, and challenge scoring
   - **Severity:** High
   - **Why it matters:** Some helpers in Juice Shop use outdated or non‑ideal crypto primitives on purpose to make challenges solvable. Semgrep flags, for example, use of weak hash functions in security‑sensitive contexts or misuse of randomness. In a real app these patterns would make brute‑forcing or prediction much easier.

5. **Unvalidated redirects / open redirects**
   - **Type:** Open redirect / unvalidated redirect
   - **Location:** code paths where the application sends the user to a URL derived from request parameters
   - **Severity:** Medium / High (depending on context)
   - **Why it matters:** If an attacker can control redirect targets, they can chain this into phishing, cookie theft, or other social‑engineering style attacks. Semgrep’s web‑focused rules do a good job of pointing at suspicious redirect patterns that deserve a manual review.

Overall, Semgrep gives strong “shift‑left” coverage: it makes it very clear where dangerous patterns live in the code, even before you ever run the application.

---

## Task 2 — DAST with ZAP, Nuclei, Nikto, and SQLmap

For dynamic analysis the target was the same Juice Shop instance exposed at `http://localhost:3000`. The idea was to look at the application from the outside, as an attacker would see it, using multiple tools that each specialise in a slightly different slice of the problem: a full web scanner (ZAP), a template‑based scanner (Nuclei), a classic web‑server scanner (Nikto), and a deep SQL injection engine (SQLmap).

### 2.1 ZAP: authenticated vs unauthenticated scanning

The first step was to run a **baseline ZAP scan without authentication**. This version of the scan crawls and probes whatever is reachable to an anonymous user: the home page, product catalogue, registration and login flows, password reset, and other public routes. It quickly finds common issues such as missing or weak security headers, easily discoverable admin clues in HTML/JS, and some reflective inputs. URL coverage in this mode is limited to what a typical unauthenticated visit would trigger.

Then, using ZAP’s automation framework and a YAML configuration, I ran a **fully authenticated scan**. The ZAP script logs in with the default admin credentials (`admin@juice-sh.op` / `admin123`), verifies that authentication succeeded by checking the JSON response, and then launches the spider and AJAX spider followed by active scanning and report generation. Compared to the baseline:

- The authenticated scan discovers **an order of magnitude more URLs**, thanks to the AJAX spider following JavaScript‑driven navigation and logged‑in features.
- New URL families appear, for example:
  - `/rest/admin/application-configuration`
  - `/rest/admin/application-version`
  - `/rest/admin/backup`
  - `/rest/basket/*`, `/rest/orders/*`, and other user‑specific endpoints
- ZAP starts reporting issues in areas that simply did not exist in the unauthenticated view: admin APIs, account management, and state‑changing operations behind the login wall.

This clearly shows why authenticated DAST matters: the most interesting and impactful vulnerabilities often live in the parts of the app that require a logged‑in session. Without authentication, a scanner only scratches the surface.

### 2.2 Tool comparison matrix

The following table summarises how each DAST tool contributed, at a high level, and what it is best suited for:

| Tool   | Typical Findings (examples)                                                                 | Severity mix (approximate)             | Best use case                                                    |
|--------|----------------------------------------------------------------------------------------------|----------------------------------------|------------------------------------------------------------------|
| ZAP    | Missing security headers, XSS candidates, insecure cookies, weak login/HTTPS settings      | Many Low/Medium, some High             | Broad web app scanning with and without authentication           |
| Nuclei | Known CVE patterns, misconfigurations, interesting files and endpoints                      | Mostly Medium, a few High              | Fast checks for known issues using community templates           |
| Nikto  | Server banner leaks, default files, outdated/probing server behaviour                       | Mostly Low/Medium                      | Web server hardening and misconfiguration checks                 |
| SQLmap | SQL injection on search and login endpoints, database enumeration, extracted user records   | High and Critical only (when found)    | Deep, targeted SQL injection testing on specific HTTP endpoints  |

Together these tools give a much more complete picture than any one of them alone: ZAP gives breadth, Nuclei brings in community knowledge about known bugs and misconfigurations, Nikto looks at the underlying HTTP server behaviour, and SQLmap goes very deep on one class of vulnerability.

### 2.3 Tool‑specific strengths and example findings

- **ZAP**
  - **Strengths:** Full featured web scanner with good crawling, active scanning, authentication support, and HTML reports. Easy to automate via Docker and YAML.
  - **Example findings:** Missing or weak security headers (e.g. `X-Frame-Options`, `Content-Security-Policy`), cookies without `Secure`/`HttpOnly`, and potential reflected inputs in several query parameters.

- **Nuclei**
  - **Strengths:** Very fast template‑based engine with a large, regularly updated template set. Great at quickly answering “am I affected by any of these known CVEs or misconfigurations?”
  - **Example findings:** Generic misconfiguration checks such as verbose server banners and generic disclosure endpoints, plus checks for common framework‑related issues that match Juice Shop’s tech stack.

- **Nikto**
  - **Strengths:** Focused specifically on web server issues. It has a long history and a large database of default files, legacy scripts, and dangerous options.
  - **Example findings:** Warnings about server identification, lack of certain security headers at the HTTP layer, and checks for default or sample files that could leak information.

- **SQLmap**
  - **Strengths:** Purpose‑built for SQL injection and database takeover. Supports multiple techniques (boolean/time‑based blind, error‑based, etc.), can fingerprint DBMS and automate data extraction.
  - **Example findings:** Confirmed SQL injection in the product search endpoint (`/rest/products/search?q=`) and in the JSON login endpoint (`/rest/user/login` when the email parameter is manipulated). Once the injection is confirmed, SQLmap is able to enumerate the backing SQLite database and dump user email addresses with password hashes, clearly demonstrating impact.

---

## Task 3 — SAST vs DAST Correlation and Assessment

SAST and DAST look at the same application from two very different angles. In this lab they complemented each other nicely.

### 3.1 Finding patterns and gaps

From the SAST side, Semgrep highlighted a broad set of code‑level issues: insecure patterns in authentication, hard‑coded secrets, suspicious data flows into queries, and weak crypto usage. On the DAST side, the tools focused on what is observable from the outside: how the app responds over HTTP, which URLs exist and are reachable, how headers and cookies are configured, and whether input validation holds up under active probing.

Several important vulnerabilities show up in both worlds. For example, the risky SQL patterns in the search and login logic that Semgrep flags as potential injection points are the same endpoints that SQLmap is later able to exploit from the outside. That correlation is powerful: the static analysis tells you where to fix the code, while the dynamic analysis proves that the issue is exploitable end‑to‑end.

At the same time, each approach finds things the other cannot:

- **SAST‑only types:**
  - Hard‑coded secrets and test keys committed in the repository.
  - Insecure use of crypto primitives in helper functions that might not be reachable in a typical HTTP workflow.
  - Internal code smells and dangerous patterns that are not wired up to a live route yet but would be problematic if exposed later.

- **DAST‑only types:**
  - Missing or weak HTTP security headers and cookie flags that are added at deployment time.
  - Authentication and session management issues that only appear when you execute a full login and navigation flow.
  - Configuration and server‑side misconfigurations (e.g. verbose banners, error pages) that are not visible in source.

### 3.2 Overall assessment and recommendations

Putting everything together, the picture is exactly what you would expect from a deliberately vulnerable training app: there are plenty of opportunities to practise fixing injection issues, hardening authentication, cleaning up secrets, and enforcing best practices at the HTTP layer. The main lesson, though, is not just “Juice Shop has bugs” — it is how different tools reveal different parts of the story.

My recommended approach for a real project that looks like this would be:

- Run **Semgrep** early and often in the development pipeline to prevent obvious security mistakes from being merged.
- Run **ZAP** (and optionally Nuclei/Nikto) against staging environments to keep web‑level misconfigurations and common web vulnerabilities under control.
- Use **SQLmap** sparingly but deliberately when code or SAST results suggest there might be injection risks in critical data flows.
- Always correlate SAST and DAST findings: when both flag the same area, prioritise it highly; when only one side sees a problem, treat it as a prompt to look closer from the other angle.

Using both SAST and DAST this way gives much stronger coverage than either technique alone and fits well into an iterative DevSecOps workflow.