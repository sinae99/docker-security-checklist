


## 1. Host & Installation Security

### 1.1 Host Hardening
- Use a hardened Linux distribution (Ubuntu LTS, RHEL, etc.)
- Keep kernel and OS fully updated
- Install only minimal packages; reduce attack surface
- Enable automatic security updates
- Disable unnecessary services and legacy protocols

### 1.2 Docker Installation
- Install Docker only from official repositories
- Verify Docker GPG keys before installation
- Use latest stable Docker Engine
- Enable kernel security frameworks: **AppArmor / SELinux**
- Create a separate partition for `/var/lib/docker`

### 1.3 Host Access Control
- Restrict access to the `docker` group (root-equivalent)
- Enforce strong SSH configuration on hosts
- Require MFA for host access
- Enable full audit logging for Docker binaries and config files

---

## 2. Docker Daemon Security

### 2.1 Core Daemon Settings
Recommended `/etc/docker/daemon.json`:
```json
{
  "icc": false,
  "live-restore": true,
  "userns-remap": "default",
  "no-new-privileges": true,
  "log-level": "info",
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```
Key controls:
- Enable **user namespace remapping**
- Disable inter-container communication (ICC)
- Set default seccomp + AppArmor profiles
- Enforce log rotation

### 2.2 Docker Socket Security
- Never expose Docker socket publicly
- If TCP daemon is required:
  ```json
  {
    "tls": true,
    "tlsverify": true,
    "tlscacert": "/etc/docker/ca.pem",
    "tlscert": "/etc/docker/cert.pem",
    "tlskey": "/etc/docker/key.pem"
  }
  ```
- Bind only to `127.0.0.1` when possible

### 2.3 Storage + Registry
- Use storage driver: `overlay2`
- Use private registries with authentication
- Enable:
  ```bash
  export DOCKER_CONTENT_TRUST=1
  ```
- Avoid insecure registries

### 2.4 Resource Defaults
Set defaults in daemon:
```json
"default-ulimits": {
  "nofile": { "Soft": 64000, "Hard": 64000 },
  "nproc": { "Soft": 32768, "Hard": 32768 }
}
```

---

## 3. Files, Permissions & Certificates

### 3.1 File Ownership
- `/etc/docker` → owned by `root:root`
- `daemon.json` → owned by `root:root`
- `/var/run/docker.sock` → owned by `root:docker`
- Systemd unit files owned by `root:root`

### 3.2 Permission Requirements
- `daemon.json` → `644`
- `docker.sock` → `660`
- TLS certs → `444`
- TLS keys → `400`
- `/etc/docker` directory → `755`

---

## 4. Image & Build Security

### 4.1 Base Image Controls
- Use minimal bases: Alpine, Distroless, scratch
- Pin image tags (never use `latest`)
- Pull only from trusted registries
- Verify signatures + checksums
- Maintain a whitelist of approved images

### 4.2 Dockerfile Best Practices
- Create a non-root user:
  ```dockerfile
  RUN addgroup -S app && adduser -S app -G app
  USER app
  ```
- Use `COPY` instead of `ADD`
- Pin package versions
- Remove build tools (clean stages)
- Avoid secrets in Dockerfile
- Use multi-stage builds
- Include a `HEALTHCHECK`

### 4.3 Image Scanning
- Scan images in CI/CD (Trivy, Snyk, Clair)
- Remove unused images regularly
- Maintain SBOMs
- Implement image signing/provenance (Notary/TUF)

---

## 5. Container Runtime Hardening

### 5.1 Container User & Capabilities
- Run all containers as non-root:
  ```bash
  docker run --user 1000:1000 ...
  ```
- Drop all capabilities by default:
  ```bash
  --cap-drop=ALL --cap-add=<only-required>
  ```
- Never use `--privileged`
- Disable privilege escalation:
  ```bash
  --security-opt no-new-privileges
  ```

### 5.2 Filesystem Controls
- Use read-only filesystem:
  ```bash
  --read-only
  ```
- Add tmpfs for writable dirs:
  ```bash
  --tmpfs /tmp
  ```
- Use read-only volume mounts:
  ```bash
  -v host:path:ro
  ```
- Never mount sensitive host folders (`/etc`, `/lib`, `/root`)
- Never mount Docker socket inside containers

### 5.3 Network Runtime Security
- Avoid `--network host`
- Use custom networks (never default bridge)
- Disable ICC on networks
- Restrict port exposure
- Bind ports to localhost:
  ```bash
  -p 127.0.0.1:8080:80
  ```

### 5.4 Resource Limits
- Set memory limits:
  ```bash
  --memory="512m" --memory-swap="512m"
  ```
- Set CPU constraints:
  ```bash
  --cpus="1.5"
  ```
- Limit PIDs:
  ```bash
  --pids-limit=100
  ```
- Configure restart policies:
  ```bash
  --restart=on-failure:5
  ```

### 5.5 Security Profiles
- Use Docker default seccomp
- Use AppArmor:
  ```bash
  --security-opt apparmor=docker-default
  ```
- Use SELinux labels where available
- Never disable seccomp/apparmor protections

### 5.6 Logging & Monitoring
- Configure log rotation
- Enable container health checks
- Monitor resource usage
- Centralize logs (ELK, Loki, Splunk)

---

## 6. Networking Best Practices

### 6.1 Network Configuration
- Use custom bridge networks per service tier
- For swarm/cluster:
  ```bash
  docker network create --opt encrypted --driver overlay net
  ```
- Implement segmentation (frontend/backend)

### 6.2 Network Security
- Restrict service ports
- Avoid public binding unless required
- Use firewalls (ufw/iptables) to limit Docker networks
- Enforce ingress/egress filtering

---

## 7. Docker Compose Security

### 7.1 Compose File Controls
- Use version 3.x+
- Pin all images
- Define CPU/memory limits
- Use `read_only: true`
- Use `tmpfs` as needed
- Add security options:
  ```yaml
  security_opt:
    - no-new-privileges:true
  cap_drop:
    - ALL
  ```

### 7.2 Secrets & Environment
- Use Docker secrets, not env vars
- Add `.env` to `.gitignore`
- Use `env_file` with chmod 600

### 7.3 Networks in Compose
```yaml
networks:
  frontend:
  backend:

services:
  api:
    networks: [frontend]
  db:
    networks: [backend]
```
- Disable default network when unneeded

### 7.4 Volumes
- Prefer named volumes over bind mounts
- Use read-only mounts whenever possible
- Never mount sensitive host paths
- Never mount Docker socket

---

## 8. Operational Security

### 8.1 Updates & Hygiene
- Update Docker Engine frequently
- Rebuild images on base updates
- Remove unused containers, images, volumes:
  ```bash
  docker system prune
  ```

### 8.2 Monitoring & Analysis
- Enable daemon audit logging
- Track API calls
- Use runtime security tools (Falco, AppArmor logs)

### 8.3 Compliance
- Run CIS Docker Benchmark regularly:
  ```bash
  docker-bench-security
  ```
- Maintain security baselines
- Document all Docker configurations

---

## 9. Advanced Security Options

### 9.1 Rootless Docker
- Greatly reduces host attack surface
- Configure subuid/subgid mappings
- Understand limitations (no overlay2 on some distros)

### 9.2 Alternative Runtimes
- Podman (daemonless, rootless default)
- gVisor (sandboxed kernel)
- Kata Containers (VM-level isolation)

### 9.3 Supply Chain Security
- Use SBOMs
- Use Notary/TUF to sign & verify images
- Validate dependencies and build artifacts

---

