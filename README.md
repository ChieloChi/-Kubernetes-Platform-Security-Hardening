# -Kubernetes-Platform-Security-Hardening

> **Context:** This repository documents a platform-wide security hardening initiative I led at [EvoCloud](https://evocloud.dev). The implementation lives in EvoCloud's public monorepo: [`evocloud-saas`](https://github.com/evocloud-dev/evocloud-saas).

---

## Overview

Most Kubernetes security guides tell you what to configure. This documents *why each control exists at the kernel level*, what threat it mitigates, and what the right engineering decision is when it cannot be fully applied.

Over the course of this initiative, I hardened **15+ production deployments**  spanning web servers, background workers, queue processors, cache layers, databases, and search engines — by systematically enforcing Linux kernel security primitives at the pod and container level. The work was not about chasing a score. It was about building a consistent, auditable, and defensible security posture across an entire platform, where every relaxed control has a documented reason and every retained control has a clear purpose.

The deployments covered include: Twenty CRM, OpenProject, Mastodon, Zammad, Plane Enterprise, Saleor, Hoppscotch, Cal.com, Docmost, Hi.Events, Listmonk, Umami, Chatwoot, ERPNext, Paperless-ngx, Nextcloud, and more.

---

## What Was Hardened & Why

Every control maps directly to a Linux kernel security primitive, Kubernetes is just the orchestration layer. Enforcement happens at the kernel level on the node.

### 1. Capability Dropping (`CAP_DROP: ALL`)

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
```

Linux splits root's privileges into ~40 granular capabilities (`NET_BIND_SERVICE`, `SYS_ADMIN`, `CHOWN`, etc.). By dropping ALL, the container process loses every elevated kernel privilege. If an exploit gains code execution inside the container, it cannot escalate — there are no capabilities to abuse.

---

### 2. Read-Only Root Filesystem

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

Uses the Linux `MS_RDONLY` mount flag. The kernel refuses any write syscall to that mount point, regardless of what the process requests. This prevents malicious binaries being written to `PATH`, runtime modification of application code, and post-deployment config tampering. Writable paths applications legitimately need (e.g. `/tmp`, cache dirs) are explicitly mounted as `emptyDir` volumes.

---

### 3. UID/GID Isolation

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser:    10001
  runAsGroup:   10001
  fsGroup:      10001
```

Enforces Linux user/group ownership at the kernel level. Running as a high-UID non-root user (>10000) avoids conflicts with host system users and reduces the blast radius of any container escape. `fsGroup` ensures mounted volumes are accessible to the correct group without granting broad permissions.

---

### 4. No Service Account Token Automounting

```yaml
spec:
  automountServiceAccountToken: false
```

By default, Kubernetes mounts a service account token into every pod, giving any process inside the container direct access to the Kubernetes API. For application workloads that don't need cluster API access, this is an unnecessary attack surface. Disabling it prevents a compromised container from querying or manipulating the cluster.

---

### 5. Seccomp (`RuntimeDefault`)

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

Seccomp filters **system calls** at the kernel level. `RuntimeDefault` uses the containerd built-in profile which blocks ~44 dangerous syscalls including `ptrace`, `reboot`, and `kexec_load` — always available on any Kubernetes node without requiring custom profile deployment.

---

### 6. Resource Limits & Requests

```yaml
resources:
  requests:
    cpu:    "500m"
    memory: "512Mi"
  limits:
    cpu:    "2"
    memory: "2Gi"
```

Prevents CPU/memory-based DoS and ensures fair scheduling across the multi-tenant cluster. Without limits, a single runaway container can degrade all co-located workloads.

---

## Tradeoffs

The real engineering is not in applying controls — it is in knowing when a control must be relaxed, understanding exactly why, and being explicit about what compensating controls remain in place.

The clearest example from this work is the **Twenty CRM server**. The upstream Docker image ships all application files owned by UID `1000` (the `node` user). At startup, the NestJS server must write the runtime `SERVER_URL` into `dist/front/index.html` — without this, the frontend displays "Unable to Reach Back-end". Tracing the failure required inspecting the `generateFrontConfig()` routine in the compiled server bundle and identifying that the process lacked write permission due to a UID mismatch between the running container (UID `10001`) and the file owner (UID `1000`).

The resolution required relaxing two controls: `readOnlyRootFilesystem: false` and `runAsUser: 1000`. All other controls — `runAsNonRoot`, `CAP_DROP: ALL`, `allowPrivilegeEscalation: false`, resource limits — remain enforced. The tradeoff is documented explicitly in the module so the reasoning is permanently auditable.

This is the work that matters: not perfect scores, but principled decisions with clear justification.

---

## Repository Reference

**[github.com/evocloud-dev/evocloud-saas](https://github.com/evocloud-dev/evocloud-saas)**

Key modules to explore:
- `twenty-crm-timoni/` — Twenty CRM with documented server tradeoff
- `openproject-timoni/` — OpenProject with seccomp enforcement
- `mastodon-timoni/` — Mastodon full-stack hardening
- `zammad-timoni/` — Zammad multi-component hardening
- `plane-enterprise-timoni/` — Plane Enterprise full hardening
