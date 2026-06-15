# Incident Investigation: Operation Kobold

**Target System:** Linux (Node.js / Dockerized Environment)
**Severity:** Critical
**Date of Analysis:** 2026-04-12
**Investigator:** ansil-prof

---

## 1. Executive Summary
During a targeted security assessment, the internal asset "Kobold" suffered a complete system compromise. The adversary achieved a multi-stage breach by establishing an initial web-based foothold through a vulnerable Node.js application, pivoting laterally via a misconfigured legacy system group, and ultimately abusing an exposed Docker socket to bypass AppArmor restrictions and achieve persistent root access.

## 2. Attack Lifecycle & Timeline
*   **Phase 1: Initial Access (Web Exploitation)**
    *   The adversary targeted a web service named "MCP Jam Inspector" hosting a custom Node.js application.
    *   By manipulating a backend API vulnerable to command injection via a POST request, the attacker executed a reverse shell payload and gained initial access as the low-privileged user `ben`.
*   **Phase 2: Lateral Movement (Group Privilege Abuse)**
    *   Internal enumeration revealed the compromised user `ben` was a member of the legacy `operator` group.
    *   The attacker exploited a system misconfiguration that permitted members of the `operator` group to execute `newgrp docker` without requiring a password.
    *   This pivot granted the attacker direct, unrestricted access to the host's Docker socket (`/var/run/docker.sock`).
*   **Phase 3: Privilege Escalation & Persistence (Docker Escape)**
    *   The attacker leveraged the exposed Docker socket to deploy a container with root privileges (`--user 0`) and mounted the host's root filesystem (`/`) into the container.
    *   To bypass existing AppArmor profiles that restricted direct file reading (preventing `cat /root/root.txt`), the adversary altered their execution strategy.
    *   The attacker injected a malicious public SSH key directly into the host's `/root/.ssh/authorized_keys` file via the container shell, establishing persistent, authenticated administrative (root) access over SSH.

## 3. Indicators of Compromise (IOCs)
*   **Process Artifacts:** Unauthorized shell execution spawned from the Node.js web application directory (`/usr/local/lib/node_modules/@mcpjam/inspector`).
*   **System Logs:** Abnormal execution of the `newgrp docker` command by standard or service users.
*   **Container Activity:** Spawning of temporary containers with the `--user 0` flag and root directory volume mounts (`-v /:/hostfs`).
*   **File System Artifacts:** Unauthorized modifications to `/root/.ssh/authorized_keys`.

## 4. Remediation Strategy
1.  **Application Code Review:** Audit and sanitize the MCP Jam Inspector Node.js backend API to prevent command injection vulnerabilities within POST requests.
2.  **Access Control & Group Auditing:** Remove non-administrative users from legacy groups such as `operator` and revoke passwordless `newgrp` execution privileges.
3.  **Docker Socket Security:** Restrict access to `/var/run/docker.sock`. Ensure developers and service accounts cannot interact with the Docker daemon, as it provides a direct path to root privilege escalation.
