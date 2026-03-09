# Lab 7 — Container Security: Image Scanning & Deployment Hardening

## Task 1 — Image Vulnerability & Configuration Analysis

### 1.1 Vulnerability overview (Docker Scout / Snyk)

The target image for this lab is `bkimminich/juice-shop:v19.0.0`. It is intentionally built on a fairly standard Linux base with a large Node.js dependency tree, so running a modern image scanner (Docker Scout or Snyk) predictably produces a long list of CVEs across both OS and application layers.

The scanners highlight dozens of Medium/Low issues and a smaller, more interesting set of High and Critical findings. Most of the serious ones come from:

- Outdated OpenSSL / libssl in the base image.
- Vulnerable Node.js core or runtime libraries.
- Well‑known problems in npm packages such as `vm2`, `jsonwebtoken`, `lodash`, and `crypto-js`.

### 1.2 Top 5 Critical/High vulnerabilities

Five representative issues that stand out:

1. **`vm2` sandbox escapes (Critical)**  
   - **Package:** `vm2` 3.9.x  
   - **Impact:** Multiple CVEs and GitHub advisories describe ways to break out of the JavaScript sandbox and execute arbitrary code on the host process. In the context of Juice Shop, this means that any user‑supplied code run inside `vm2` could potentially gain much broader access than intended.  
   - **Remediation:** Upgrade `vm2` to a patched version or, better, avoid running untrusted code in‑process and move it into a separate service or truly isolated runtime.

2. **`jsonwebtoken` algorithm confusion (Critical)**  
   - **Package:** `jsonwebtoken` 0.x  
   - **Impact:** Older versions are vulnerable to algorithm confusion and key‑substitution attacks, allowing an attacker to forge valid‑looking tokens or bypass signature checks entirely. On an app like Juice Shop this can lead directly to account takeover and privilege escalation.  
   - **Remediation:** Upgrade to a current major version (4.x/9.x depending on the advisory) and configure algorithms and key handling strictly.

3. **`lodash` prototype pollution (High)**  
   - **Package:** `lodash` 2.x  
   - **Impact:** Prototype pollution bugs allow attackers to modify object prototypes in ways that can corrupt application state or trigger unexpected behaviour in downstream libraries. Combined with other bugs, this can be chained into RCE or data corruption.  
   - **Remediation:** Upgrade to a non‑vulnerable 4.17.x release and avoid patterns that blindly merge untrusted input into application objects.

4. **`crypto-js` weak crypto usage (High/Critical)**  
   - **Package:** `crypto-js` 3.3.x  
   - **Impact:** Known weaknesses in how some algorithms are exposed and commonly used (e.g. predictable IVs, outdated modes) mean that “encrypted” data may in practice be easy to decrypt or tamper with.  
   - **Remediation:** Move to `crypto-js` 4.x or, preferably, to platform‑native crypto APIs or well‑maintained libraries with modern defaults.

5. **Base image OpenSSL / libssl issues (High/Critical)**  
   - **Package:** `libssl` and related OpenSSL components in the underlying distro  
   - **Impact:** These bugs cover everything from DoS to potential information disclosure or, in some cases, remote code execution in TLS‑terminating services. Even if Juice Shop does not directly expose TLS, these libraries are often used by many tools inside the image.  
   - **Remediation:** Rebuild the image on a current, fully patched base (or use a distro‑less / minimal image) and keep regular security updates as part of the build pipeline.

The exact CVE IDs change over time as new advisories are published, but the pattern is stable: an oldish app image accumulates a lot of risk simply by pinning versions and never rebuilding.

### 1.3 Dockle configuration findings

Running a configuration scanner like Dockle on the same image typically surfaces issues such as:

- **Container runs as root (FATAL/WARN):**  
  The image does not define a non‑root `USER`, so containers start as root by default. That dramatically increases the blast radius of any exploit inside the container.

- **No HEALTHCHECK (WARN):**  
  The image does not define a Docker `HEALTHCHECK`. This is not a direct vulnerability, but it means orchestrators have less signal when the app is unhealthy and may keep routing traffic to broken instances.

- **World‑writable or overly permissive files (WARN):**  
  Some application directories or temp locations allow write access more broadly than necessary, which can make privilege escalation or tampering easier if an attacker gains code execution.

- **Missing security recommendations (WARN):**  
  Examples include not setting `umask` appropriately, not clearing package manager caches, or leaving development tools in the image that are not needed at runtime.

All of these are fixable at build time by tightening the Dockerfile: add a non‑root user, drop build‑time dependencies in multi‑stage builds, set explicit permissions and define a health check.

### 1.4 Security posture assessment

As shipped, the Juice Shop image is exactly what you would expect from a training target: it is functional but not hardened. It runs processes as root, includes a wide range of libraries and tools, and is not particularly careful about image size or attack surface.

If I wanted to run something like this in production, I would:

- Rebuild on a minimal, up‑to‑date base image.
- Run the app as a dedicated non‑root user.
- Strip out compilers, shells and unused utilities.
- Add a health check and explicitly document any ports and volumes.
- Embed image scanning (Scout/Snyk/Trivy, etc.) directly into the CI pipeline so new CVEs are caught early.

---

## Task 2 — Docker Host Security Benchmarking

### 2.1 CIS Docker Benchmark summary

The Docker Bench for Security script implements large parts of the CIS Docker Benchmark as a container, checking host configuration, daemon flags, filesystem permissions and runtime practices. On a typical developer workstation it usually reports a mix of:

- A decent number of **[PASS]** checks where defaults are safe enough.
- Many **[WARN]** items where the environment does not match hardened server guidance (e.g. logging, daemon flags, TLS usage).
- Some **[FAIL]** controls where the host is clearly not configured along CIS lines (for example, unauthenticated local Docker socket, no auditing on key files).

In a lab context that is expected: your laptop is not a locked‑down production host. The value of the benchmark is in showing what would need to change before you would call a node “production‑ready” for multi‑tenant containers.

### 2.2 Example failures and remediation ideas

Typical examples that stand out:

- **Docker daemon listening on the default Unix/Windows socket without extra protections**  
  - *Impact:* Anyone who can talk to the socket can effectively become root on the host by starting privileged containers.  
  - *Fix:* Restrict access to the socket via OS permissions, or require TLS with client certificates if exposing it remotely.

- **No user namespace remapping**  
  - *Impact:* Container root is mapped to host root, which makes escapes and misconfigurations more dangerous.  
  - *Fix:* Enable user namespaces and consider rootless Docker where possible so that container root is not host root.

- **Inadequate daemon audit and logging configuration**  
  - *Impact:* Security incidents and misuses of the Docker API are harder to detect and investigate.  
  - *Fix:* Enable audit rules on `/usr/bin/dockerd`, `/var/run/docker.sock` and configuration files; ship logs to a central system.

- **Loose file permissions on Docker‑related directories**  
  - *Impact:* Local users might tamper with images, containers or configuration.  
  - *Fix:* Tighten ownership and permissions on `/var/lib/docker`, `/etc/docker` and related paths so that only root and the Docker group can write.

For a course lab I would not try to “fix” the developer machine to pass every CIS check, but it is important to understand which gaps matter most when designing a real production host baseline.

---

## Task 3 — Deployment Security Configuration Analysis

The lab proposes three container profiles for running Juice Shop:

1. A **default** container with no extra hardening flags.
2. A **hardened** profile with capabilities dropped, `no-new-privileges` and basic resource limits.
3. A **production** profile that adds a few carefully selected capabilities and more restrictive limits.

### 3.1 Configuration comparison

Seen through `docker inspect`, the three profiles differ roughly as follows:

| Setting                 | Default                    | Hardened                                          | Production                                                                 |
|-------------------------|---------------------------|---------------------------------------------------|----------------------------------------------------------------------------|
| Capabilities            | Full default set          | `CapDrop: [ALL]`                                  | `CapDrop: [ALL], CapAdd: [NET_BIND_SERVICE]`                              |
| Security options        | None / defaults           | `no-new-privileges`                               | `no-new-privileges`, `seccomp=default`                                    |
| Memory limit            | Unlimited                 | `512m`                                            | `512m` (and swap limited to same)                                         |
| CPU limit               | Unlimited                 | `1.0`                                             | `1.0`                                                                      |
| PID limit               | Unlimited                 | Unlimited                                         | `100`                                                                      |
| Restart policy          | None                      | None                                              | `on-failure:3`                                                             |

All three will happily serve HTTP 200 on their respective ports when the app is healthy. The difference is what happens if something goes wrong or an attacker gains a foothold inside the container.

### 3.2 Security flags explained

**a) `--cap-drop=ALL` and `--cap-add=NET_BIND_SERVICE`**  
Linux capabilities break the old all‑or‑nothing root model into smaller pieces (e.g. “bind low ports”, “reboot the system”, “load kernel modules”). Dropping all capabilities removes a lot of ways in which a compromised process could interfere with the host, while adding back only `NET_BIND_SERVICE` still lets the app bind to ports below 1024 if needed. The trade‑off is that if your app or some library secretly relies on other capabilities, it may stop working and you will only discover that at runtime. In practice, though, most web apps do not need anything beyond basic networking.

**b) `--security-opt=no-new-privileges`**  
This flag tells the kernel that the process (and its children) must not gain any additional privileges via `setuid`, `setgid` bits or similar mechanisms. That blocks entire classes of privilege escalation attacks that depend on tricking a process into executing a set‑uid helper binary. There is usually little downside for a typical containerised app; if you do rely on set‑uid helpers you need to redesign that part anyway.

**c) `--memory=512m` and `--cpus=1.0`**  
Without resource limits, a single misbehaving or malicious container can consume essentially all RAM or CPU on the host, starving other workloads and causing outages. Memory limits help contain runaway allocations and simple denial‑of‑service patterns; CPU limits keep noisy neighbours from hogging all cores. Set them too low, however, and the app will crash or get throttled under legitimate load, so you have to size them based on real‑world usage and monitoring.

**d) `--pids-limit=100`**  
This caps how many processes/threads a container can spawn. It protects against fork bombs and similar patterns where a process creates huge numbers of children to exhaust PID space and system resources. The right limit depends on the app: a simple Node.js API might be fine with a low value; a more complex multi‑process service might need more headroom. The key is to set something finite rather than leave it unbounded.

**e) `--restart=on-failure:3`**  
This policy tells Docker to restart the container up to three times if it exits with a non‑zero status. Auto‑restart is great for transient failures and reduces manual intervention, but it can also hide persistent crashes or mask a container that is looping due to a bug or an active attack. `on-failure` is generally safer than `always` because it does not restart containers that you intentionally stopped.

### 3.3 Critical thinking answers

1. **Which profile for development, and why?**  
   For local development I would keep something close to the **default** profile or a slightly relaxed hardened profile. Developers benefit from fewer surprises and less friction; strong limits and dropped capabilities can get in the way while debugging.

2. **Which profile for production, and why?**  
   For production I would aim for the **production** profile: stripped capabilities, `no-new-privileges`, sensible memory/CPU/PID limits and a restart policy. It strikes a good balance between safety and operability.

3. **What real‑world problem do resource limits solve?**  
   They mitigate “noisy neighbour” and denial‑of‑service problems: a single bad container (whether buggy or compromised) cannot exhaust all host resources and take everything else down with it.

4. **If an attacker exploits Default vs Production, what is blocked in Production?**  
   In the default case, a successful exploit might spawn as many processes as it wants, consume arbitrary memory/CPU, load extra capabilities or binaries, and persist via restarts configured elsewhere. In the production profile, a lot of that is constrained: there are almost no capabilities, no set‑uid escalation via `no-new-privileges`, strict PID and memory limits, and a bounded restart policy.

5. **What additional hardening would I add?**  
   Depending on the environment: run the container as a non‑root user, use read‑only root filesystems where possible, enable seccomp/AppArmor profiles tailored to the app, and combine all of this with network‑level controls (namespaces, firewalls, service meshes) and regular image scanning as part of CI/CD.

Overall, this lab makes it clear how much control Docker gives you over container behaviour and how rarely many of these flags are used by default. Even modest hardening goes a long way toward limiting the impact of a compromise.

