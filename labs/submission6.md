# Lab 6 — IaC Security Scanning and Tool Comparison

## Task 1 — Terraform & Pulumi Security Scanning

For this lab I worked against the intentionally vulnerable IaC under `labs/lab6/vulnerable-iac/`, which includes Terraform modules for AWS, Pulumi configs, and Ansible playbooks. The goal was to treat them as real infrastructure code and run several scanners over them, then compare what each tool is good at.

### 1.1 Terraform: tfsec vs Checkov vs Terrascan

The Terraform code (`terraform/main.tf`, `security_groups.tf`, `database.tf`, `iam.tf`, `variables.tf`) is deliberately packed with bad practices: public S3 buckets, security groups open to `0.0.0.0/0`, unencrypted RDS instances, wildcard IAM permissions and hard‑coded values. I ran three scanners over this directory:

- **tfsec** (Aquasec) — Terraform‑focused, lightweight and fast.
- **Checkov** (Bridgecrew) — policy‑as‑code style scanner with a large rule set.
- **Terrascan** (Tenable) — OPA‑based scanner with strong compliance mapping.

In all three cases the tools did what I would expect on this type of code: they flagged public S3 buckets, unencrypted databases, over‑permissive security groups, overly broad IAM policies and hard‑coded credentials. The exact counts differ slightly between them (each vendor has its own rule catalog), but they all agree that this Terraform should never be deployed as‑is.

From a usability point of view:

- **tfsec** feels the simplest: point it at the folder and it quickly returns a focused list of issues. Its messages are fairly clear and the noise level is low, which makes it a good fit as an always‑on CI check for Terraform.
- **Checkov** is more verbose but also richer. It brings in a ton of built‑in policies (including some that feel more like governance than security), which is useful in larger organisations. The JSON output is structured around resources, policy IDs and severities, which makes it easy to feed into dashboards or policy engines.
- **Terrascan** maps findings to compliance frameworks (PCI‑DSS, HIPAA, etc.) and leans into OPA‑style policy logic. For this small lab codebase that is arguably overkill, but it shows why Terrascan is attractive in environments where somebody has to justify compliance coverage.

Overall impression: if I only needed one quick safety net for Terraform in CI, tfsec would probably be my first pick. For broader policy coverage across multiple IaC types, Checkov and Terrascan start to shine.

### 1.2 Pulumi with KICS

The Pulumi part lives in `pulumi/__main__.py`, `Pulumi.yaml` and `Pulumi-vulnerable.yaml`. It defines infrastructure similar in spirit to the Terraform: public buckets, open security groups, unencrypted databases and so on. For this I used **KICS (Checkmarx)**, which understands Pulumi YAML manifests and ships with Pulumi‑aware queries.

KICS correctly identifies a long list of issues in the Pulumi configuration:

- Public S3‑style storage without encryption or proper access controls.
- Security groups that allow inbound traffic from the entire internet.
- Databases without at‑rest encryption and with public exposure.
- Weak or missing tags that would normally be required for governance.
- Insecure defaults and potential secrets baked into config.

What I like about KICS here is that it “speaks” the Pulumi domain in its messages: it refers to Pulumi resources and properties rather than just generic JSON blobs, which makes it easier to connect findings back to the code. The Pulumi‑specific query catalog feels reasonably mature for a training environment: it covers the big categories (networking, encryption, IAM, secrets) well enough to be useful.

### 1.3 Terraform vs Pulumi security issues

Looking at Terraform and Pulumi side by side, the actual security problems are very similar: too many `0.0.0.0/0`, not enough encryption, broad IAM permissions and secrets in plain text. The main difference is **how** those issues are expressed:

- Terraform’s HCL is purely declarative; each resource is a static block. Rule engines like tfsec and Checkov can walk these in a very predictable way.
- Pulumi mixes Python and YAML: infrastructure is code, with loops, conditionals and helper functions. KICS focuses on the Pulumi YAML manifests where the final resolved resources live, which sidesteps a lot of the complexity of parsing arbitrary Python.

From a security scanning perspective that means:

- Terraform tends to be easier to scan exhaustively; the tools have fewer surprises and can flag issues very precisely at the resource level.
- Pulumi gives developers more flexibility, but also more room to hide problems in logic. KICS’s Pulumi support is a good start, but in a real project I would complement it with code‑aware checks (linters, Semgrep‑style rules) on the Python itself.

### 1.4 Critical infrastructure findings

Across Terraform and Pulumi the most concerning issues are:

1. **Public, unencrypted storage (S3 buckets)** — buckets are world‑readable, versioning/encryption are disabled. Fix: enable `server_side_encryption`, make them private and front them with controlled access (CDN, signed URLs, etc.).
2. **Security groups open to the world** — rules allowing `0.0.0.0/0` on sensitive ports (SSH, DB, admin APIs). Fix: restrict CIDR ranges, or use VPN/bastion hosts instead.
3. **Unencrypted, publicly exposed databases** — RDS instances or equivalent without `storage_encrypted = true`, optionally accessible from the internet. Fix: turn on encryption, put them in private subnets, and tighten security group rules.
4. **Wildcard IAM policies** — `Action = "*"`, `Resource = "*"`, or similar patterns. Fix: replace with least‑privilege policies that only allow the specific actions and resources needed.
5. **Hard‑coded secrets in variable defaults** — API keys and passwords in `variables.tf`, Pulumi YAML or code. Fix: move them into a secrets manager (AWS Secrets Manager, Parameter Store, Vault, etc.) and reference them dynamically.

These are exactly the sorts of misconfigurations real‑world incidents are made of, which is why IaC scanning is so valuable.

---

## Task 2 — Ansible Security Scanning with KICS

The Ansible code under `ansible/` (`deploy.yml`, `configure.yml`, `inventory.ini`) is also deliberately unsafe. It contains hard‑coded credentials, weak SSH settings and tasks that happily log sensitive data.

### 2.1 KICS findings on Ansible

Running KICS over this directory highlights several clear anti‑patterns:

- **Secrets in plain text** — passwords and other sensitive values appear directly in playbooks and inventory. That is fine for a training lab but unacceptable in production.
- **Tasks leaking sensitive data** — operations that manipulate secrets do not use `no_log: true`, which means credentials and tokens would end up in logs.
- **Risky use of `shell` and `command`** — some tasks use raw shell commands where safer, more specific Ansible modules exist. That increases the risk of command injection or subtle behaviour differences across hosts.
- **Insecure file permissions** — files created with overly permissive modes (e.g. `0777`), which violates the principle of least privilege.

KICS’s Ansible rule catalog is quite practical: it focuses on secrets management, command execution, permissions and authentication. It does a good job of catching the kinds of mistakes that are easy to make when you are in a hurry writing playbooks.

### 2.2 Best‑practice violations and impact

Three illustrative examples:

1. **Hard‑coded passwords in `deploy.yml`**
   - **Violation:** credentials are written directly in the YAML.
   - **Impact:** anyone with repo access or log access can see them; rotation becomes painful because secrets are checked into version control.
   - **Fix:** move secrets into Ansible Vault or an external secrets manager; load them via vars files encrypted with `ansible-vault`.

2. **Missing `no_log` on sensitive tasks**
   - **Violation:** tasks that handle passwords or keys do not set `no_log: true`.
   - **Impact:** sensitive values will be printed to stdout and end up in CI logs or centralised logging systems.
   - **Fix:** add `no_log: true` for those tasks so that Ansible masks their output.

3. **Using `shell` instead of a dedicated module**
   - **Violation:** shell pipelines where a specific module (like `user`, `file`, `service`, etc.) should be used.
   - **Impact:** harder to audit, more fragile across distributions, and potentially vulnerable to injection if variables are interpolated incorrectly.
   - **Fix:** replace shell calls with the nearest idempotent module and pass parameters as structured data instead of strings.

These might feel minor compared to something like a public database, but in aggregate they dramatically weaken the security posture of configuration management.

---

## Task 3 — Comparative Tool Analysis & Security Insights

### 3.1 Tool comparison matrix

Based on how the tools behave on this intentionally vulnerable codebase, the picture looks roughly like this:

| Criterion              | tfsec                          | Checkov                               | Terrascan                              | KICS (Pulumi + Ansible)                    |
|------------------------|--------------------------------|---------------------------------------|----------------------------------------|--------------------------------------------|
| **Total findings**     | Tens of issues on Terraform    | Similar order of magnitude on Terraform | Comparable number of violations       | Dozens across Pulumi + Ansible            |
| **Scan speed**         | Fast                           | Medium (more policies)                | Medium                                  | Medium                                     |
| **False positives**    | Low to medium                  | Medium (wider policy surface)         | Medium                                  | Medium                                     |
| **Report quality**     | Clear, concise CLI/JSON        | Rich JSON plus compact CLI            | Human/JSON with compliance context     | JSON/HTML reports + minimal console view  |
| **Ease of use**        | Very easy for Terraform only   | Easy once docker/image is set up      | Slightly more complex CLI              | Straightforward across multiple IaC types  |
| **Platform support**   | Terraform only                 | Many (Terraform, CFN, K8s, etc.)      | Many IaC/cloud targets                  | Terraform/Pulumi/Ansible and more         |
| **Output formats**     | Text, JSON, SARIF              | JSON, CLI, SARIF, others              | JSON, text                             | JSON, HTML, SARIF, console                |
| **CI/CD integration**  | Simple one‑liner               | Well‑documented, widely used          | Good when you care about compliance    | Good as a unified scanner                 |
| **Unique strengths**   | Terraform‑focused, low friction| Huge policy catalog, multi‑framework  | Compliance frameworks via OPA          | First‑class Pulumi and Ansible support    |

In short: tfsec is a great “Terraform seatbelt”, Checkov and Terrascan broaden coverage and policy depth, and KICS adds solid scanning for Pulumi and Ansible so you don’t need a separate tool for each.

### 3.2 Vulnerability categories and best tools

Thinking in terms of security domains rather than individual tools:

- **Encryption issues (S3, RDS, etc.):** all three Terraform scanners (tfsec, Checkov, Terrascan) spot missing at‑rest encryption flags. KICS does the same for Pulumi resources.
- **Network security (security groups, open ports):** again, tfsec/Checkov/Terrascan are strong here and KICS mirrors that for Pulumi. These are some of the clearest, least ambiguous findings.
- **Secrets management:** hard‑coded secrets show up in Terraform variables, Pulumi configs and Ansible playbooks. Checkov and KICS are particularly vocal about these, and KICS’s Ansible rules are very explicit about best practices.
- **IAM/permissions:** Terraform IAM policies with wildcards are well covered by tfsec and Checkov; Pulumi IAM resources hit KICS rules in a similar way.
- **Access control & compliance:** Terrascan adds an extra layer by mapping violations to compliance controls; Checkov and KICS both have rules that read like checklist items from standards and best‑practice guides.

No single tool “wins” everywhere, but patterns emerge: tfsec is the quickest Terra‑only guardrail, Checkov and Terrascan broaden Terraform’s policy coverage, KICS fills the gap for Pulumi and Ansible, and together they give a healthy overlap on the dangerous stuff.

### 3.3 Top 5 critical findings and remediation

Across all IaC types, the five most important issues to fix first would be:

1. **Public, unencrypted storage buckets** — always require encryption at rest and tight access policies; consider separate locked‑down buckets for logs and backups.
2. **Databases exposed to the internet without encryption** — move them to private subnets, enable at‑rest and in‑transit encryption, and restrict access to specific application subnets or VPNs.
3. **Wildcard IAM roles and policies** — replace `*` with the minimal set of actions and resources necessary; introduce role separation between application components.
4. **Hard‑coded secrets in Terraform, Pulumi and Ansible** — centralise secrets in a dedicated secrets management solution and remove them from code completely.
5. **Overly permissive Ansible tasks and file modes** — lock down file permissions, avoid raw shell when modules exist, and keep logs free of sensitive data.

Fixing these five areas alone would dramatically harden the infrastructure defined by this lab.

### 3.4 Tool selection and CI/CD strategy

If I were wiring this into a real DevSecOps pipeline, my plan would be:

- Use **tfsec** or **Checkov** on every Terraform change (pre‑commit and CI) to keep obvious misconfigurations from ever landing.
- Use **KICS** in CI for Pulumi and Ansible so that developers get consistent feedback across different IaC frameworks.
- Add **Terrascan** or a similar policy‑driven tool where formal compliance reporting matters.
- Periodically review and tune rules to strike a balance between coverage and noise, and track remediation of critical findings with clear SLAs.

The main lesson from this lab is that IaC scanning is not optional anymore. With a small, well‑chosen mix of tools it is entirely feasible to catch most serious misconfigurations long before anything is deployed. 

