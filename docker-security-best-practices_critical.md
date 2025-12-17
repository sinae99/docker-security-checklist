## 1. Host & Daemon Security (Critical)
- Use a **hardened, updated Linux host** with minimal packages.
- Install Docker **only from official sources**.
- Restrict access to the `docker` group (full root equivalent).
- Enable:
  - `"live-restore": true`
  - `"userns-remap": "default"`
  - Logging level: `"info"`
- **Never** expose the Docker daemon socket without TLS.
- Prefer **rootless Docker** when possible.

---

## 2. Image Security (High Impact)
- Use **minimal, trusted base images** (Alpine, Distroless, scratch).
- **Do not use `:latest`** — always pin versions.
- Scan images with **Trivy/Snyk** before use.
- Ensure no secrets are in Dockerfiles or layers.
- Use **multi‑stage builds** to keep runtime images small and clean.

---

## 3. Runtime Hardening (Highest Impact)
- Run containers as **non‑root user**.
- Drop privileges:
  ```bash
  --cap-drop=ALL --cap-add=<needed>
  ```
- Avoid dangerous flags:
  - `--privileged`
  - `--network host`
  - `-v /var/run/docker.sock:/var/run/docker.sock`
- Use:
  ```bash
  --read-only
  --security-opt no-new-privileges
  ```
- Apply resource limits:
  ```bash
  --memory, --cpus, --pids-limit
  ```

---

## 4. Filesystem & Volume Security
- Use **read‑only root FS**.
- Use **tmpfs** for writable paths.
- Use read‑only volumes when possible.
- Never mount sensitive host paths (/, /etc, /lib, /boot).

---

## 5. Networking Essentials
- Use **custom Docker networks** (avoid default bridge).
- Disable inter‑container communication unless needed.
- Bind ports to specific interfaces, e.g.:
  ```bash
  -p 127.0.0.1:8080:80
  ```
- Avoid exposing ports to `0.0.0.0`.

---

## 6. Docker Compose Best Practices
- Pin image versions.
- Define resource limits.
- Run services as non‑root.
- Use secrets (`docker secrets`) instead of env vars.
- Avoid `network_mode: host`.
- Mark services as `read_only`.

---

## 7. Monitoring & Compliance
- Regularly update Docker and base images.
- Enable Docker audit logging.
- Scan images on every CI/CD build.
- Periodically run:
  ```bash
  docker-bench-security
  ```

---

*