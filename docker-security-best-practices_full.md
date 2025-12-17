# Docker Security Best Practices Checklist

> Based on **CIS Docker Benchmark v1.6.0** and **OWASP Docker Security Guidelines**

---

## 1. Docker Installation & Host Configuration

### 1.1 Pre-Installation
- [ ] Use a hardened Linux distribution like ---> LTS
- [ ] Keep the host OS kernel updated to latest stable version
- [ ] Install only the minimal required packages on the host
- [ ] Enable automatic security updates for the host system

### 1.2 Docker Installation
- [ ] Install Docker from official Docker repositories
- [ ] Verify Docker GPG keys before installation
- [ ] Install the latest stable version of Docker Engine

### 1.3 Host Hardening
- [ ] Create a separate partition for Docker (`/var/lib/docker`)
- [ ] Restrict access to the `docker` group
- [ ] Enable and configure audit logging
- [ ] Enable SELinux, AppArmor and ...
- [ ] Disable legacy insecure protocols and services on host

---

## 2. Docker Daemon Configuration

### 2.1 Basic Daemon Settings
- [ ] Set logging level to `info` (not `debug`)
- [ ] Enable live restore: `"live-restore": true`
- [ ] Enable user namespace remapping: `"userns-remap": "default"`

### 2.2 Network Security
- [ ] Never expose Docker daemon socket over TCP without TLS
- [ ] If TCP socket required, enable TLS authentication:
  ```json
  {
    "tls": true,
    "tlsverify": true,
    "tlscacert": "/path/to/ca.pem",
    "tlscert": "/path/to/server-cert.pem",
    "tlskey": "/path/to/server-key.pem"
  }
  ```
- [ ] Disable inter-container communication by default: `"icc": false`
- [ ] Bind daemon to localhost only if TCP socket needed

### 2.3 Storage and Registry
- [ ] Use approved storage drivers (like overlay2)
- [ ] Enable content trust: `export DOCKER_CONTENT_TRUST=1`
- [ ] Configure insecure registries only when absolutely necessary
- [ ] Use private registries with authentication

### 2.4 Resource Controls
- [ ] Set default ulimits in daemon.json:
  ```json
  {
    "default-ulimits": {
      "nofile": { "Name": "nofile", "Hard": 64000, "Soft": 64000 },
      "nproc": { "Name": "nproc", "Hard": 32768, "Soft": 32768 }
    }
  }
  ```
- [ ] Configure default seccomp profile
- [ ] Set default AppArmor/SELinux profile

### 2.5 Daemon Configuration File
Sample `/etc/docker/daemon.json`:
```json
{
  "icc": false,
  "userns-remap": "default",
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true,
  "log-level": "info",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "disable-legacy-registry": true
}
```

---

## 3. Docker Files and Directory Permissions

### 3.1 Critical Files Ownership
- [ ] Verify `/etc/docker` owned by `root:root`
- [ ] Verify `/etc/docker/daemon.json` owned by `root:root`
- [ ] Verify Docker socket `/var/run/docker.sock` owned by `root:docker`
- [ ] Verify Docker systemd unit files owned by `root:root`
- [ ] Verify `/etc/docker/certs.d` owned by `root:root`

### 3.2 File Permissions
- [ ] Set `/etc/docker/daemon.json` to `644` or more restrictive
- [ ] Set `/var/run/docker.sock` to `660` or more restrictive
- [ ] Set Docker TLS CA certificate to `444` or more restrictive
- [ ] Set Docker TLS certificate files to `444` or more restrictive
- [ ] Set Docker TLS key files to `400` or more restrictive
- [ ] Set `/etc/docker` directory to `755` or more restrictive

---

## 4. Container Images and Build Files

### 4.1 Base Images
- [ ] Use minimal base images (Alpine, Distroless, scratch)
- [ ] Pull images from trusted registries only
- [ ] Use specific image tags (never use `:latest` in production)
- [ ] Verify image signatures and checksums
- [ ] Scan all images for vulnerabilities before use
- [ ] Maintain an approved image whitelist

### 4.2 Dockerfile Security
- [ ] Create and use a non-root user in Dockerfile:
  ```dockerfile
  RUN groupadd -r appuser && useradd -r -g appuser appuser
  USER appuser
  ```
- [ ] Use `COPY` instead of `ADD` (unless extracting tar required)
- [ ] Pin versions for all packages and dependencies
- [ ] Remove package managers after installation (if possible)
- [ ] Minimize layers and image size
- [ ] Use multi-stage builds to exclude build dependencies
- [ ] Do not store secrets in Dockerfile or image layers
- [ ] Add `HEALTHCHECK` instruction to images
- [ ] Set appropriate `WORKDIR` (avoid root directory)

### 4.3 Image Management
- [ ] Implement regular image scanning in CI/CD pipeline
- [ ] Use tools like Trivy, Snyk, Clair, or Docker Scout
- [ ] Remove old and unused images regularly
- [ ] Tag images with meaningful version numbers
- [ ] Create and maintain Software Bill of Materials (SBOM)
- [ ] Implement image signing workflow

### 4.4 Secrets Management
- [ ] Never hardcode secrets in Dockerfiles or images
- [ ] Use Docker secrets or external secret management (Vault, etc.)
- [ ] Scan images for leaked credentials
- [ ] Use `.dockerignore` to exclude sensitive files

---

## 5. Container Runtime Security

### 5.1 User and Permissions
- [ ] Run containers as non-root user:
  ```bash
  docker run --user 1000:1000 image
  ```
- [ ] Drop all capabilities and add only required ones:
  ```bash
  docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE image
  ```
- [ ] Never use `--privileged` flag unless absolutely necessary
- [ ] Disable privilege escalation:
  ```bash
  docker run --security-opt=no-new-privileges image
  ```

### 5.2 Filesystem Security
- [ ] Mount root filesystem as read-only:
  ```bash
  docker run --read-only image
  ```
- [ ] Use temporary filesystems for writable directories:
  ```bash
  docker run --read-only --tmpfs /tmp image
  ```
- [ ] Mount volumes as read-only when possible:
  ```bash
  docker run -v /host/path:/container/path:ro image
  ```
- [ ] Do not mount sensitive host directories (`/`, `/boot`, `/dev`, `/etc`, `/lib`)
- [ ] Never mount Docker socket into containers:
  ```bash
  # DANGEROUS - Never do this:
  # docker run -v /var/run/docker.sock:/var/run/docker.sock image
  ```

### 5.3 Network Security
- [ ] Use custom bridge networks (not default bridge)
- [ ] Isolate containers using separate networks
- [ ] Disable inter-container communication when not needed
- [ ] Do not use host network mode (`--network host`) unless required
- [ ] Publish only necessary ports
- [ ] Bind ports to specific interfaces:
  ```bash
  docker run -p 127.0.0.1:8080:80 image
  ```

### 5.4 Resource Limits
- [ ] Set memory limits:
  ```bash
  docker run --memory="512m" --memory-swap="512m" image
  ```
- [ ] Set CPU limits:
  ```bash
  docker run --cpus="1.5" image
  ```
- [ ] Limit restart attempts:
  ```bash
  docker run --restart=on-failure:5 image
  ```
- [ ] Set PIDs limit:
  ```bash
  docker run --pids-limit=100 image
  ```
- [ ] Configure file descriptor limits

### 5.5 Security Profiles
- [ ] Use default seccomp profile (or custom restrictive profile)
- [ ] Enable AppArmor profile:
  ```bash
  docker run --security-opt apparmor=docker-default image
  ```
- [ ] Enable SELinux labels:
  ```bash
  docker run --security-opt label=level:s0:c100,c200 image
  ```
- [ ] Never disable security features:
  ```bash
  # DANGEROUS - Never do this:
  # docker run --security-opt seccomp=unconfined image
  # docker run --security-opt apparmor=unconfined image
  ```

### 5.6 Logging and Monitoring
- [ ] Configure appropriate logging driver
- [ ] Set log rotation limits:
  ```bash
  docker run --log-opt max-size=10m --log-opt max-file=3 image
  ```
- [ ] Enable container health checks
- [ ] Monitor container resource usage
- [ ] Collect and analyze container logs centrally

### 5.7 Other Runtime Configurations
- [ ] Set appropriate hostname (not default container ID)
- [ ] Configure DNS servers explicitly if needed
- [ ] Use `--init` flag to handle signals properly
- [ ] Set resource reservations for critical containers
- [ ] Configure appropriate restart policies

---

## 6. Docker Networking

### 6.1 Network Configuration
- [ ] Create custom networks for different application tiers
- [ ] Use overlay networks for multi-host deployments
- [ ] Enable network encryption for overlay networks:
  ```bash
  docker network create --opt encrypted --driver overlay mynet
  ```
- [ ] Avoid using default bridge network for production
- [ ] Implement network segmentation

### 6.2 Network Security
- [ ] Disable ICC (Inter-Container Communication) on networks when not needed
- [ ] Use firewall rules to restrict Docker network access
- [ ] Implement ingress/egress filtering
- [ ] Avoid exposing unnecessary ports to `0.0.0.0`
- [ ] Use reverse proxies for external access

---

## 7. Docker Compose Security

### 7.1 Compose File Security
- [ ] Use Compose file version 3.x or later
- [ ] Pin all image versions explicitly
- [ ] Define resource limits for all services:
  ```yaml
  services:
    web:
      image: nginx:1.21.6
      deploy:
        resources:
          limits:
            cpus: '0.5'
            memory: 512M
          reservations:
            cpus: '0.25'
            memory: 256M
  ```
- [ ] Configure read-only root filesystems:
  ```yaml
  services:
    web:
      read_only: true
      tmpfs:
        - /tmp
  ```
- [ ] Set security options:
  ```yaml
  services:
    web:
      security_opt:
        - no-new-privileges:true
      cap_drop:
        - ALL
      cap_add:
        - NET_BIND_SERVICE
  ```

### 7.2 User and Permissions in Compose
- [ ] Specify non-root user for all services:
  ```yaml
  services:
    web:
      user: "1000:1000"
  ```
- [ ] Use secrets for sensitive data:
  ```yaml
  secrets:
    db_password:
      file: ./db_password.txt

  services:
    db:
      secrets:
        - db_password
  ```
- [ ] Use configs for non-sensitive configuration files

### 7.3 Network Configuration in Compose
- [ ] Define custom networks and assign services:
  ```yaml
  networks:
    frontend:
    backend:

  services:
    web:
      networks:
        - frontend
    db:
      networks:
        - backend
  ```
- [ ] Disable default network if not needed
- [ ] Do not use `network_mode: host`

### 7.4 Volume Security in Compose
- [ ] Mount volumes as read-only when possible:
  ```yaml
  volumes:
    - ./config:/etc/app:ro
  ```
- [ ] Use named volumes instead of bind mounts
- [ ] Set appropriate volume permissions
- [ ] Never mount Docker socket
- [ ] Never mount sensitive host paths

### 7.5 Environment and Configuration
- [ ] Use `.env` files for environment variables (never commit to git)
- [ ] Add `.env` to `.gitignore`
- [ ] Use Docker secrets instead of environment variables for sensitive data
- [ ] Validate compose file before deployment:
  ```bash
  docker-compose config
  ```
- [ ] Use `env_file` with proper permissions (600 or more restrictive)

### 7.6 Health Checks in Compose
- [ ] Define health checks for all services:
  ```yaml
  services:
    web:
      healthcheck:
        test: ["CMD", "curl", "-f", "http://localhost"]
        interval: 30s
        timeout: 10s
        retries: 3
        start_period: 40s
  ```
- [ ] Set appropriate start periods for initialization
- [ ] Configure dependencies with health check conditions

### 7.7 Deployment Best Practices
- [ ] Use `docker-compose` for development, orchestrators for production
- [ ] Regularly update docker-compose version
- [ ] Review compose file with security scanning tools
- [ ] Implement CI/CD validation for compose files
- [ ] Use build arguments securely (not for secrets)

---

## 8. Security Operations & Maintenance

### 8.1 Regular Updates
- [ ] Update Docker Engine regularly
- [ ] Update base images monthly (or per security policy)
- [ ] Rebuild images after base image updates
- [ ] Subscribe to Docker security advisories
- [ ] Monitor CVE databases for container vulnerabilities

### 8.2 Image and Container Hygiene
- [ ] Remove unused images regularly: `docker image prune -a`
- [ ] Remove stopped containers: `docker container prune`
- [ ] Remove unused volumes: `docker volume prune`
- [ ] Remove unused networks: `docker network prune`
- [ ] Limit number of images per host
- [ ] Track image provenance and history

### 8.3 Monitoring and Auditing
- [ ] Enable Docker daemon audit logging
- [ ] Monitor Docker API calls
- [ ] Track container lifecycle events
- [ ] Implement runtime security monitoring
- [ ] Set up alerts for security events
- [ ] Regular security audits with `docker-bench-security`

### 8.4 Access Control
- [ ] Implement role-based access control (RBAC)
- [ ] Use certificate-based authentication for registry access
- [ ] Regularly review and rotate credentials
- [ ] Audit user access to Docker daemon
- [ ] Implement least privilege principle

### 8.5 Compliance and Validation
- [ ] Run CIS Docker Benchmark tests:
  ```bash
  docker run --rm --net host --pid host --userns host \
    --cap-add audit_control -v /:/host \
    docker/docker-bench-security
  ```
- [ ] Schedule regular compliance checks
- [ ] Document security configurations
- [ ] Maintain security baselines
- [ ] Conduct periodic security reviews

---

## 9. Advanced Security Considerations

### 9.1 Rootless Docker
- [ ] Consider using rootless Docker mode for enhanced security
- [ ] Run Docker daemon as unprivileged user
- [ ] Configure appropriate subuid/subgid mappings
- [ ] Understand rootless mode limitations

### 9.2 Alternative Runtimes
- [ ] Evaluate Podman for daemonless container management
- [ ] Consider gVisor for additional kernel isolation
- [ ] Explore Kata Containers for VM-level isolation

### 9.3 Supply Chain Security
- [ ] Implement image provenance tracking
- [ ] Generate and verify SBOM for all images
- [ ] Use signed images with Notary/TUF
- [ ] Validate dependencies and layers
- [ ] Implement artifact attestation

---

## 10. Quick Security Validation Commands

```bash
# Check Docker daemon configuration
docker info

# List running containers with security status
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Status}}\t{{.Names}}"

# Inspect container security settings
docker inspect <container> --format='{{.HostConfig.Privileged}}'
docker inspect <container> --format='{{.HostConfig.CapDrop}}'
docker inspect <container> --format='{{.HostConfig.SecurityOpt}}'

# Check file permissions
ls -l /etc/docker/daemon.json
ls -l /var/run/docker.sock

# Scan image for vulnerabilities (using Trivy)
trivy image <image-name>

# Run CIS Docker Benchmark
docker run --rm --net host --pid host --userns host \
  --cap-add audit_control -v /:/host \
  docker/docker-bench-security
```

---

## References and Resources

### Official Documentation
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [Docker Security Documentation](https://docs.docker.com/engine/security/)

### Tools
- [Docker Bench Security](https://github.com/docker/docker-bench-security) - Automated CIS benchmark testing
- [Trivy](https://github.com/aquasecurity/trivy) - Vulnerability scanner
- [Snyk](https://snyk.io/) - Container security platform
- [Clair](https://github.com/quay/clair) - Vulnerability static analysis
- [Docker Scout](https://docs.docker.com/scout/) - Docker's native security analysis

### Additional Reading
- [OWASP Docker Top 10](https://owasp.org/www-project-docker-top-10/)
- [Aqua Security - Docker CIS Benchmark](https://www.aquasec.com/cloud-native-academy/docker-container/docker-cis-benchmark/)
- [Docker Security Best Practices (Medium)](https://medium.com/@caring_smitten_gerbil_914/%EF%B8%8F-securing-your-containers-a-deep-dive-into-cis-docker-benchmarks-68efc8eee292)

---

**Document Version:** 1.0
**Last Updated:** 2025-11-29
**Based on:** CIS Docker Benchmark v1.6.0 & OWASP Docker Security Guidelines

---

## Checklist Summary

**Total Security Controls:** 150+

### By Category:
- Installation & Host: 15 controls
- Daemon Configuration: 20 controls
- Files & Permissions: 12 controls
- Images & Build: 25 controls
- Container Runtime: 35 controls
- Networking: 10 controls
- Docker Compose: 25 controls
- Operations: 20 controls
- Advanced: 10 controls

---

**Note:** This checklist should be adapted to your specific environment and security requirements. Not all controls may be applicable or necessary for every deployment. Regularly review and update your security posture as threats evolve.
