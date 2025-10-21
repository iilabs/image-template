# Container Environment Analysis

**Date**: 2025-10-21
**Environment**: gVisor (runsc) sandboxed container with network allow: `*`

## Executive Summary

This document details the capabilities and constraints of the container environment used for building and testing image-template. The environment provides full root access, generous compute resources, and unrestricted network access, but has critical limitations around loop device support due to gVisor sandboxing.

## System Information

- **Kernel**: Linux 4.4.0 (gVisor/runsc)
- **Architecture**: x86_64
- **Container Runtime**: gVisor (runsc) with 9p filesystem
- **User**: root (uid=0, gid=0)
- **Hostname**: runsc

## Capabilities & Resources

### User Permissions

- **Root Access**: Full root privileges
- **Sudo**: Passwordless sudo available
- **Capabilities**: Extensive capability set including:
  - `CAP_SYS_ADMIN` - System administration operations
  - `CAP_MKNOD` - Create device nodes
  - `CAP_NET_ADMIN` - Network administration
  - `CAP_NET_RAW` - Raw network access
  - `CAP_SYS_CHROOT` - chroot operations
  - `CAP_SYS_PTRACE` - Process tracing
  - `CAP_SETUID`, `CAP_SETGID` - Change user/group IDs
  - `CAP_AUDIT_WRITE` - Write audit logs

**Capability Details:**
```
Current: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,
         cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_admin,cap_net_raw,
         cap_sys_chroot,cap_sys_ptrace,cap_sys_admin,cap_mknod,cap_audit_write,
         cap_setfcap=eip

Bounding set: cap_chown,cap_kill,cap_setgid,cap_setuid,cap_net_bind_service,
              cap_sys_chroot,cap_audit_write
```

### Compute Resources

**Memory:**
- Total: 13 GB RAM
- Available: ~12 GB free
- Swap: None (0 B)
- cgroup limit: 9223372036854775807 bytes (~9.2 EB, effectively unlimited)

**CPU:**
- Total CPUs: 16 cores
- Architecture: x86_64
- Threads per core: 1
- Sockets: 1

**Resource Limits (ulimit):**
- Core file size: unlimited
- Data segment size: unlimited
- File size: unlimited
- Open files: 20,000
- Max user processes: unlimited
- Virtual memory: unlimited
- CPU time: unlimited

### Disk & Filesystem

**Storage:**
- Total: 9.8 GB
- Used: 3.2 MB (24 MB in workspace)
- Available: 9.3 GB
- Workspace: `/home/user/image-template`

**Filesystem Types:**
- Root (`/`): 9p (Plan 9 protocol) - read/write
- `/dev`: tmpfs (mode 0755)
- `/dev/shm`: tmpfs (mode 1777)
- `/sys`: sysfs (read-only)
- `/proc`: proc
- `/sys/fs/cgroup`: Various cgroup controllers (cpu, memory, devices, etc.)

**Mount Capabilities:**
- ✅ Can create and mount tmpfs filesystems
- ✅ Full write access to workspace
- ✅ Successfully tested tmpfs mount/umount operations

### Network

**Connectivity:**
- Network policy: Allow `*` (unrestricted)
- DNS: Configured and functional
- Outbound connectivity: Fully functional

**Tested Endpoints:**
- ✅ GitHub (https://github.com): HTTP 200
- ✅ Google (https://www.google.com): HTTP 302
- ✅ Quay.io (https://quay.io): HTTP 200
- ✅ Fedora Registry (https://registry.fedoraproject.org): HTTP 302

**Available Tools:**
- ✅ curl - Fully functional
- ❌ ip - Not available
- ❌ ping - Not available
- ❌ netstat - Not available

### Development Tools

**Available:**
- ✅ git - Version control
- ✅ make - Build automation
- ✅ gcc - C compiler
- ✅ apt - Package manager
- ✅ Standard build toolchain

**Not Available:**
- ❌ podman - Container runtime
- ❌ docker - Container runtime
- ❌ buildah - Container builder
- ❌ skopeo - Container image operations
- ❌ lsmod - Kernel module listing

## Constraints & Limitations

### Critical: Loop Device Support

**Status**: ❌ **NOT FUNCTIONAL**

Loop devices are critical for many image-template operations (mounting ISOs, disk images, etc.) but are **not supported** in this gVisor environment.

**Findings:**
- No loop device nodes exist by default in `/dev/`
- Device nodes can be created manually with `mknod` (e.g., `/dev/loop0`)
- However, `losetup` operations fail with "No such device or address"
- Root cause: gVisor doesn't emulate the loop device kernel module

**Test Results:**
```bash
# Created loop device nodes successfully
mknod /dev/loop0 b 7 0  # ✅ Success

# But losetup fails
losetup /dev/loop0 /tmp/test.img  # ❌ Failed
# Error: losetup: /dev/loop0: failed to set up loop device:
#        No such device or address
```

**Impact:**
- Cannot mount ISO images directly
- Cannot mount disk images for modification
- Cannot use loop-based filesystems
- Many traditional image manipulation workflows blocked

**Workarounds:**
- Use `7z` or similar tools to extract instead of mount
- Use overlay filesystems where possible
- Consider userspace filesystem tools (FUSE)
- Use archive-based approaches rather than block device operations

### Container Runtime Limitations

**Missing Container Tools:**
All standard container tools are unavailable:
- No podman/docker for container operations
- No buildah for building OCI images
- No skopeo for image operations

**Impact:**
- Cannot run nested containers
- Cannot build container images using standard tools
- Must rely on alternative build methods

### Network Tool Limitations

Standard network diagnostic tools are missing:
- No `ip` command for interface management
- No `ping` for connectivity testing
- No `netstat` or `ss` for port inspection
- No `lsmod` for kernel module inspection

**Workarounds:**
- Use `curl` for connectivity testing
- Check `/proc/net/*` directly for network info
- Use alternative tools where available

### gVisor Sandboxing

**Runtime**: gVisor (runsc) provides application kernel layer

**Implications:**
- Not all Linux syscalls are supported
- Kernel modules cannot be loaded
- Some device drivers are emulated or unavailable
- Performance may differ from native containers
- Enhanced security isolation

## Recommendations

### For Image Building Workflows

1. **Avoid loop device dependencies**
   - Use extraction-based workflows instead of mounting
   - Leverage tools like `7z`, `tar`, `cpio` for archive manipulation
   - Consider FUSE-based alternatives where needed

2. **Leverage available resources**
   - 13 GB RAM is sufficient for large builds
   - 16 CPU cores enable parallel operations
   - 9.3 GB disk space adequate for most builds

3. **Network operations**
   - Full network access allows package downloads
   - Can clone repositories directly
   - Can push/pull from registries (if tools available)

4. **Alternative approaches**
   - Use directory-based operations instead of block devices
   - Build from extracted contents rather than mounted images
   - Consider overlay filesystem techniques

### For Testing

1. **Test environment parity**
   - Document differences from production environments
   - Test critical operations that depend on loop devices separately
   - Consider hybrid testing strategies

2. **Workaround validation**
   - Verify extraction-based workflows produce identical results
   - Validate that alternatives maintain compatibility
   - Document any behavioral differences

## Appendix: Raw Test Data

### Capability Test
```bash
$ capsh --print
Current: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,
         cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_admin,cap_net_raw,
         cap_sys_chroot,cap_sys_ptrace,cap_sys_admin,cap_mknod,cap_audit_write,
         cap_setfcap=eip
```

### Memory Test
```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:            13Gi       307Mi        12Gi          0B       126Mi        12Gi
Swap:             0B          0B          0B
```

### Mount Test
```bash
$ mkdir -p /tmp/mount_test && mount -t tmpfs tmpfs /tmp/mount_test
$ umount /tmp/mount_test
# Result: SUCCESS
```

### Loop Device Test
```bash
$ dd if=/dev/zero of=/tmp/test.img bs=1M count=10
$ mknod /dev/loop0 b 7 0
$ losetup /dev/loop0 /tmp/test.img
losetup: /dev/loop0: failed to set up loop device: No such device or address
# Result: FAILED
```

### Network Connectivity Test
```bash
$ curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" https://github.com
HTTP Status: 200

$ curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" https://quay.io
HTTP Status: 200
```

## Container Info

```json
{
  "container_name": "container_011CULVDMsEY6sLuc5q89bJ1--round-best-golden-test",
  "creation_time": 1761056258.1115136
}
```

## Conclusions

This gVisor-based container environment provides:
- ✅ Strong security isolation via gVisor
- ✅ Generous compute and memory resources
- ✅ Full network connectivity
- ✅ Root privileges and extensive capabilities
- ✅ Standard development tools

But has critical limitations:
- ❌ No loop device support (blocks traditional image mounting)
- ❌ No container runtime tools
- ❌ Limited kernel functionality

**For image-template development**: The environment is well-suited for builds that can work with extracted/unpacked contents, but workflows requiring loop device mounting will need alternative approaches.
