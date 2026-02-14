## Task 1 — Threagile Baseline Model

### 1.1 Generation & Outputs

- **Model:** `labs/lab2/threagile-model.yaml`
- **Output directory:** `labs/lab2/baseline/`
- **Generated artifacts:** `risks.json`, `stats.json`, `technical-assets.json`

> **Note:** On some systems (e.g., Windows + Docker), Fontconfig errors may prevent `report.pdf` and diagram PNGs. The JSON outputs contain all risk data. If you need the PDF, run Threagile on Linux/WSL or with a writable font cache.

### 1.2 Risk Ranking Methodology

**Composite score formula:** `Severity×100 + Likelihood×10 + Impact`

| Dimension | Values |
|-----------|--------|
| **Severity** | critical (5) > elevated (4) > high (3) > medium (2) > low (1) |
| **Likelihood** | very-likely (4) > likely (3) > possible (2) > unlikely (1) |
| **Impact** | high (3) > medium (2) > low (1) |

**Example:** Unencrypted Communication (Direct to App): Severity=elevated(4), Likelihood=likely(3), Impact=high(3) → **Composite = 4×100 + 3×10 + 3 = 433**

### 1.3 Top 5 Risks (Baseline)

| Rank | Severity | Category | Asset / Link | Likelihood | Impact | Composite |
|-----:|----------|----------|--------------|------------|--------|----------:|
| 1 | Elevated | Unencrypted Communication | Direct to App (User Browser → Juice Shop) | Likely | High | **433** |
| 2 | Elevated | Cross-Site Scripting (XSS) | Juice Shop Application | Likely | Medium | **432** |
| 3 | Elevated | Missing Authentication | To App (Reverse Proxy → Juice Shop) | Likely | Medium | **432** |
| 4 | Elevated | Unencrypted Communication | To App (Reverse Proxy → Juice Shop) | Likely | Medium | **432** |
| 5 | Medium | Cross-Site Request Forgery (CSRF) | Via Direct to App / To App | Very-likely | Low | **241** |

### 1.4 Critical Security Concerns

1. **Unencrypted traffic:** User-to-app and proxy-to-app links use HTTP. Credentials and session tokens can be eavesdropped on the network (MitM risk).
2. **XSS:** Juice Shop is a deliberately vulnerable app; unescaped user input in product reviews enables stored XSS.
3. **Missing authentication on internal link:** Reverse Proxy → Juice Shop has no authentication, enabling impersonation if an attacker reaches the internal network.
4. **CSRF:** No CSRF tokens on state-changing requests; attackers can forge requests from malicious sites.

---

## Task 2 — HTTPS Variant & Risk Comparison

### 2.1 Secure Model Changes

| Change | Original | Secure |
|--------|----------|--------|
| User Browser → **Direct to App** | `protocol: http` | `protocol: https` |
| Reverse Proxy → **To App** | `protocol: http` | `protocol: https` |
| **Persistent Storage** | `encryption: none` | `encryption: transparent` |

- **Model:** `labs/lab2/threagile-model.secure.yaml`
- **Output directory:** `labs/lab2/secure/`

### 2.2 Risk Category Delta Table

| Category | Baseline | Secure | Δ |
|----------|---------:|-------:|---:|
| container-baseimage-backdooring | 1 | 1 | 0 |
| cross-site-request-forgery | 2 | 2 | 0 |
| cross-site-scripting | 1 | 1 | 0 |
| missing-authentication | 1 | 1 | 0 |
| missing-authentication-second-factor | 2 | 2 | 0 |
| missing-build-infrastructure | 1 | 1 | 0 |
| missing-hardening | 2 | 2 | 0 |
| missing-identity-store | 1 | 1 | 0 |
| missing-vault | 1 | 1 | 0 |
| missing-waf | 1 | 1 | 0 |
| server-side-request-forgery | 2 | 2 | 0 |
| **unencrypted-asset** | **2** | **1** | **-1** |
| **unencrypted-communication** | **2** | **0** | **-2** |
| unnecessary-data-transfer | 2 | 2 | 0 |
| unnecessary-technical-asset | 2 | 2 | 0 |

### 2.3 Delta Run Explanation

**Model changes:**
1. **Direct to App:** `protocol: http` → `https` — TLS for direct browser-to-app traffic.
2. **Reverse Proxy → To App:** `protocol: http` → `https` — TLS for proxy-to-app traffic.
3. **Persistent Storage:** `encryption: none` → `transparent` — transparent encryption at rest.

**Observed impact:**
- **unencrypted-communication:** 2 → 0 (−2). Both communication links are now modeled as encrypted, so Threagile no longer flags them as unencrypted.
- **unencrypted-asset:** 2 → 1 (−1). Persistent Storage with `encryption: transparent` is no longer considered unencrypted; Juice Shop remains unencrypted in memory.

**Interpretation:**  
These changes directly address transport and storage encryption. They reduce eavesdropping risk and partially mitigate data-at-rest risk. Other categories (XSS, CSRF, SSRF, authentication, hardening) are unchanged and require different controls (input validation, CSRF tokens, WAF, etc.).

---

## Diagrams & References

- **Baseline:** `labs/lab2/baseline/` — `risks.json`, `stats.json`, `technical-assets.json`
- **Secure:** `labs/lab2/secure/` — same structure

---
