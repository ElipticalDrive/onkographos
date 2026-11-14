# Pinned Memory Data Transfer

Also called: cudaMallocHost, Page-locked Host Memory Transfer, Pinned Memory Optimization

## Overview
Pinned Memory Data Transfer utilizes page-locked (pinned) host memory to enable direct, high-throughput Direct Memory Access (DMA) transfers between the CPU and GPU, bypassing the system's physical page table mechanisms [S1, S2, unofficial]. Unlike standard pageable host memory, which requires the GPU driver to stage data through a temporary pinned buffer, pinned memory allows the PCIe bus to access host memory addresses directly [S1, unofficial]. This method is distinct from CUDA Unified Memory (introduced in CUDA 7.x), which uses a demand-paged, transparent memory management system, whereas pinned memory requires explicit allocation and management [S2, unofficial].

## How it works
**Inputs:** Host-side data buffer (e.g., mesh vertices, volume textures); transfer size in bytes.
**Outputs:** Device-side buffer populated with data; host-side pinned memory handle.

**Data Structures:**
1.  **Host Pinned Buffer:** Allocated via `cudaHostAlloc` or `cudaMallocHost`. Layout is contiguous in virtual memory.
2.  **Device Buffer:** Allocated via `cudaMalloc`. Layout is contiguous in device global memory.
3.  **Transfer Descriptor:** Not strictly required if using `cudaMemcpyDefault`, but logically represents the source pointer, destination pointer, and byte count.

**Steps:**
1.  **Allocate Pinned Host Memory:** Request page-locked memory from the CUDA runtime. This memory is not swappable by the OS [S1, S2, unofficial].
2.  **Populate Host Data:** Copy or generate the source data (e.g., triangle vertices) into the pinned host buffer [S2, unofficial].
3.  **Allocate Device Memory:** Reserve space in GPU global memory [S2, unofficial].
4.  **Execute Transfer:** Initiate the DMA transfer from the pinned host buffer to the device buffer [S2, unofficial].

**Formulas:**
*   Transfer Overhead Reduction: $T_{pinned} < T_{pageable}$, where $T$ is the total transfer time including staging.
*   Memory Constraint: The amount of pinned memory is limited by physical RAM and system stability, as it reduces available pageable memory [G].

**Parameters:**
*   `cudaHostAllocDefault`: Standard flag for pinned memory.
*   `cudaMemcpyDefault`: Auto-detects transfer direction based on pointer types (host vs. device) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Exhaustion:** Excessive use of pinned memory can starve the host OS of pageable memory, leading to system instability or thrashing [G].
*   **Alignment:** Data should be aligned to cache line boundaries (typically 64 bytes) for optimal DMA performance [G].
*   **Legacy vs. Modern:** In CUDA < 7.x, explicit pinned memory was the primary method for fast transfers. In CUDA 7.x+, Unified Memory is often preferred for simplicity, though pinned memory remains faster for bulk, predictable transfers [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Allocate pinned host memory
# h_data is a pointer to host memory
# size_bytes is the total size in bytes
h_data = cuda_malloc_host(size_bytes)

# Step 2: Populate data (example: copying from a standard host buffer)
# source_buffer is a standard pageable host array
memcpy(h_data, source_buffer, size_bytes)

# Step 3: Allocate device memory
d_data = cuda_malloc(size_bytes)

# Step 4: Transfer data to device
# cudaMemcpyDefault automatically detects h_data is host and d_data is device
cuda_memcpy(dst=d_data, src=h_data, count=size_bytes, kind=cudaMemcpyDefault)

# Cleanup
cuda_free_host(h_data)
cuda_free(d_data)
```

## Subtopics

### CUDA Unified Memory
A higher-level abstraction that manages paging between host and device automatically. It simplifies coding but may incur overhead from page faults compared to explicit pinned memory transfers [S2, unofficial].

### Zero-Copy Memory
Allows the GPU to access host memory directly without copying. It is useful for small, irregular accesses but suffers from lower bandwidth than pinned memory transfers due to PCIe latency [G].

## Operating characteristics
*   **Throughput:** Significantly higher than pageable memory transfers, often approaching the theoretical PCIe bandwidth limit for large blocks [S1, unofficial].
*   **Latency:** Setup latency is low, but the actual transfer is limited by PCIe bandwidth.
*   **Memory Cost:** Consumes physical RAM that cannot be paged to disk. Over-allocation can degrade system performance [G].
*   **Use Case:** Ideal for large, frequent data transfers (e.g., volume textures, mesh data) where bandwidth is critical [S1, S2, unofficial]. Not recommended for small, sporadic updates where the overhead of management outweighs the benefit.

## Background
The concept of pinned memory in CUDA dates back to the early versions of the CUDA toolkit, prior to the introduction of Unified Memory in CUDA 7.0 (2015) [S2, unofficial]. It was the standard method for optimizing host-to-device data movement in high-performance computing (HPC) and graphics applications [S1, unofficial]. The technique is widely used in voxelization pipelines and volume rendering frameworks where large datasets (e.g., triangle meshes) must be uploaded to the GPU for processing [S2, unofficial].

## Sources
[S1] wyh1309/hpc_foundational_demos, https://github.com/wyh1309/hpc_foundational_demos, fetched 2026-10-26 (unofficial)
[S2] GitHub - Forceflow/cuda_voxelizer: CUDA Voxelizer to convert polygon ..., https://github.com/Forceflow/cuda_voxelizer, fetched 2026-10-27 (unofficial)
