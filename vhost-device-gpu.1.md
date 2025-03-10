% VHOST-DEVICE-GPU(1) Version 0.1.0 | rust-vmm/vhost-device

NAME
===========

**vhost-device-gpu** — vhost-user backend for a VirtIO GPU device

SYNOPSIS
===========

```
vhost-device-gpu --socket-path <path> --gpu-mode <GPU_MODE>
```

DESCRIPTION
===========

A virtio-gpu device using the vhost-user protocol.

OPTIONS
===========

- **`-s, --socket-path <SOCKET>`**  
  vhost-user Unix domain socket.

- **`-g, --gpu-mode <GPU_MODE>`**  
  The mode specifies which backend implementation to use.

  Possible values:
  - `virglrenderer`: OpenGL implementation, superseded by Virgl2.
  - `gfxstream`: Vulkan implementation (partial support only).

- **`-c, --capset <CAPSET>`**  
  Comma-separated list of enabled capsets.

  Possible values:
  - `virgl`: OpenGL implementation, superseded by Virgl2.
  - `virgl2`: OpenGL implementation.
  - `gfxstream-vulkan`: Vulkan implementation (partial support only).  
    *NOTE: Can only be used for 2D display output for now, no hardware acceleration yet.*
  - `gfxstream-gles`: OpenGL ES implementation (partial support only).  
    *NOTE: Can only be used for 2D display output for now, no hardware acceleration yet.*

- **`--use-egl <USE_EGL>`**  
  Enable backend to use EGL.  
  *Default: `true`*  
  *Possible values: `true`, `false`*

- **`--use-glx <USE_GLX>`**  
  Enable backend to use GLX.  
  *Default: `false`*  
  *Possible values: `true`, `false`*

- **`--use-gles <USE_GLES>`**  
  Enable backend to use GLES.  
  *Default: `true`*  
  *Possible values: `true`, `false`*

- **`--use-surfaceless <USE_SURFACELESS>`**  
  Enable surfaceless backend option.  
  *Default: `true`*  
  *Possible values: `true`, `false`*

- **`-h, --help`**  
  Print help and usage information.

- **`-V, --version`**  
  Print version information.

LIMITATIONS
===========

- This device links native libraries (due to the usage of Rutabaga) compiled with GNU libc. The CI is set up to not build this device for musl targets.
- It might be possible to build those libraries using musl and then build the GPU device, but this is untested.
- Currently, only sharing the display output to QEMU through a socket using the `transfer_read` operation (`VIRTIO_GPU_CMD_TRANSFER_FROM_HOST_3D`) is supported.
- Directly sharing display output resources using dmabuf is not yet supported.
- The following features are not yet supported:
  - `VIRTIO_GPU_CMD_RESOURCE_CREATE_BLOB`
  - `VIRTIO_GPU_CMD_SET_SCANOUT_BLOB`
  - `VIRTIO_GPU_CMD_RESOURCE_ASSIGN_UUID`
  
  These require [rust-vmm/vhost#251](https://github.com/rust-vmm/vhost/pull/251), which in turn requires QEMU API stabilization.
- Because blob resources are not yet supported, some capsets are limited:
  - Venus (Vulkan implementation in virglrenderer) support is unavailable.
  - `gfxstream-vulkan` and `gfxstream-gles` are exposed but can only be used for display output (no hardware acceleration yet).

FEATURES
===========

The device leverages the [`rutabaga_gfx`](https://crates.io/crates/rutabaga_gfx) crate to provide rendering with virglrenderer and gfxstream.

- Gfxstream support is compiled by default but can be disabled by building without the `gfxstream` feature flag:

  ```
  cargo build --no-default-features
  ```

- With Virglrenderer, Rutabaga translates OpenGL and Vulkan calls to an intermediate representation, allowing OpenGL acceleration on the host.
- With the gfxstream rendering mode, GLES and Vulkan calls are forwarded to the host with minimal modification.

EXAMPLES
===========

Start the daemon on the host machine using either GPU mode:

1. **Virglrenderer**
2. **Gfxstream** (if the crate has been compiled with the `gfxstream` feature)

```
host# vhost-device-gpu --socket-path /tmp/gpu.socket --gpu-mode virglrenderer
```

With QEMU, two device frontends can be used:

1. **`vhost-user-gpu-pci`**
2. **`vhost-user-vga`** (also implements VGA, allowing visibility of boot messages before GPU initialization)

By default, QEMU adds another VGA output. Disable it using `-vga none`.

### Using `vhost-user-gpu-pci`

```
qemu-system-x86_64 \
  -chardev socket,id=vgpu,path=/tmp/gpu.socket \
  -device vhost-user-gpu-pci,chardev=vgpu,id=vgpu \
  -object memory-backend-memfd,share=on,id=mem0,size=4G \
  -machine q35,memory-backend=mem0,accel=kvm \
  -display gtk,gl=on,show-cursor=on \
  -vga none
```

### Using `vhost-user-vga`

```
qemu-system-x86_64 \
  -chardev socket,id=vgpu,path=/tmp/gpu.socket \
  -device vhost-user-vga,chardev=vgpu,id=vgpu \
  -object memory-backend-memfd,share=on,id=mem0,size=4G \
  -machine q35,memory-backend=mem0,accel=kvm \
  -display gtk,gl=on,show-cursor=on \
  -vga none
```

ENVIRONMENT
===========

**RUST_LOG**

:   Logging level. Set to `debug` for maximum output.

BUGS
====

See GitHub Issues: <https://github.com/rust-vmm/vhost-device/issues>

AUTHORS
======

Dorinda Bassey <dbassey@redhat.com>

Matej Hrica <mhrica@redhat.com>

SEE ALSO
========

**qemu(1)**
