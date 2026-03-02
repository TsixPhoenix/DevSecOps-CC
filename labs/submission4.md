# Lab 4 — SBOM and SCA (Juice Shop v19.0.0)

## Task 1 — SBOM generation (Syft and Trivy)

The image was scanned using Syft and Trivy in Docker against `bkimminich/juice-shop:v19.0.0`. The results are stored under `labs/lab4/syft/` and `labs/lab4/trivy/`.

**Package types.**  
Syft’s table shows a single consolidated list of components, mostly npm packages (name and version), with duplicates explicitly marked. In the JSON, the artifact type is in the `type` field (npm, deb, etc.). Trivy’s report is grouped by target: a Debian layer with 27 vulnerabilities, plus separate entries for each `package.json` (node-pkg). In practice Syft is more convenient for a “flat” SBOM by components, while Trivy is better when you care about which layer or `package.json` a dependency comes from.

**Dependencies.**  
Both tools find a large number of npm dependencies. Syft’s table shows name and version without an explicit direct/transitive split. Trivy shows the path to the `package.json` (for example `juice-shop/node_modules/...`), which makes it easier to see where a package is coming from. For a classic SBOM with a dependency tree the Syft JSON has a bit more structure; for a quick “what is inside this image” overview, the Trivy table is very readable.

**Licenses.**  
In Syft, licenses are listed in `artifacts[].licenses[]`, so every component has its own license list, which is handy for “component → license” style reports. Trivy can run a dedicated license scan (`--scanners license`), with results in `trivy-licenses.json` as a license list per package. For compliance checks both are usable; Syft feels more SBOM-centric, while Trivy focuses on one combined image report.

---

## Task 2 — SCA (Grype and Trivy)

**Grype.**  
Grype was run against the Syft SBOM: `grype sbom:.../juice-shop-syft-native.json`. The results are in `labs/lab4/syft/grype-vuln-table.txt` and `grype-vuln-results.json`. The table contains dozens of entries with package name, installed version, fixed version, type, vulnerability ID, severity and EPSS. In practice Grype showed the same critical issues as Trivy.

**Five most important findings (from Grype/Trivy reports):**

1. **vm2 3.9.17** (Critical) — multiple sandbox escape issues (several GHSAs). Recommendation: upgrade to 3.9.18 or 3.10.x depending on the advisory. In production I would avoid vm2 entirely or at least isolate untrusted code in a separate process/container.

2. **jsonwebtoken 0.1.0 / 0.4.0** (Critical) — algorithm confusion and key substitution issues. Fix: move to 4.2.2 or 9.x. Updating is mandatory because it directly affects authentication and token validation.

3. **lodash 2.4.2** (Critical) — prototype pollution. Upgrading to 4.17.12+ significantly reduces the risk. For an app like Juice Shop this is a very typical building block for exploit chains.

4. **crypto-js 3.3.0** (Critical) — known weaknesses in cryptographic primitives and usage. Recommendation: upgrade to 4.2.0+ and, for real secrets, prefer the platform crypto APIs or modern audited libraries.

5. **libssl3 (Debian)** — CVE-2025-15467 and several other High/Critical issues. Fix: update OpenSSL packages to 3.0.18-1~deb12u2 (or newer). In practice this is solved by updating the base image and running `apt upgrade` during the build.

**Secret scanning (Trivy).**  
Trivy was run as `trivy image --scanners secret`; the output is in `trivy-secrets.txt`. For all targets (debian, node-pkg) the Secrets column is `-`, so no suspicious secrets were detected in the filesystem of the image. That is expected for this training image, but in a real project I would keep this step as a gate before deployment.

**Licenses and compliance.**  
The Trivy license scan shows a mostly standard mix of MIT, ISC, Apache-2.0 and similar licenses. For production I would keep a simple policy: explicitly allow MIT/ISC/Apache/BSD, and route any GPL/AGPL or unknown licenses to a legal/compliance review to avoid pulling strong copyleft into closed-source code.

---

## Task 3 — Comparing Syft+Grype and Trivy

**Accuracy and coverage.**  
On the same image both approaches find the same critical packages. The number of records and severity levels line up conceptually; the main difference is format. For Juice Shop that is enough to treat the coverage as equivalent for practical purposes.

**Strengths and weaknesses.**  
Syft+Grype: the main advantage is the clear separation of concerns — first generate an SBOM with Syft, then run vulnerability analysis on that SBOM with Grype. That works well when the SBOM is also needed for audits or other tools. The downside is more moving parts: two tools, two formats, and a slightly more complex pipeline.  
Trivy: a single tool that goes from image to packages, vulnerabilities, and optionally secrets and licenses. In CI it is easier to plug in one command. The flip side is that its SBOM output is tuned for the scanner rather than being a “pure” SPDX/CycloneDX document like Syft can produce.

**When to use which.**  
For quick image checks in CI and a single report that includes vulnerabilities/secrets/licenses, Trivy is the more convenient choice. When you need a stable SBOM in a standard format and a separate vulnerability check step, Syft + Grype is a better fit. In a real project I would often combine them: Syft to publish SBOMs, Trivy for regular image scans in the pipeline.

**Integration in practice.**  
Both tools run fine in Docker. On Windows without WSL, redirecting Grype output to a file can be a bit picky, but once the commands are wired, the table and JSON are easy to consume. In Linux/WSL or CI both approaches are installed with a single command and write results to files, so operational complexity is roughly the same.
