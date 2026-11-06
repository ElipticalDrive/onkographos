# Inference Fast Path for Volume Rendering
Also called: NeedsBackward specialization, Stage 2 optimization

## Overview
The Inference Fast Path is a compile-time specialization of the volume rendering kernel that eliminates backward-only output storage and computation when gradients are not required. It differs from the register accumulation stage by removing the largest remaining source of global-memory traffic: the per-sample `outWs` store and the host-side zero-initialization of the $N$-sized buffer [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume data, ray origins, ray directions, and a compile-time boolean template parameter `NeedsBackward`.
*   **Outputs:**
    *   `outRgb`: Accumulated color (always allocated).
    *   `outWs`: Per-sample weights (allocated only if `NeedsBackward` is true).
    *   `outDepth`: Per-ray depth (allocated only if `NeedsBackward` is true).
    *   `outTotalSamples`: Per-ray total sample count (allocated only if `NeedsBackward` is true).

**Data Structures:**
*   `outRgb`: Tensor of shape `[num_rays, channels]`.
*   `outWs`: Tensor of shape `[num_rays, max_samples]` (or similar $N$).
*   `outDepth`: Tensor of shape `[num_rays]`.
*   `outTotalSamples`: Tensor of shape `[num_rays]`.
*   When `NeedsBackward` is false, `outWs`, `outDepth`, and `outTotalSamples` are size-0 placeholder tensors with matching dtype and device.

**Algorithm:**
1.  **Dispatch:** The host-side dispatcher uses `std::true_type` / `std::false_type` tag-dispatch inside `AT_DISPATCH_V2` to select the kernel specialization based on the `NeedsBackward` template parameter [S1, unofficial].
2.  **Kernel Execution (CUDA/CPU):**
    *   If `NeedsBackward` is **true**:
        *   Allocate `outWs` using `torch::zeros` because the backward pass's `thrust::inclusive_scan` reads into the early-termination tail [S1, unofficial].
        *   Allocate `outDepth` and `outTotalSamples`.
        *   The kernel computes and stores per-sample weights to `outWs` and per-ray depth/total samples to `outDepth`/`outTotalSamples` [S1, unofficial].
    *   If `NeedsBackward` is **false**:
        *   Skip the per-sample store to `outWs`.
        *   Skip the per-ray stores to `outDepth` and `outTotalSamples`.
        *   Return size-0 placeholder tensors for these outputs [S1, unofficial].
3.  **Host-Side Allocation:**
    *   If `NeedsBackward` is false, allocate size-0 placeholder tensors for `outWs`, `outDepth`, and `outTotalSamples` with matching dtype and device [S1, unofficial].
    *   If `NeedsBackward` is true, allocate the full buffers as described above [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Host-side dispatcher logic
def volume_render_fwd_callback(volume_data, ray_origins, ray_directions, needs_backward):
    # Tag dispatch based on needs_backward
    if needs_backward:
        # Allocate full output tensors
        out_rgb = torch.empty(...)
        out_ws = torch.zeros(...) # Required for backward's inclusive_scan
        out_depth = torch.empty(...)
        out_total_samples = torch.empty(...)
        
        # Launch kernel with NeedsBackward=True
        launch_kernel(volume_data, ray_origins, ray_directions, 
                      out_rgb, out_ws, out_depth, out_total_samples, 
                      needs_backward=True)
    else:
        # Allocate color tensor
        out_rgb = torch.empty(...)
        
        # Create size-0 placeholders for backward-only outputs
        out_ws = torch.empty(0, dtype=..., device=...)
        out_depth = torch.empty(0, dtype=..., device=...)
        out_total_samples = torch.empty(0, dtype=..., device=...)
        
        # Launch kernel with NeedsBackward=False
        launch_kernel(volume_data, ray_origins, ray_directions, 
                      out_rgb, out_ws, out_depth, out_total_samples, 
                      needs_backward=False)
    
    return out_rgb, out_ws, out_depth, out_total_samples

# Kernel specialization (conceptual)
@cuda_kernel
def volume_render_kernel(out_rgb, out_ws, out_depth, out_total_samples, needs_backward):
    # ... ray marching logic ...
    
    # Accumulate color (always done)
    out_rgb[ray_id] += contribution
    
    if needs_backward:
        # Store per-sample weight for backward pass
        out_ws[ray_id, sample_idx] = weight
        
        # Store per-ray depth and total samples
        out_depth[ray_id] = depth
        out_total_samples[ray_id] = total_samples
    else:
        # Skip storing out_ws, out_depth, out_total_samples
        pass
```

**Parameters:**
*   `NeedsBackward`: Compile-time template parameter (bool). When `false`, the inference fast path is enabled [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Requirement:** The Python wrapper must route through the CUDA forward with `needsBackward=true` whenever any input requires grad, ensuring saved tensors and backward kernels function correctly [S1, unofficial].
*   **Placeholder Tensors:** Size-0 tensors must have matching dtype and device to avoid runtime errors in subsequent operations that might expect non-empty tensors [S1, unofficial].

## Operating characteristics
*   **Performance:** Further reduced `volume_render` stage time from 6171.4 ms to 2127.0 ms (~2.9x speedup) and end-to-end walltime from 11887.0 ms to 7862.8 ms (~1.86x total speedup vs baseline) in a fire renderer at 256 spp [S1, unofficial].
*   **Memory:** Eliminates the allocation and zero-initialization of the $N$-sized `outWs` buffer and the per-sample stores to `outDepth` and `outTotalSamples` during inference [S1, unofficial].
*   **Strengths:** Significant reduction in global-memory traffic and kernel execution time for inference-only workloads.
*   **Limitations:** Only applicable when gradients are not required. Requires careful handling of placeholder tensors in the host code.

## Background
This optimization is part of a series of improvements to the `volume_render` stage in the FVDB library, specifically PR #640 [S1, unofficial]. It follows the "Register accumulation" stage (Stage 1) and precedes other optimizations like the "Add Vec2 and double fast paths to SampleGridTrilinear" [S1, unofficial]. The autograd layer was moved to Python (`fvdb/_volume_render.py`) to facilitate this specialization [S1, unofficial].

## Sources
[S1] Optimizations for `volume_render` and move its autograd layer to Python, https://github.com/openvdb/fvdb-core/pull/640, fetched 2026-10-27 (unofficial)
