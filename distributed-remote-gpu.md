# Distributed Remote GPU via DRM Proxy

## Executive Summary

A Linux machine emulates a local GPU device (`/dev/dri/renderD*`) while transparently forwarding GPU workloads over the network to a remote execution node—initially targeting Apple Silicon running Asahi Linux. GPU access becomes a networked resource, like NFS for storage or SSH for compute.

---

## Problem

Modern GPU access is tightly coupled to hardware locality:

- PCIe attachment required
- Vendor-specific stacks (CUDA, ROCm) limit portability
- No standardized remote execution protocol at the DRM layer
- Heterogeneous hardware (Apple Silicon, ARM GPUs, integrated GPUs) sits underutilized on LANs

**Target user:** A developer with a GPU-less x86 laptop and an M-series Mac (or Asahi Linux machine) on the same network who wants to run Vulkan compute or ML inference without a cloud subscription.

---

## Prior Art & Differentiation

| Project | Approach | Gap |
|---|---|---|
| VirtIO-GPU | Paravirtualized GPU for VMs | Requires hypervisor; not network-transparent |
| SPICE/QXL | VM display remoting | Display only, not compute |
| Parsec / Moonlight | Game streaming | Screen-level, not API-level |
| GPUDirect RDMA | HPC GPU-to-GPU | Requires RDMA fabric, same vendor |
| **This project** | DRM-level API remoting over LAN | Vendor-neutral, no hypervisor, real API surface |

The key insight: intercept at the **Vulkan ICD or libdrm ioctl layer**, before GPU-specific encoding, so the protocol is GPU-agnostic on the wire.

---

## Architecture

```
[Client Machine]
  Application (Vulkan / OpenGL / compute)
       │
  ┌────▼─────────────────────────────┐
  │  GPU Proxy (libgpu-remote.so)    │  ← LD_PRELOAD or Vulkan ICD
  │  - Intercepts Vulkan calls       │
  │  - Serializes commands + buffers │
  └────────────┬─────────────────────┘
               │  TCP / Unix socket (local → remote)
               │  Protocol: Cap'n Proto or FlatBuffers (zero-copy)
  ┌────────────▼─────────────────────┐
  │  GPU Remote Server (gpud)        │  ← runs on Asahi node
  │  - Deserializes commands         │
  │  - Calls local Vulkan/Mesa       │
  │  - Returns results + fences      │
  └────────────┬─────────────────────┘
               │
  [Apple Silicon / Asahi Linux]
  Mesa + Vulkan (Honeykrisp driver)
  Physical GPU (AGX)
```

### Component Breakdown

#### 1. Client Proxy (`libgpu-remote.so`)

Two implementation strategies (start with A, graduate to B):

**Option A — Vulkan ICD shim (MVP)**
- Implement a Vulkan Installable Client Driver that registers itself via `VK_ICD_FILENAMES`
- All Vulkan calls go through the shim before any actual GPU call
- Serialize with Cap'n Proto; send over TCP or a Unix socket tunneled via SSH

**Option B — DRM kernel module (long-term)**
- A `misc` or `platform` driver that exposes `/dev/dri/renderD128`
- Handles ioctl from Mesa/libdrm directly
- Forwards binary ioctl payload to remote server
- Full transparency: unmodified applications work without LD_PRELOAD

#### 2. Wire Protocol

Requirements: low latency, minimal copies, out-of-order completion.

Recommended: **Cap'n Proto** (zero-copy, schema-defined, good C++ and Rust support)

Message types:
```
VkCommand       { cmd_type, serialized_params }
BufferAlloc     { size, usage_flags } → handle
BufferWrite     { handle, offset, data[] }
BufferRead      { handle, offset, length } → data[]
FenceWait       { fence_handle, timeout_ns } → status
SwapBuffers     { surface_handle } → image_data (for display)
```

Transport options:
- **Same LAN:** raw TCP, ~50–100µs RTT → acceptable for batched compute
- **Compression:** zstd on buffer transfers (textures, vertex data compress well)
- **RDMA (future):** if both nodes have InfiniBand or RoCE

#### 3. Remote Server (`gpud`)

A daemon written in Rust or C++ that:
- Listens on a Unix socket (local) or TCP (remote)
- Maintains a per-client Vulkan device + command pool
- Executes incoming commands against the real Mesa Vulkan driver
- Returns results and fence completion events

Key challenges:
- **Buffer ownership:** all `VkDeviceMemory` lives on the server; client holds opaque handles
- **Synchronization:** fences and semaphores must map to server-side primitives with round-trip signaling
- **Multi-tenancy:** multiple clients sharing one GPU need Vulkan queue isolation

---

## Memory Model

No shared memory between client and server, so:

```
Client action          Server action
─────────────────────────────────────
vkAllocateMemory()  →  allocate on server, return handle
vkMapMemory()       →  server maps; client requests range via BufferRead
vkUnmapMemory()     →  implicit; dirty ranges uploaded via BufferWrite
vkFreeMemory()      →  server deallocates
```

**Optimization strategies:**
- **Lazy upload:** only transfer dirty pages on explicit flush or fence wait
- **Staging buffer pool:** pre-allocated transfer buffers reused across frames
- **Delta compression:** for repeated buffer updates (animation, streaming data)

---

## Synchronization

| Vulkan primitive | Remote mapping |
|---|---|
| `VkFence` | Server-side fence; client blocks on RPC call |
| `VkSemaphore` | Server-side semaphore; referenced by handle |
| `vkQueueWaitIdle` | Blocking RPC; server drains queue |
| Timeline semaphores | Full support; map directly to server side |

For async pipelines: client can submit batches with a `signal_fence` field and poll the server asynchronously, enabling pipelining without a full round-trip per frame.

---

## Implementation Roadmap

### Phase 1 — RPC Proof of Concept (2–3 weeks)
- Implement `gpud` server with a single Vulkan compute queue
- Write a minimal Cap'n Proto schema covering `vkCreateBuffer`, `vkCreateShaderModule`, `vkCmdDispatch`, `vkQueueSubmit`, `vkGetFenceStatus`
- Run a simple compute shader (matrix multiply) end-to-end
- Measure round-trip latency baseline

### Phase 2 — Vulkan ICD Shim (3–4 weeks)
- Implement a valid Vulkan ICD (follow Vulkan loader spec)
- Handle the full instance/device/queue creation flow
- Forward descriptor set and pipeline operations
- Test with a real application: `vkmark`, `vulkan-samples`, or a PyTorch backend

### Phase 3 — Memory & Sync Hardening (2–3 weeks)
- Implement staging buffer pool
- Add lazy dirty-range tracking for mapped memory
- Handle timeline semaphores
- Add reconnect logic and error propagation

### Phase 4 — Compute Workload Compatibility (ongoing)
- Target ML inference via Vulkan compute (llama.cpp Vulkan backend)
- Test stable diffusion via `stable-diffusion.cpp`
- Fix compatibility issues as they arise

### Phase 5 — DRM Kernel Module (optional, 4–6 weeks)
- Write a `drm/misc` virtual GPU driver
- Implement GEM buffer objects backed by remote handles
- Expose render node; test with unmodified Mesa

---

## Performance Targets

| Workload | Expected overhead vs local |
|---|---|
| LLM inference (batch) | 5–20% (network + serialization) |
| Vulkan compute (non-interactive) | 10–30% |
| Real-time graphics (60fps) | Not a target; latency too high |

Network requirement: **1GbE LAN minimum**, 10GbE recommended for large buffer transfers. WiFi 6 is borderline for compute; not recommended for graphics.

---

## Security

- **Authentication:** mutual TLS with client certificates; `gpud` refuses unauthenticated connections
- **Encryption:** TLS 1.3 for all traffic; optional bypass for localhost/trusted LAN
- **Command validation:** server validates all handles before use; rejects out-of-bounds accesses
- **Resource limits:** per-client VRAM cap, command queue depth limit, timeout on idle connections

---

## Open Questions

1. **Shader portability:** client may have shaders compiled for x86 SPIR-V quirks; server runs AGX. SPIR-V is GPU-agnostic so this should be fine—but driver-specific extensions may differ.
2. **WSI (windowed rendering):** swapchain images need to come back to the client for display. This requires image readback, which is expensive. Compute-only avoids this entirely.
3. **Vulkan validation layers:** run on server side for debugging; disable on client side to minimize overhead.
4. **Multi-GPU aggregation:** out of scope for v1, but the handle-based protocol naturally extends to routing across multiple backends.

---

## Future Work

- Load-balance across multiple remote GPU nodes
- gRPC streaming for lower-latency pipelines
- WebGPU frontend (browser → remote GPU via this system)
- Integration with ML frameworks (PyTorch, JAX custom device backend)
- Compression pipeline for sparse/structured data (model weights, activations)
