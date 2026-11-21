# CLM

Also called: Early Adam Update, CLM Pipelining, CPU-GPU Memory Barrier Removal, Smart Caching Policy for Gaussian States

## Overview
CLM is a training optimization for 3D Gaussian Splatting that removes the GPU memory barrier by offloading Gaussian states to CPU memory and overlapping CPU-side optimizer updates with GPU rendering [S2]. It solves the problem of VRAM exhaustion and communication stalls in large-scale scenes by implementing a smart caching policy that keeps frequently updated Gaussians on the GPU while streaming others [S1, unofficial][S2]. Unlike baseline offloading which incurs significant latency, CLM uses pipelining and early Adam updates to hide communication overhead, enabling training of scenes that exceed GPU memory on a single device [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A batch of training views (microbatches), the current set of 3D Gaussian parameters $\theta$ (position, covariance, opacity, SH coefficients), and the Adam optimizer state.
*   **Output:** Updated Gaussian parameters $\theta'$ and rendered images.

**Data Structures:**
1.  **CPU Memory Buffer:** Stores the full set of Gaussian attributes. Attributes are concatenated and padded to be contiguous and cache-line aligned [S2].
2.  **GPU Memory Buffer:** Stores a subset of "in-frustum" Gaussians required for the current microbatch [S2].
3.  **Double Buffer:** Used in the pipelining stage to enable parameter prefetching and delayed gradient offloading [S2].
4.  **Signal Buffer:** A buffer in CPU pinned memory used to synchronize the GPU communication stream and the CPU Adam thread [S2].

**Algorithm Steps:**

1.  **Pre-compute Finalization Microbatch ($L_g$):**
    For each Gaussian $g$ in the scene, determine the microbatch index $L_g$ at which it is "finalized." A Gaussian is finalized in microbatch $j$ if it is accessed in microbatch $j$ but not in any subsequent microbatch $k > j$ within the current training batch $B$.
    $$ L_g = \max \{ i \mid g \in S_i \} $$
    where $S_i$ is the set of Gaussians accessed in microbatch $i$. If $g$ is not accessed in the batch, $L_g = 0$ [S2].

2.  **Microbatch Scheduling:**
    Schedule the order of microbatches to maximize two factors:
    *   **Spatial Locality:** Maximize the intersection $|S_{i+1} \cap S_i|$ to improve GPU caching efficiency [S2].
    *   **Early Finalization:** Arrange microbatches so that more Gaussians are finalized in earlier indices, allowing more CPU Adam updates to overlap with GPU work [S2].

3.  **Selective Loading (GPU-initiated):**
    Instead of using standard `cudaMemcpy`, a custom CUDA kernel initiates data transfer from pinned CPU memory to GPU registers, then to GPU memory [S2].
    *   The kernel splits the contiguous CPU buffer and concatenates only the in-frustum Gaussian attributes into GPU memory [S2].
    *   This avoids CPU-GPU coordination overhead and utilizes PCIe bandwidth efficiently [S2].

4.  **Forward and Backward Pass:**
    The GPU renders the current microbatch using the loaded Gaussians and computes gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ [S2].

5.  **Gradient Offloading:**
    A custom kernel transfers gradients from GPU memory to CPU memory. The kernel fetches previously accumulated gradients, adds the new gradients in CUDA registers, and stores the sum back to CPU memory [S2].

6.  **Early CPU Adam Update:**
    A dedicated CPU thread executes the Adam optimizer [S2].
    *   At the end of every microbatch $j$, the CPU thread updates all Gaussians $g$ where $L_g = j$ [S2].
    *   The Python Global Interpreter Lock (GIL) is released during this process to allow concurrent execution with the primary thread [S2].
    *   Synchronization is handled via the signal buffer: the GPU sets a signal via DMA when gradients are ready, triggering the CPU update [S2].
    *   Gaussians finalized in the last microbatch ($L_g = B$) do not have their Adam computation overlapped with subsequent GPU work [S2].

7.  **Caching Policy:**
    The system maintains a cache of frequently updated Gaussians on the GPU. Gaussians not in the cache are streamed from CPU memory as needed [S1, unofficial][S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_batch(batch_views, gaussians, optimizer_state):
    # 1. Determine finalization microbatch for each Gaussian
    # S_i is the set of Gaussians accessed in microbatch i
    L_g = {}
    for g in gaussians:
        accessed_microbatches = [i for i, view in enumerate(batch_views) if g in view.frustum_gaussians]
        if accessed_microbatches:
            L_g[g] = max(accessed_microbatches)
        else:
            L_g[g] = 0

    # 2. Schedule microbatches to maximize overlap (implementation detail omitted)
    scheduled_views = schedule_microbatches(batch_views)

    # 3. Initialize double buffers for pipelining
    gpu_params_buffer = allocate_gpu_buffer()
    cpu_grad_buffer = allocate_cpu_pinned_buffer()
    
    # 4. Process microbatches
    for j, view in enumerate(scheduled_views):
        # Selective Loading: Load in-frustum Gaussians from CPU to GPU
        # Initiated by GPU kernel to avoid CPU-GPU coordination
        in_frustum_gaussians = get_in_frustum_gaussians(view, L_g)
        selective_load_kernel(in_frustum_gaussians, gpu_params_buffer)

        # Forward and Backward Pass
        render_output = forward_pass(gpu_params_buffer, view)
        gradients = backward_pass(render_output, view)

        # Gradient Offloading: Accumulate gradients in CPU memory
        # Kernel adds new gradients to existing accumulated gradients
        accumulate_gradients_kernel(gradients, cpu_grad_buffer)

        # Signal GPU communication stream completion
        signal_buffer.set_signal()

        # Early CPU Adam Update
        # Update Gaussians finalized in this microbatch
        gaussians_to_update = [g for g, l in L_g.items() if l == j]
        if gaussians_to_update:
            cpu_adam_thread.update(gaussians_to_update, cpu_grad_buffer, optimizer_state)

    return gaussians
```

## Subtopics

### Out-of-Core Training and Rendering
A framework that stores the full scene in CPU RAM and dynamically streams only visible Gaussians to GPU memory, enabling training and rendering of scenes exceeding VRAM on a single consumer GPU [S3, unofficial]. It uses a lightweight caching system to track recently used Gaussians and reduce CPU-GPU data transfer [S3, unofficial]. A pitfall is that a single distant view may require access to the full scene, causing high transfer overhead [S3, unofficial].

### Selective Loading Kernel
A custom CUDA kernel that loads Gaussian parameters from pinned CPU memory to GPU registers and then to GPU memory, initiated from the GPU to avoid CPU-GPU coordination overhead [S2]. It stores offloaded Gaussian attributes in pinned CPU memory, concatenated and padded for cache-line alignment [S2]. The kernel splits attributes from the contiguous CPU buffer and concatenates only in-frustum attributes into GPU memory [S2]. A similar kernel is used to transfer gradients from GPU to CPU [S2].

## Operating characteristics
*   **Memory Usage:** Significantly reduces GPU memory usage for Gaussian model states by offloading them to CPU RAM [S2]. It slightly increases "other" memory usage due to the double buffer design for pipelining [S2].
*   **Performance:** Reduces training time by overlapping communication and CPU Adam computation with GPU computation [S2]. In naive offloading, communication and CPU Adam overheads account for >50% of training time; CLM reduces this by hiding these costs [S2].
*   **Scalability:** Enables training on scenes that exceed VRAM on a single GPU [S2][S3, unofficial].
*   **Hardware:** Requires pinned CPU memory and a dedicated CPU thread for Adam updates [S2]. Tested on RTX 2080 Ti and RTX 4090 [S2].
*   **Limitations:** The effectiveness of overlapping CPU Adam depends on the scene; scenes with fewer early-finalized Gaussians see less benefit [S2].

## Background
CLM was introduced in the paper "CLM: Removing the GPU Memory Barrier for 3D Gaussian Splatting" [S2]. It extends the Grendel training framework [S2] and incorporates gsplat rasterization kernels [S2]. The method addresses the limitations of naive offloading and distributed training approaches like Grendel-GS [S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] CLM: Removing the GPU Memory Barrier for 3D Gaussian Splatting, https://doi.org/10.1145/3779212.3790140, fetched 2026-10-31
[S3] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
