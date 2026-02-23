# Lab 3 Submission — Secure Git

## Task 1 — SSH Commit Signature Verification

### 1.1 Why Sign Commits?

Signing commits proves two things: **who** made the change and that the change wasn’t **tampered with** after it was created.

- **Integrity:** The commit hash is tied to your private key. If someone changes so much as a single character in the commit, the signature becomes invalid and GitHub will show it as unverified.
- **Authenticity:** Only someone with your private key can produce a valid signature. So when people see “Verified” on a commit, they know it really came from you, not from a spoofed email or a compromised account, if not only key was compromised.
- **Accountability:** In team and open-source workflows, signed commits make it clear who approved or authored what, which matters for audits and compliance.

Without signing, anyone can set their Git `user.name` and `user.email` to your identity and push commits that look like yours. Signing stops that: the signature is bound to your key, not just your email.

### 1.2 SSH Key Setup and Git Configuration

**What was done (do the same on your machine if you haven’t yet):**

1. **Generate an SSH key for signing**

2. **Add the public key to GitHub**
3. **Tell Git to use that key for signing and to sign every commit**
  
**Evidence of setup**

- [ ] **Config check:** 
  ```
git config --global --list | grep -E "signingkey|gpgsign|gpg.format"
user.signingkey=C:/Users/Phoenix/.ssh/id_ed25519.pub
commit.gpgsign=true
gpg.format=ssh

  ```
- [ ] **Verified badge:**
  ```
  /labs/lab3/docs/badge.png
  ```

### 1.3 Why Commit Signing Matters in DevSecOps

In DevSecOps, code moves fast and automation (CI/CD, merges, releases) trusts Git history. If history can’t be trusted, every downstream step is at risk.

So in short, commit signing is a small step that makes the whole pipeline’s trust model stronger and makes security and compliance reviews meaningful.

---

## Task 2 — Pre-commit Secret Scanning

### 2.1 Pre-commit Hook Setup

A **pre-commit hook** runs automatically before each `git commit`. For this lab it:

1. Lists all **staged** files.
2. Splits them into:
   - **Non-lectures:** everything outside a `lectures/` folder.
   - **Lectures:** files under `lectures/` (treated as educational; secrets there don’t block the commit).
3. Runs **TruffleHog** (Docker) on non-lectures only — if it finds possible secrets, the commit is **blocked**.
4. Runs **Gitleaks** (Docker) on every staged file — if it finds secrets in a non-lectures file, the commit is **blocked**; findings only in `lectures/` are reported but **allowed**.
5. If any blocking condition is true, the hook exits with code 1 and the commit is aborted; otherwise the commit proceeds.

**Configuration:**

- **Hook file:** `.git/hooks/pre-commit`  
  On Windows this is in your repo’s `.git` folder (e.g. `DevSecOps-CC\.git\hooks\pre-commit`). The version in this repo is set up to run with **Git Bash** (so use Git Bash when you run `git commit` if you use the bash hook), or you can use the PowerShell hook — see **TODO.md**.
- **Requirements:** Docker Desktop (or Docker Engine) must be running so the hook can run TruffleHog and Gitleaks in containers. No need to install them on the host.

**How the hook was created:**

- The hook script was added under `.git/hooks/pre-commit`.

### 2.2 Secret Detection Test

To show that the hook really blocks commits when secrets are present and allows them when they’re not:

**Evidence to add:**

- [ ] **Blocked commit:**:
  ```
  /labs/lab3/docs/
  ```
- [ ] **Successful commit:**
  ```
  /labs/lab3/docs/successful.png
  ```

### 2.3 How Automated Secret Scanning Helps

So in practice, the pre-commit hook is a small automation that makes “don’t commit secrets” enforceable and repeatable for the whole team.

---
