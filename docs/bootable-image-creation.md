# Bootable Image Creation: Current Approach and Alternatives

**Date**: 2025-10-21
**Context**: Analysis of image-template's bootable disk image creation process and evaluation of alternatives for constrained environments

## Executive Summary

The image-template repository uses **bootc-image-builder (BIB)** to convert OCI container images into bootable disk images (QCOW2, RAW, ISO). This process has a critical dependency on **loop devices**, which are not available in gVisor-based sandboxed container environments. This document analyzes the current approach, identifies the blockers, and explores potential alternatives.

## Current Approach: bootc-image-builder

### Overview

bootc-image-builder is the standard tool for creating bootable disk images from bootc-compatible container images. It's a containerized wrapper around osbuild that generates various disk image formats.

### How It Works

1. **Container Execution**: BIB runs as a privileged container
2. **Manifest Generation**: Creates osbuild manifest files defining the build pipeline
3. **Disk Image Creation**: Creates a raw disk image file (disk.img)
4. **Loop Device Setup**: Attaches disk.img to a loop device to treat it as a block device
5. **Partitioning**: Uses standard partitioning tools on the loop device
6. **Filesystem Creation**: Formats partitions with appropriate filesystems (ext4, btrfs, etc.)
7. **Mount Setup**: Mounts all required partitions (/boot, /boot/efi, /)
8. **bootc Installation**: Calls `bootc install to-filesystem` to install the container image
9. **Bootloader Installation**: Installs GRUB to the MBR
10. **Conversion**: Converts raw image to target format (QCOW2, ISO, etc.)

### image-template Implementation

From the Justfile (lines 161-188):

```bash
_build-bib $target_image $tag $type $config:
    #!/usr/bin/env bash
    set -euo pipefail

    args="--type ${type} "
    args+="--use-librepo=True "
    args+="--rootfs=btrfs"

    BUILDTMP=$(mktemp -p "${PWD}" -d -t _build-bib.XXXXXXXXXX)

    sudo podman run \
      --rm \
      -it \
      --privileged \
      --pull=newer \
      --net=host \
      --security-opt label=type:unconfined_t \
      -v $(pwd)/${config}:/config.toml:ro \
      -v $BUILDTMP:/output \
      -v /var/lib/containers/storage:/var/lib/containers/storage \
      "${bib_image}" \
      ${args} \
      "${target_image}:${tag}"
```

### Required Capabilities

**System Requirements:**
- Privileged container execution (`--privileged`)
- Access to `/var/lib/containers/storage` (for accessing container images)
- SELinux unconfined context or appropriate osbuild SELinux policies
- **Loop device support** (CRITICAL)
- Block device access (`/dev`)

**Container Flags:**
```bash
--privileged                                # Full system access
--security-opt label=type:unconfined_t     # SELinux bypass
-v /var/lib/containers/storage:/...         # Container image access
```

### Supported Output Formats

- **qcow2**: QEMU/KVM virtual machine images
- **raw**: Raw disk images (can be dd'd to physical media)
- **anaconda-iso**: Fedora/RHEL installer ISO
- **ami**: Amazon Machine Images
- **vmdk**: VMware images
- **vhd**: Hyper-V images
- **gce**: Google Compute Engine images

## Loop Device Dependency Analysis

### Why Loop Devices Are Required

Loop devices allow a file to be treated as a block device, which is essential for:

1. **Disk Image Manipulation**: Working with disk.img files as if they were real disks
2. **Partitioning**: Using fdisk/sfdisk/parted on the file
3. **Filesystem Operations**: Formatting and mounting individual partitions
4. **Bootloader Installation**: Writing GRUB to the MBR requires block device access

### The Technical Flow

```
disk.img (file)
    ↓
losetup /dev/loop0 disk.img
    ↓
/dev/loop0 (block device)
    ↓
fdisk /dev/loop0 (create partitions)
    ↓
/dev/loop0p1, /dev/loop0p2 (partition devices via kpartx)
    ↓
mkfs.ext4 /dev/loop0p1
    ↓
mount /dev/loop0p1 /mnt/boot
    ↓
bootc install to-filesystem /mnt
    ↓
grub-install --target=x86_64-efi /dev/loop0
```

### Why gVisor Doesn't Support Loop Devices

gVisor provides application kernel-level sandboxing:
- Implements a subset of Linux kernel functionality in userspace
- Doesn't support kernel modules (loop.ko is a kernel module)
- Loop device operations require kernel syscalls that gVisor doesn't fully emulate
- Even when device nodes are created with `mknod`, the underlying kernel support is missing

From our testing:
```bash
$ mknod /dev/loop0 b 7 0       # ✅ Creates device node
$ losetup /dev/loop0 test.img  # ❌ Failed: No such device or address
```

## Alternative Approaches

### Option 1: Use GitHub Actions for Disk Image Builds

**Status**: ✅ **Already Implemented**

The repository already has `.github/workflows/build-disk.yml` that builds disk images in GitHub Actions runners, which have full loop device support.

**Pros:**
- Already working and tested
- Full hardware support (native Linux kernel)
- Handles heavy builds with adequate resources
- Produces artifacts automatically

**Cons:**
- Cannot build locally in constrained environments
- Requires pushing to GitHub to trigger builds
- CI minutes usage

**Use Cases:**
- Production releases
- Official builds
- Automated testing

### Option 2: podman-bootc

**Status**: ❌ **Not Compatible** (requires loop devices)

A simpler CLI tool for running bootc images in VMs.

**How it works:**
```bash
podman-bootc run localhost/my-image:latest
```

Under the hood, it uses `bootc install to-disk --via-loopback` which still requires loop device support.

**Verdict**: Not suitable for gVisor environments.

### Option 3: bootc install to-disk / to-filesystem

**Status**: ❌ **Not Compatible** (requires loop devices or real block devices)

Direct usage of bootc's installation commands.

**to-disk variant:**
```bash
podman run --privileged --pid=host \
  -v /dev:/dev \
  -v /var/lib/containers:/var/lib/containers \
  my-bootc-image \
  bootc install to-disk /dev/sdX
```

**to-filesystem variant:**
```bash
# Requires pre-partitioned and mounted filesystem
bootc install to-filesystem \
  --karg=root=UUID=xxx \
  /mnt
```

**Requirements:**
- `to-disk`: Needs real block device or loop device
- `to-filesystem`: Needs pre-created partitions and mounts (still requires loop devices for disk images)
- Both need `--privileged` and full `/dev` access

**Verdict**: Both variants require capabilities not available in gVisor.

### Option 4: mkosi (Make Operating System Image)

**Status**: ⚠️ **Possibly Compatible** (needs investigation)

mkosi builds bootable images from distribution packages rather than container images.

**Different Paradigm:**
- Builds from RPM/DEB packages, not container images
- Creates Discoverable Disk Images (DDIs)
- Supports container, VM, and bare-metal boot

**Example mkosi.conf:**
```ini
[Output]
Format=disk
ImageId=my-os

[Content]
Packages=
  systemd
  kernel
  grub

BaseTrees=/path/to/container/rootfs
```

**Pros:**
- More flexible build approach
- Can work with extracted container contents
- Supports multiple output formats

**Cons:**
- Different workflow (not bootc-native)
- May still require loop devices for disk image creation
- Requires extracting container image first
- Steeper learning curve

**Verdict**: Might work with extracted container contents, but likely still hits loop device limitations for disk image formats.

### Option 5: Container Image as Deliverable

**Status**: ✅ **Fully Compatible**

Skip disk image creation entirely and deliver the OCI container image.

**Approach:**
Users can pull the container image and create disk images on their local machines:

```bash
# User's machine (with loop device support)
podman pull ghcr.io/username/image-template:latest

# Build disk image locally
sudo podman run --rm --privileged \
  --pull=newer \
  --security-opt label=type:unconfined_t \
  -v $(pwd)/output:/output \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  --type qcow2 \
  --local \
  ghcr.io/username/image-template:latest
```

**Pros:**
- No loop device requirements in CI/build environment
- Users create images in their preferred format
- Faster CI builds (just container image build)
- Smaller artifact storage (container layers vs full disk images)

**Cons:**
- Users must create disk images themselves
- Requires users to have loop device support locally
- Extra step for end users
- Need to document the process clearly

**Use Cases:**
- Development and testing in constrained environments
- Users who want to customize disk layout
- Building in restricted cloud environments

### Option 6: Nested Virtualization with QEMU/KVM

**Status**: ❌ **Not Compatible** (requires /dev/kvm)

Run a full VM inside the container to build disk images.

**Requirements:**
- `/dev/kvm` access
- Nested virtualization support
- Significant memory overhead

**Verdict**: gVisor environments typically don't provide KVM access.

### Option 7: NBD (Network Block Device) Approach

**Status**: ⚠️ **Experimental** (untested in gVisor)

Use network block devices instead of loop devices.

**Concept:**
```bash
# Create NBD server for disk image
qemu-nbd -c /dev/nbd0 disk.img

# Treat NBD device like loop device
fdisk /dev/nbd0
```

**Requirements:**
- NBD kernel module (nbd.ko)
- QEMU utilities (qemu-nbd)

**Verdict**: Likely blocked by same gVisor limitations (kernel module support).

## Recommendations

### For Development in Constrained Environments

**Primary Strategy**: Build container images only, defer disk image creation

1. **Local Development**:
   - Build and test OCI container images
   - Use `podman run` for functional testing
   - Validate container contents without disk images

2. **Disk Image Creation**:
   - Use GitHub Actions workflow for official builds
   - Document user-side disk image creation process
   - Provide scripts to simplify local disk image creation (for users with loop device support)

3. **Testing Strategy**:
   - Test container image directly in VMs using `podman machine`
   - Use `bootc install to-existing-root` for in-place testing (if available)
   - Leverage CI/CD for full end-to-end testing

### For Production Workflows

**Recommended Approach**: Hybrid strategy

1. **CI/CD Pipeline** (GitHub Actions):
   - Build container images on every commit
   - Build disk images on release tags or manual dispatch
   - Publish both container images and disk artifacts

2. **User Options**:
   - **Option A**: Download pre-built disk images from releases
   - **Option B**: Pull container image and build disk locally
   - **Option C**: Use `bootc install` on existing systems

3. **Documentation**:
   - Clearly document both workflows
   - Provide helper scripts for local disk image creation
   - Include troubleshooting guide for common issues

### Implementation Priorities

1. **Immediate** (works now):
   - Continue using GitHub Actions for disk image builds
   - Focus on container image development in constrained environments
   - Document the separation of concerns

2. **Short-term** (enhance workflow):
   - Create helper scripts for users to build disk images locally
   - Add documentation for direct container image usage
   - Implement testing that doesn't require disk images

3. **Long-term** (if loop device support becomes available):
   - Evaluate podman-bootc for local development
   - Consider mkosi as alternative build system
   - Investigate userspace block device tools (FUSE-based approaches)

## Technical Comparison Matrix

| Approach | Loop Device Required | Works in gVisor | Complexity | Recommended |
|----------|---------------------|----------------|------------|-------------|
| bootc-image-builder | Yes | ❌ No | Low | ✅ Yes (via CI) |
| podman-bootc | Yes | ❌ No | Low | ❌ No |
| bootc install to-disk | Yes | ❌ No | Medium | ❌ No |
| bootc install to-filesystem | Yes | ❌ No | High | ❌ No |
| mkosi | Likely | ❌ Probably not | High | ⚠️ Maybe |
| Container-only delivery | No | ✅ Yes | Low | ✅ Yes (dev) |
| GitHub Actions CI | N/A | ✅ Yes | Low | ✅ Yes (prod) |
| Nested virtualization | No (but needs KVM) | ❌ No | Very High | ❌ No |
| NBD approach | Likely | ❌ Probably not | High | ❌ No |

## Conclusion

**For the current gVisor-based container environment:**

1. **Container image building**: ✅ Fully functional
2. **Disk image building**: ❌ Blocked by loop device limitation
3. **Recommended workflow**: Build containers locally, create disk images via CI/CD

**Key Insight**: The fundamental constraint is not a limitation of bootc or bootc-image-builder, but rather an architectural difference between gVisor's sandboxed kernel and native Linux kernel capabilities. Loop device support requires kernel module functionality that gVisor intentionally does not provide for security isolation.

**Practical Path Forward**: Embrace a hybrid workflow where container image development happens in any environment (including constrained ones), while disk image creation is delegated to environments with full kernel capabilities (GitHub Actions, local machines with native containers, or bare metal systems).

This approach maintains security isolation in constrained environments while still enabling the full bootc workflow where appropriate.

## Additional Resources

- [bootc-image-builder Documentation](https://osbuild.org/docs/bootc/)
- [bootc Installation Guide](https://bootc-dev.github.io/bootc/bootc-install.html)
- [mkosi Documentation](https://github.com/systemd/mkosi)
- [Fedora Bootable Containers](https://docs.fedoraproject.org/en-US/bootc/getting-started/)
- [Loop Device Kernel Documentation](https://www.kernel.org/doc/html/latest/block/loop.html)

## Appendix: Quick Reference Commands

### Test Loop Device Support
```bash
# Create test image
dd if=/dev/zero of=test.img bs=1M count=10

# Create loop device node (if needed)
mknod /dev/loop0 b 7 0

# Try to attach loop device
losetup /dev/loop0 test.img

# Expected in gVisor: "failed to set up loop device: No such device or address"
# Expected in native: Successfully attached
```

### Build Container Image Only
```bash
# Using Justfile
just build localhost/image-template latest

# Direct podman
podman build -t localhost/image-template:latest .
```

### Create Disk Image (requires loop device support)
```bash
# Using Justfile (on machine with loop support)
just build-qcow2 localhost/image-template latest

# Direct BIB invocation
sudo podman run --rm --privileged \
  --security-opt label=type:unconfined_t \
  -v $(pwd)/disk_config/disk.toml:/config.toml:ro \
  -v $(pwd)/output:/output \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  --type qcow2 \
  --rootfs btrfs \
  localhost/image-template:latest
```

### Test Container Image Directly
```bash
# Run interactive shell in container
podman run --rm -it localhost/image-template:latest /bin/bash

# Test specific functionality
podman run --rm localhost/image-template:latest systemctl list-units
```
