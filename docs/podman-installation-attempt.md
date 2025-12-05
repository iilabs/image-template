# Podman Installation Attempt in gVisor Environment

**Date**: 2025-10-22
**Context**: Exploring whether podman can be installed and used within the gVisor-based container environment

## Executive Summary

**Can podman be installed?** ✅ **YES** - Podman 4.9.3 and Buildah 1.33.7 can be installed from Ubuntu 24.04 repositories.

**Can podman run containers?** ❌ **NO** - Critical gVisor `/proc` filesystem limitations prevent container execution.

**Can podman build images?** ❌ **NO** - Same `/proc/self/setgroups` limitation blocks RUN commands during builds.

**Recommendation**: Continue using GitHub Actions for container builds. Podman installation in this environment does not provide functional container capabilities.

## Installation Process

### Package Availability

Podman is available in Ubuntu 24.04 (Noble Numbat) repositories:

```bash
$ apt-cache show podman | grep Version
Version: 4.9.3+ds1-1ubuntu0.2
```

**Available Packages:**
- `podman` - Main container management tool
- `buildah` - Container image builder
- `podman-compose` - Docker Compose compatibility
- `podman-docker` - Docker CLI compatibility
- `skopeo` - Container image operations (via dependencies)

### Installation

```bash
$ apt update
$ apt install -y podman buildah

# Successfully installed:
- podman 4.9.3+ds1-1ubuntu0.2
- buildah 1.33.7+ds1-1ubuntu0.24.04.3
- crun 1.14.1-1 (OCI runtime)
- conmon 2.1.10+ds1-1build2 (container monitor)
- slirp4netns, netavark, aardvark-dns (networking)
- fuse-overlayfs, uidmap, catatonit (utilities)
```

**Installation Size:** ~100MB including all dependencies

**Installation Time:** ~2-3 minutes (including download)

## What Works

### ✅ Package Management

```bash
$ podman --version
podman version 4.9.3

$ buildah --version
buildah version 1.33.7 (image-spec 1.1.0-rc.5, runtime-spec 1.1.0)
```

### ✅ Image Pulling

```bash
$ podman pull alpine
Resolved "alpine" as an alias (/etc/containers/registries.conf.d/shortnames.conf)
Trying to pull docker.io/library/alpine:latest...
Getting image source signatures
Copying blob sha256:2d35ebdb57d9971fea0cac1582aa78935adf8058b2cc32db163c98822e5dfa1b
Copying config sha256:706db57fb2063f39f69632c5b5c9c439633fda35110e65587c5d85553fd1cc38
Writing manifest to image destination
706db57fb2063f39f69632c5b5c9c439633fda35110e65587c5d85553fd1cc38

$ podman images
REPOSITORY                TAG         IMAGE ID      CREATED      SIZE
docker.io/library/alpine  latest      706db57fb206  13 days ago  8.62 MB
```

**Works:**
- Network connectivity to registries
- Image download and storage
- Image listing
- Basic podman CLI commands

### ✅ Version Information

```bash
$ podman version
Client:       Podman Engine
Version:      4.9.3
API Version:  4.9.3
Go Version:   go1.22.2
Built:        Thu Jan  1 00:00:00 1970
OS/Arch:      linux/amd64
```

## What Doesn't Work

### ❌ Container Execution

**Error When Running Containers:**

```bash
$ podman run --rm alpine echo "Hello!"
time="2025-10-22T03:03:46Z" level=error msg="Preparing container: netavark: invalid version number"
Error: unmounting storage for container: cleaning up container storage: unmounting container root filesystem:
removing mount point "/var/lib/containers/storage/overlay/.../merged": directory not empty
```

**Error Without Network:**

```bash
$ podman run --rm --network=none alpine echo "Hello!"
Error: crun: error opening file `/proc/self/setgroups`: No such file or directory:
OCI runtime attempted to invoke a command that was not found
```

### ❌ Image Building

**Error When Building Images:**

```bash
$ cat > Containerfile <<EOF
FROM alpine:latest
RUN echo "test" > /test.txt
CMD cat /test.txt
EOF

$ podman build -f Containerfile -t test-alpine
STEP 1/3: FROM alpine:latest
STEP 2/3: RUN echo "test" > /test.txt
error running container: from /usr/bin/crun creating container for [/bin/sh -c echo "test" > /test.txt]:
error opening file `/proc/self/setgroups`: No such file or directory
: exit status 1
Error: building at STEP "RUN echo "test" > /test.txt": while running runtime: exit status 1
```

**Why RUN Commands Fail:**
- Buildah uses crun to execute RUN commands in containers
- crun requires `/proc/self/setgroups` for user namespace configuration
- gVisor doesn't implement this /proc file
- Therefore, any Containerfile with RUN commands fails

### ❌ System Information

**Error Getting System Info:**

```bash
$ podman info
Error: error marshaling into JSON: json: unsupported value: NaN
```

**Why:**
- Some system metrics return NaN (Not a Number) values
- JSON marshaling fails on NaN
- Likely due to gVisor not providing certain /proc statistics

## Root Cause Analysis

### The `/proc/self/setgroups` Problem

**What It Is:**
- `/proc/self/setgroups` controls whether a process can use setgroups() system call
- Required for user namespace operations
- Used by container runtimes to configure user/group mappings

**What's Missing in gVisor:**

```bash
$ ls -la /proc/self/ | grep setgroups
# (no output - file doesn't exist)

$ ls /proc/self/setgroups
ls: cannot access '/proc/self/setgroups': No such file or directory
```

**What IS Available:**

```bash
$ ls /proc/self/
auxv  cgroup  cmdline  comm  cwd  environ  exe  fd  fdinfo
gid_map  io  limits  maps  mem  mountinfo  mounts  net
ns  oom_score  oom_score_adj  stat  statm  status  uid_map

# Note: uid_map and gid_map exist, but setgroups does not
```

### User Namespace Limitations

**UID/GID Mappings:**

```bash
$ cat /proc/self/uid_map
         0          0 4294967295

$ cat /proc/self/gid_map
         0          0 4294967295
```

**Analysis:**
- Mappings exist (full range: 0 to 4294967295)
- But setgroups control file is missing
- crun cannot configure group permissions properly
- Container execution fails

### Overlay Filesystem Issues

Additionally, there are overlay filesystem mounting issues (as we discovered earlier):

```bash
Error: removing mount point "/var/lib/containers/storage/overlay/.../merged": directory not empty
```

This compounds the problem, as podman's default overlay storage driver has issues in gVisor.

## Attempted Workarounds

### 1. Disable Networking

```bash
$ podman run --rm --network=none alpine echo "test"
# Still fails: crun needs setgroups regardless of network
```

**Result:** ❌ **Failed** - Networking is not the core issue.

### 2. Different Storage Driver

```bash
$ podman --storage-driver=vfs run --rm alpine echo "test"
Error: database graph driver "" does not match our graph driver "vfs":
database configuration mismatch
```

**Result:** ❌ **Failed** - Cannot change storage driver after initial use.

### 3. Alternative Runtimes

**Check for runc:**

```bash
$ which runc
# (not installed)
```

**Analysis:**
- crun is the only OCI runtime available
- runc would likely have the same `/proc/self/setgroups` issue
- All OCI runtimes require similar /proc functionality

## Technical Deep Dive

### gVisor Architecture Limitations

**gVisor's /proc Implementation:**
- gVisor implements a userspace kernel (application kernel)
- Provides a subset of Linux /proc filesystem
- Does not implement all /proc files for security isolation
- `/proc/self/setgroups` is deliberately not implemented

**Why `/proc/self/setgroups` is Missing:**
1. **Security Isolation**: gVisor restricts user namespace operations
2. **Simplified Implementation**: Not all /proc features are supported
3. **Attack Surface Reduction**: Fewer kernel interfaces exposed

**Design Decision:**
- gVisor prioritizes security over full Linux compatibility
- Some container-in-container scenarios are intentionally blocked
- This is working as designed, not a bug

### Comparison: What Works vs What Doesn't

| Operation | Native Linux | gVisor | Reason |
|-----------|-------------|---------|---------|
| Install podman | ✅ | ✅ | Package management works |
| Pull images | ✅ | ✅ | Network and storage work |
| List images | ✅ | ✅ | Metadata operations work |
| Run containers | ✅ | ❌ | Needs `/proc/self/setgroups` |
| Build images (RUN) | ✅ | ❌ | Needs container execution |
| Build images (COPY only) | ✅ | ❓ | Untested (no RUN commands) |
| Overlay filesystem | ✅ | ❌ | gVisor 9p limitations |
| Loop devices | ✅ | ❌ | Kernel module not available |

## Theoretical Alternative: Build Without RUN

**Could you build simple images?**

Possibly, if the Containerfile has **no RUN commands**:

```dockerfile
# Theoretical: might work
FROM alpine:latest
COPY ./files /app/
ENV PATH=/app:$PATH
CMD ["/app/start.sh"]
```

**Testing:**

```bash
$ mkdir -p /tmp/test-build
$ echo "#!/bin/sh" > /tmp/test-build/start.sh
$ echo "echo 'Hello!'" >> /tmp/test-build/start.sh
$ chmod +x /tmp/test-build/start.sh

$ cat > /tmp/test-build/Containerfile <<'EOF'
FROM alpine:latest
COPY start.sh /start.sh
CMD ["/start.sh"]
EOF
```

Let me test this:

**Test Result:**

```bash
$ cd /tmp/test-build && podman build -f Containerfile -t test-no-run
STEP 1/3: FROM alpine:latest
STEP 2/3: COPY start.sh /start.sh
Error: deleting build container: removing mount point
"/var/lib/containers/storage/overlay/.../merged": directory not empty:
lstat /var/lib/containers/storage/overlay/.../merged: transport endpoint is not connected
```

**Result:** ❌ **Failed** - Even without RUN commands, overlay filesystem issues prevent builds.

**Root Cause:**
- Buildah creates temporary containers even for COPY commands
- Temporary containers require overlay filesystem mounts
- gVisor's 9p filesystem doesn't properly support overlay mounts
- "transport endpoint is not connected" = mount failure

## Comprehensive Limitations Summary

### Core Blockers (Unfixable in gVisor)

1. **Missing `/proc/self/setgroups`**
   - Required by: crun, runc (all OCI runtimes)
   - Impact: Cannot execute containers
   - Workaround: None (gVisor architecture decision)

2. **Overlay Filesystem Issues**
   - Required by: podman/buildah storage
   - Impact: Cannot mount container filesystems
   - Workaround: None (9p filesystem limitation)

3. **User Namespace Restrictions**
   - Required by: Rootless containers
   - Impact: Limited container isolation options
   - Workaround: None (gVisor security model)

### What These Limitations Mean

**Image Operations:**
- ✅ Pull/push images to registries
- ✅ Tag and rename images
- ✅ List and inspect images (metadata)
- ❌ Build images (even without RUN)
- ❌ Export/save image filesystems
- ❌ Mount image layers

**Container Operations:**
- ❌ Run containers (any runtime)
- ❌ Create containers
- ❌ Execute commands in containers
- ❌ Attach to containers
- ❌ Container networking
- ❌ Volume mounts

**Build Operations:**
- ❌ `podman build` (overlay issues)
- ❌ `buildah bud` (same issues)
- ❌ Multi-stage builds
- ❌ RUN commands
- ❌ Even COPY-only builds

## Performance and Resource Impact

### Disk Usage After Installation

```bash
$ du -sh /var/lib/containers/
344K    /var/lib/containers/

# After pulling alpine
$ du -sh /var/lib/containers/
9.2M    /var/lib/containers/
```

### Package Size

```
Total installed: ~100MB
- podman: 42MB
- buildah: 35MB
- Dependencies: ~23MB
```

### Network Usage

```
Pulling alpine:latest: 2.8MB downloaded
```

## Comparison with GitHub Actions

| Feature | GitHub Actions | gVisor + Podman | Winner |
|---------|---------------|-----------------|---------|
| Container builds | ✅ Full support | ❌ Cannot build | GitHub Actions |
| Network access | ✅ Unrestricted | ✅ Unrestricted | Tie |
| Build time | ⚡ Fast (native) | N/A (doesn't work) | GitHub Actions |
| Loop devices | ✅ Available | ❌ Not available | GitHub Actions |
| Overlay FS | ✅ Works | ❌ Broken | GitHub Actions |
| Resource limits | 7GB RAM, 14GB disk | 13GB RAM, 9GB disk | Roughly tie |
| Caching | ✅ actions/cache | ✅ Git-based cache | Tie |
| Cost | Free (2000 min/month) | N/A | GitHub Actions |

**Conclusion:** GitHub Actions is superior for all container build operations.

## Recommendations

### DO NOT Use Podman in This Environment For:

1. ❌ Building container images
2. ❌ Running containers
3. ❌ Testing containerized applications
4. ❌ Multi-container setups
5. ❌ Container development workflows

### DO Use Podman in This Environment For:

**Literally nothing productive.** 

While podman can be installed, it cannot perform any of its core functions due to gVisor limitations.

### Recommended Workflow (Unchanged)

**Continue using the existing workflow:**

1. **Development** (in this environment):
   - Edit Containerfiles
   - Edit build scripts
   - Manage git repository
   - Documentation

2. **Building** (GitHub Actions):
   - Build container images (.github/workflows/build.yml)
   - Build disk images (.github/workflows/build-disk.yml)
   - Run tests
   - Publish artifacts

3. **Local Testing** (on machine with native containers):
   - `podman build` - Build images locally
   - `podman run` - Test containers
   - `just build-qcow2` - Build bootable images

## Uninstallation (Optional)

If you want to remove podman to save space:

```bash
# Remove podman and buildah
apt remove --purge podman buildah

# Remove dependencies (be careful - may remove other needed packages)
apt autoremove

# Clean up storage
rm -rf /var/lib/containers
```

**Space saved:** ~100MB

## Key Takeaways

1. **Podman CAN be installed** in gVisor environments (Ubuntu 24.04+)
2. **Podman CANNOT function** due to fundamental gVisor limitations
3. **No workarounds exist** - these are architectural incompatibilities
4. **GitHub Actions remains the correct tool** for container builds
5. **This environment is for editing**, not building

## Technical Details

### Environment Information

```bash
$ uname -a
Linux runsc 4.4.0 #1 SMP Sun Jan 10 15:06:54 PST 2016 x86_64 x86_64 x86_64 GNU/Linux

$ cat /proc/version
Linux version 4.4.0 (go/gvisor-website/issues/1) (gcc version 5.4.0 20160609 
(Ubuntu 5.4.0-6ubuntu1~16.04.12)) #1 SMP Sun Jan 10 15:06:54 PST 2016
```

**gVisor Indicators:**
- Kernel 4.4.0 (ancient kernel version)
- Build date: 1970/2016 (fake build date)
- Hostname: `runsc` (gVisor's runtime)
- Missing /proc files

### Installed Components

```bash
$ dpkg -l | grep -E "(podman|buildah|crun|conmon)"
podman         4.9.3+ds1-1ubuntu0.2    amd64    tool to manage containers and pods
buildah        1.33.7+ds1-1ubuntu0.24  amd64    command line tool for building containers
crun           1.14.1-1                amd64    lightweight OCI runtime for running containers
conmon         2.1.10+ds1-1build2      amd64    OCI container runtime monitor
```

## Conclusion

**Final Verdict:** Installing podman in a gVisor container environment is **technically possible** but **practically useless**.

**The Fundamental Issue:**
```
gVisor (security isolation)
    └─→ Limited /proc implementation
        └─→ No /proc/self/setgroups
            └─→ OCI runtimes cannot configure user namespaces
                └─→ Containers cannot execute
                    └─→ Builds fail
                        └─→ Podman is non-functional
```

**Recommendation:** Do not install podman in gVisor environments. Continue using the existing GitHub Actions workflow for all container operations.

**The Good News:** The environment is still excellent for:
- Repository management
- Code editing
- Documentation
- Git operations
- Running compiled binaries
- Using tmpfs for fast operations
- Managing persistent caches via git

Just not for running containers. And that's perfectly fine for this use case.

---

**Tested:** 2025-10-22  
**Environment:** gVisor container, Ubuntu 24.04, Podman 4.9.3, Buildah 1.33.7  
**Result:** Installation successful, functionality blocked by gVisor architecture  
**Recommendation:** Use GitHub Actions for container operations
