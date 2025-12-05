# Community Solutions: Container Builds in Restricted Environments

**Date**: 2025-10-22
**Context**: Research on how the community handles container builds in sandboxed, restricted, or gVisor environments

## Executive Summary

After researching community approaches to building containers in restricted environments, several clear patterns emerge:

1. **Most people avoid nested containers** - Use the host's container runtime instead
2. **Daemonless builders are popular** - Kaniko, BuildKit, and Buildah for CI/CD
3. **Remote builds are standard** - GitHub Actions, Cloud Build, GitLab CI
4. **Alternative paradigms exist** - Nix, WebAssembly for reproducible builds

**Key Insight:** The industry consensus is to **separate the development environment from the build environment**. This aligns perfectly with our current approach using GitHub Actions.

## 1. gVisor + Containers: Community Experience

### Current Status (2024-2025)

**Can you run containers in gVisor?** Sometimes, with limitations.

**gVisor GitHub Issue #311:** "runsc doesn't work with rootless podman"
- **Status**: Open since 2018
- **Problem**: gVisor works with sudo but panics with rootless Podman
- **Community consensus**: Use sudo or avoid nested containers

### What Works

✅ **gVisor as container runtime** (running gVisor containers from host)
```bash
# On the host, use gVisor to run containers
docker run --runtime=runsc myimage
```

This is the **recommended approach**: gVisor runs containers, not inside containers.

### What Doesn't Work

❌ **gVisor inside containers** (nested containerization)
- Missing `/proc/self/setgroups` blocks OCI runtimes
- Overlay filesystem issues
- Process limit exhaustion
- Security model prevents it

### Community Workarounds

**1. Mount Docker Socket (Most Common)**

Instead of Docker-in-Docker, mount the host's Docker socket:

```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock myimage
```

**Benefits:**
- Containers built on host (native performance)
- No nested virtualization
- Works with gVisor

**Drawbacks:**
- Security risk (container accesses host Docker)
- Not isolated
- Requires privileged access to socket

**Use cases:** Trusted CI/CD environments, development workflows

**2. Separate Build Containers**

Run containers at the same level, not nested:

```bash
# Instead of: container → docker → build
# Do: container1 (dev) + container2 (build) via shared socket
```

**3. Force Different Runtimes**

Use gVisor for app containers, runc for system containers:

```bash
# App container (gVisor)
docker run --runtime=runsc app:latest

# Build container (runc - default)
docker run --privileged buildah build .
```

## 2. Daemonless Container Builders

The community has moved toward **daemonless builders** for restricted environments.

### Kaniko (Google, archived 2025)

**Status**: 🟡 **Archived** (Google, June 2025), but **forked** by Chainguard for maintenance

**What it is:**
- Builds container images without Docker daemon
- Executes Dockerfile commands in userspace
- Designed for Kubernetes/restricted environments

**How it works:**
```dockerfile
# Kaniko runs inside a container
FROM gcr.io/kaniko-project/executor:latest AS builder

# Build another image from within Kaniko container
COPY . /workspace
RUN /kaniko/executor \
    --dockerfile=/workspace/Dockerfile \
    --context=/workspace \
    --destination=myregistry/myimage:tag
```

**Advantages:**
- ✅ No Docker daemon required
- ✅ Rootless (unprivileged)
- ✅ Works in Kubernetes pods
- ✅ Caching support

**Disadvantages:**
- ❌ Archived by Google (community forks exist)
- ❌ Slower than BuildKit (linear, not parallel)
- ❌ Still requires some kernel features (won't work in gVisor)

**Current recommendation**: Migrate to BuildKit (2025 consensus)

### BuildKit (Docker, actively maintained)

**Status**: ✅ **Active**, recommended replacement for Kaniko

**What it is:**
- Modern build backend for Docker
- Can run daemonless and rootless
- Parallel builds, advanced caching

**How it works:**
```bash
# Standalone BuildKit (no Docker daemon)
buildctl build \
    --frontend dockerfile.v0 \
    --local context=. \
    --local dockerfile=. \
    --output type=image,name=myimage:tag
```

**Or with buildx:**
```bash
docker buildx build \
    --builder=container \
    --platform linux/amd64,linux/arm64 \
    -t myimage:tag .
```

**Advantages:**
- ✅ 30-50% faster than Kaniko (parallel stages)
- ✅ Better caching (layer-aware)
- ✅ Multi-platform builds
- ✅ Actively maintained
- ✅ Rootless mode available

**Disadvantages:**
- ❌ More complex setup than Kaniko
- ❌ Still requires kernel features (won't work in gVisor)

**Current status**: Industry standard for CI/CD (2025)

### Buildah (Red Hat, actively maintained)

**Status**: ✅ **Active**, popular in Red Hat/Fedora ecosystems

**What it is:**
- Daemonless image builder
- Script-based or Dockerfile-based builds
- Part of Podman ecosystem

**How it works:**
```bash
#!/bin/bash
# Script-based build (no Dockerfile)
container=$(buildah from fedora)
buildah run $container dnf install -y httpd
buildah copy $container ./index.html /var/www/html/
buildah config --cmd /usr/sbin/httpd $container
buildah commit $container mywebserver
```

**Or with Dockerfile:**
```bash
buildah bud -f Dockerfile -t myimage .
```

**Advantages:**
- ✅ No daemon required
- ✅ Rootless builds
- ✅ Script-based flexibility (beyond Dockerfiles)
- ✅ Fine-grained control

**Disadvantages:**
- ❌ Red Hat ecosystem focus
- ❌ Less popular than Docker/BuildKit
- ❌ Still requires kernel features (won't work in gVisor)

**Use cases:** Rootless CI/CD, Kubernetes, Red Hat environments

### Summary: Daemonless Builders in gVisor

**Do they work in gVisor?** ❌ **NO**

All three (Kaniko, BuildKit, Buildah) still require:
- User namespaces (`/proc/self/setgroups`)
- Overlay filesystem support
- OCI runtime (crun/runc)

These are not available in gVisor environments. They work in **Kubernetes** but not **gVisor containers**.

## 3. Remote Build Services (Recommended Approach)

The community consensus: **Build remotely, develop locally**.

### GitHub Actions (Most Popular)

**What it provides:**
- Native Linux runners with Docker/Podman
- 2,000 free minutes/month
- GitHub Container Registry (GHCR)
- Integrated CI/CD

**Example workflow:**
```yaml
name: Build Container
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t myimage .
      - name: Push to GHCR
        run: |
          echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}:latest
```

**Advantages:**
- ✅ Free for public repos
- ✅ Full Docker/BuildKit support
- ✅ No local setup required
- ✅ Integrated with GitHub
- ✅ Works with any development environment

**This is what image-template already uses!** ✅

### Google Cloud Build

**What it provides:**
- Scalable build infrastructure
- Multi-cloud support
- Advanced caching
- Integration with GCR/Artifact Registry

**Example:**
```yaml
# cloudbuild.yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/myimage', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/myimage']
```

**Trigger from GitHub Actions:**
```yaml
- name: Trigger Cloud Build
  uses: google-github-actions/setup-gcloud@v1
  with:
    service_account_key: ${{ secrets.GCP_SA_KEY }}
- run: gcloud builds submit --tag gcr.io/project/myimage
```

**Advantages:**
- ✅ Highly scalable
- ✅ Fast builds (multi-core)
- ✅ Advanced caching
- ✅ Pay-as-you-go

**Disadvantages:**
- ❌ Costs money (after free tier)
- ❌ More complex setup

### GitLab CI/CD

**What it provides:**
- Integrated CI/CD
- Container Registry
- Kubernetes integration

**Example:**
```yaml
# .gitlab-ci.yml
build:
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

**Advantages:**
- ✅ All-in-one platform
- ✅ Self-hosted option
- ✅ Free tier available

### Comparison: Remote Build Services

| Service | Free Tier | Speed | Caching | Ease of Use |
|---------|-----------|-------|---------|-------------|
| GitHub Actions | 2000 min/mo | Fast | Good | ⭐⭐⭐⭐⭐ Excellent |
| Cloud Build | 120 min/day | Very Fast | Excellent | ⭐⭐⭐⭐ Good |
| GitLab CI | 400 min/mo | Fast | Good | ⭐⭐⭐⭐ Good |

**Recommendation:** GitHub Actions (already in use) is the best choice for open source.

## 4. Cloud Development Environments

How do other cloud IDEs handle container builds?

### GitHub Codespaces

**Container Build Support:**
- ✅ Docker-in-Docker (DinD) available
- ✅ Full Docker daemon access
- ✅ Privileged containers supported

**Setup:**
```json
// .devcontainer/devcontainer.json
{
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  },
  "runArgs": ["--privileged"]
}
```

**How it works:**
- Codespaces run on Azure VMs (not gVisor)
- Full Linux kernel access
- Can mount Docker socket
- Can run DinD containers

**Implication:** Codespaces are NOT as restricted as gVisor environments.

### Gitpod

**Container Build Support:**
- ✅ Docker builds supported
- ✅ Dockerfile customization
- ✅ docker-compose support

**Example:**
```yaml
# .gitpod.yml
image:
  file: .gitpod.Dockerfile

tasks:
  - name: Build Container
    command: docker build -t myimage .
```

**How it works:**
- Gitpod workspaces run in Kubernetes
- Docker daemon available
- Can run privileged containers
- Rebuild images automatically

**Implication:** Gitpod allows container builds that gVisor blocks.

### Comparison: Cloud IDEs

| Platform | Runtime | Docker Available | Nested Containers |
|----------|---------|------------------|-------------------|
| GitHub Codespaces | Azure VM | ✅ Yes | ✅ Yes (DinD) |
| Gitpod | Kubernetes | ✅ Yes | ✅ Yes |
| **gVisor (this env)** | **Sandboxed** | **❌ Limited** | **❌ No** |

**Key difference:** Most cloud IDEs use **VMs or Kubernetes**, not gVisor's tight sandboxing.

## 5. Alternative Build Paradigms

Some communities avoid containers entirely.

### Nix: Declarative Container Images

**What it is:**
- Functional package manager
- Reproducible builds
- Can build OCI images without Docker

**How it works:**
```nix
# flake.nix
{
  outputs = { self, nixpkgs }: {
    packages.x86_64-linux.myimage = nixpkgs.dockerTools.buildLayeredImage {
      name = "myimage";
      tag = "latest";
      contents = [ nixpkgs.bash nixpkgs.coreutils ];
      config.Cmd = [ "/bin/bash" ];
    };
  };
}
```

**Build it:**
```bash
nix build .#myimage
docker load < result
```

**Advantages:**
- ✅ No Docker daemon required
- ✅ Deterministic builds
- ✅ Incremental layers (efficient)
- ✅ Minimal images (only dependencies)
- ✅ **Could work in restricted environments**

**Disadvantages:**
- ❌ Steep learning curve
- ❌ Different paradigm from Dockerfiles
- ❌ Nix ecosystem required

**Community interest:** Growing rapidly (Replit, Flox, etc.)

### WebAssembly (WASM)

**What it is:**
- Lightweight sandboxed execution
- Near-native speed
- Cross-platform

**How it works:**
```bash
# Compile to WASM
cargo build --target wasm32-wasi

# Run with wasmtime (no containers)
wasmtime app.wasm
```

**Advantages:**
- ✅ No container runtime needed
- ✅ Lightweight (KBs not MBs)
- ✅ Secure sandbox
- ✅ **Works in restricted environments**

**Disadvantages:**
- ❌ Limited language support (Rust, C, Go, AssemblyScript)
- ❌ Not for all workloads
- ❌ Ecosystem still maturing

**Use cases:** Serverless, edge computing, plugins

### Comparison: Alternative Paradigms

| Approach | Container Images | Build Tool | Works in gVisor? |
|----------|------------------|------------|------------------|
| Nix | ✅ Yes (OCI) | Nix flakes | ⚠️ Maybe (needs testing) |
| WASM | ❌ No (WASM modules) | Cargo/TinyGo | ✅ Likely yes |
| Traditional | ✅ Yes | Docker/Podman | ❌ No |

## 6. What Should We Do?

Based on community research, here's the recommended approach:

### ✅ Current Approach (Keep This!)

**Development Environment** (gVisor/restricted):
- Edit code, Containerfiles, configs
- Manage git repository
- Use fast tmpfs for temp operations
- Implement git-based caching

**Build Environment** (GitHub Actions):
- Build container images
- Build bootable disk images
- Run tests
- Publish artifacts

**This is the industry standard approach!** ✅

### ⚠️ Consider for Future

**1. Nix for Reproducible Builds**

If determinism is critical:
- Use Nix flakes instead of Dockerfiles
- Build OCI images with `dockerTools.buildLayeredImage`
- Never needs Docker daemon
- Fully reproducible

**Example migration:**
```nix
# Instead of Containerfile
# FROM ghcr.io/ublue-os/bazzite:stable
# RUN dnf install -y tmux
# ...

# Use flake.nix
{
  outputs = { nixpkgs, ... }: {
    packages.x86_64-linux.image-template =
      nixpkgs.dockerTools.buildLayeredImage {
        name = "image-template";
        contents = [ nixpkgs.tmux nixpkgs.podman ];
        # ... rest of config
      };
  };
}
```

**Benefit:** Could potentially build images in gVisor (needs testing).

**2. Remote BuildKit**

For faster CI/CD builds:
- Use BuildKit in GitHub Actions
- 30-50% faster than current builds
- Better caching

**Example:**
```yaml
# .github/workflows/build.yml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build with BuildKit
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Benefit:** Faster builds, better caching than standard Docker builds.

## 7. Summary of Findings

### What Doesn't Work in gVisor

| Tool | Works? | Reason |
|------|--------|--------|
| Docker/Podman inside gVisor | ❌ No | Missing /proc/self/setgroups, overlay FS issues |
| Kaniko | ❌ No | Needs user namespaces |
| BuildKit | ❌ No | Needs OCI runtime |
| Buildah | ❌ No | Needs user namespaces |

### What Works Elsewhere

| Approach | Environment | How They Do It |
|----------|-------------|----------------|
| GitHub Codespaces | Azure VMs | Docker-in-Docker, full kernel |
| Gitpod | Kubernetes | Docker daemon, privileged pods |
| Cloud Build | GCP VMs | Native Docker on VMs |
| Nix | Any | No daemon, pure functional builds |

### Recommended Solutions (Priority Order)

1. **✅ GitHub Actions** (already implemented)
   - Build container images
   - Build disk images
   - Publish to GHCR

2. **✅ Git-based caching** (documented in persistent-cache-strategy.md)
   - 70-85% faster subsequent builds
   - Works in any environment

3. **⚠️ Consider Nix** (advanced users)
   - Deterministic builds
   - No Docker daemon needed
   - May work in gVisor (untested)

4. **⚠️ Upgrade to BuildKit** (optimization)
   - Faster CI/CD builds
   - Better caching
   - Easy migration

## 8. Key Takeaways

**The Community Consensus:**

1. **Separate dev from build environments** - Don't try to build containers where you develop
2. **Use remote CI/CD** - GitHub Actions, Cloud Build, GitLab CI
3. **Daemonless builders for Kubernetes** - Kaniko/BuildKit in k8s pods, not in dev containers
4. **Cloud IDEs provide more access** - Codespaces/Gitpod use VMs, not gVisor
5. **Alternative paradigms exist** - Nix and WASM for special needs

**For This Repository:**

✅ **Current approach is optimal** - Editing code locally, building remotely
✅ **Aligns with industry best practices** - Separation of concerns
✅ **No changes needed** - GitHub Actions already handles builds
✅ **Caching documented** - Git-based persistence strategy ready

**What We Learned:**

- gVisor limitations are **by design** (security isolation)
- Most cloud IDEs **don't use gVisor** (use VMs instead)
- Industry moved to **remote builds** (not local nested containers)
- **Our workflow is already correct** ✅

## 9. References

### GitHub Issues
- gVisor #311: "runsc doesn't work with rootless podman" (open since 2018)
- Podman #6699: "Gvisor support? Podman compatibility with gvisor"
- Podman #16877: "FR/PR: gVisor/runsc-Support for Podman"

### Documentation
- gVisor FAQ: https://gvisor.dev/docs/user_guide/faq/
- GitHub Codespaces Dev Containers: https://docs.github.com/en/codespaces
- Kaniko GitHub: https://github.com/GoogleContainerTools/kaniko
- BuildKit Documentation: https://docs.docker.com/build/buildkit/
- Nix dockerTools: https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-dockerTools

### Community Articles
- "Rootless container builds on Kubernetes" (CERN, 2025)
- "Why BuildKit Replaces Kaniko in Modern CI/CD" (2025)
- "Building containers without Docker" (Alex Ellis)
- "Use flake.nix, not Dockerfile" (Community discussion)
- "Safe Ride into the Dangerzone: Reducing attack surface with gVisor" (2024)

---

**Research Date:** 2025-10-22
**Conclusion:** Current approach (GitHub Actions + local editing) matches industry best practices. No changes recommended.
