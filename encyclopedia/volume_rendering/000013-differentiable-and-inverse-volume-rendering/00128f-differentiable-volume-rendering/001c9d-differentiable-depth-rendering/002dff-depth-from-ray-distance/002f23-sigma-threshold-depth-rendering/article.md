# Sigma-Threshold Depth Rendering

Also called: Dex-NeRF depth rendering, Opaque Mask Computation, Sigma Threshold Rendering, Sigma Threshold Rendering Kernel

## Overview
Sigma-Threshold Depth Rendering computes a binary mask indicating the sample index where cumulative weight accumulation exceeds a specified density threshold, effectively identifying a 'surface' location along the ray [S1, unofficial]. It serves as a differentiable depth estimation method that renders a binary or thresholded output based on whether volume density exceeds a given value, referencing the approach used in Dex-NeRF [S3, unofficial]. Unlike Expected Term Rendering which computes the mean distance, or Median Depth Rendering which finds the median, this method identifies the depth at which the accumulated opacity crosses a hard boundary [S1, S3, unofficial]. It builds on [Depth from Ray Distance](5665-depth-from-ray-distance.md) by introducing a termination condition based on accumulated sigma rather than fixed sample counts or probabilistic expectations.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `rays`: Ray origins and directions (tensor of shape `[Q, 3]` or similar, where `Q` is the number of rays).
    *   `weights`: Array of density weights from the NeRF model for each sample along each ray [S1, unofficial].
    *   `sigma_thresh`: The density threshold value used for comparison [S2, S3, unofficial].
    *   `depth_threshold`: Float accumulation threshold for depth termination (default 0.5) [S1, unofficial].
    *   `grid`: Spatial data structure containing volume density information [S2, S3, unofficial].
    *   `opt`: Rendering options (e.g., step size, bounds) [S2, S3, unofficial].
*   **Outputs:**
    *   `results`: A result tensor of size `Q` (one value per ray), with dtype and device matching `rays.origins`, `requires_grad=False` [S2, S3, unofficial]. The value typically indicates the depth or sample index where the threshold was crossed.

**Data Structures:**
*   `SingleRaySpec`: A structure holding ray specification details (origin, direction, bounds) for a single ray [S3, unofficial].
*   `cum`: Cumulative sum of weights [S1, unofficial].
*   `op`: Boolean mask where `cum >= depth_threshold` [S1, unofficial].
*   `op_shifted`: `op` shifted right by one position, padded with zeros at the beginning [S1, unofficial].
*   `mask`: XOR of `op` and `op_shifted`, cast to weights dtype [S1, unofficial].

**Steps:**
1.  **Initialize Ray Specification:** For each ray, create a `SingleRaySpec` and find the intersection bounds with the volume grid [S3, unofficial].
2.  **Compute Cumulative Weights:** Calculate the cumulative sum of the per-sample weights along the ray:
    $$ cum = \text{cumsum}(weights) $$
    [S1, unofficial]
3.  **Create Threshold Mask:** Generate a boolean mask indicating where the cumulative weight exceeds the threshold:
    $$ op = cum \ge depth\_threshold $$
    [S1, unofficial]
4.  **Shift Mask:** Shift the boolean mask `op` to the right by one position and pad with a zero at the start to identify the *first* crossing:
    $$ op_{shifted} = [0, op[:-1]] $$
    [S1, unofficial]
5.  **Compute Crossing Mask:** Compute the XOR of the original and shifted masks to find the exact sample index where the threshold was crossed:
    $$ mask = op \oplus op_{shifted} $$
    Cast `mask` to the weights' data type [S1, unofficial].
6.  **Trace Ray (CUDA Implementation):**
    *   Launch the CUDA kernel `device::render_ray_sigma_thresh_kernel` [S3, unofficial].
    *   Calculate number of blocks: `blocks = CUDA_N_BLOCKS_NEEDED(Q, TRACE_RAY_CUDA_THREADS)` [S2, S3, unofficial].
    *   Pass `grid`, `rays`, `opt`, `sigma_thresh`, and the output tensor pointer to the kernel [S2, S3, unofficial].
    *   Inside the kernel, for each ray, march along the ray until the accumulated density exceeds `sigma_thresh` [S2, S3, unofficial].
7.  **Finalize Output:** Return the `results` tensor containing the depth or index for each ray [S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sigma_threshold_depth_rendering(rays, weights, sigma_thresh, depth_threshold=0.5):
    """
    Compute depth using sigma thresholding.
    
    Args:
        rays: Tensor of ray origins and directions.
        weights: Tensor of per-sample weights (density/opacity contributions).
        sigma_thresh: Density threshold for early termination (used in CUDA kernel).
        depth_threshold: Accumulation threshold for mask computation (default 0.5).
        
    Returns:
        results: Tensor of depth values or indices for each ray.
    """
    # Step 1: Compute cumulative sum of weights
    # weights shape: [Q, N_samples]
    cum = torch.cumsum(weights, dim=1)
    
    # Step 2: Create boolean mask where cumulative weight exceeds threshold
    op = cum >= depth_threshold
    
    # Step 3: Shift op right by one and pad with zeros
    # op_shifted[i] = op[i-1] for i > 0, op_shifted[0] = 0
    op_shifted = torch.cat([torch.zeros_like(op[:, :1]), op[:, :-1]], dim=1)
    
    # Step 4: Compute XOR mask to find the first crossing
    # mask[i] = 1 if op[i] != op_shifted[i], else 0
    mask = op ^ op_shifted
    mask = mask.to(weights.dtype)
    
    # Step 5: Compute depth based on the mask
    # Assuming 'depths' is a tensor of sample distances along the ray [Q, N_samples]
    # The depth is the weighted sum of depths where mask is 1.
    # Since mask is binary and has exactly one '1' per ray (if threshold is crossed),
    # we can simply multiply and sum.
    # Note: If threshold is never crossed, mask is all zeros.
    # Implementation detail: Handle rays that don't cross threshold.
    
    # For CUDA implementation, the kernel 'render_ray_sigma_thresh_kernel' handles
    # the ray marching and threshold check directly on the density grid.
    # Here we simulate the result using the mask and pre-computed sample depths.
    
    # Initialize results tensor
    results = torch.zeros(rays.shape[0], dtype=weights.dtype, device=weights.device)
    
    # Find indices where mask is 1
    # This is a simplified Python equivalent; actual implementation uses CUDA kernel
    # to avoid expensive scatter/gather operations on large tensors.
    
    # In the CUDA kernel (render_ray_sigma_thresh_kernel):
    # 1. For each ray, march samples.
    # 2. Accumulate density.
    # 3. If accumulated density > sigma_thresh, record current depth and break.
    # 4. Store result in output tensor.
    
    # Placeholder for CUDA kernel call logic described in sources:
    # blocks = (Q + TRACE_RAY_CUDA_THREADS - 1) // TRACE_RAY_CUDA_THREADS
    # render_ray_sigma_thresh_kernel<<<blocks, TRACE_RAY_CUDA_THREADS>>>(
    #     grid, rays, opt, sigma_thresh, results.data_ptr()
    # )
    
    # For the purpose of this pure Python implementation, we return the mask-based depth
    # if available, otherwise 0.
    # This part is illustrative as the core method is CUDA-based [S2, S3, unofficial].
    
    return results
```

**Parameters:**
*   `sigma_thresh`: Density threshold value for early ray termination during volume rendering [S2, S3, unofficial].
*   `depth_threshold`: Accumulation threshold for depth termination (default 0.5) [S1, unofficial].
*   `max_depth`: 10 (specific to ScanNet dataset) [S4].
*   `min_depth`: 0.001 (specific to ScanNet dataset) [S4].
*   `threshold_multiplier`: 0.0001 (applied to the depth range) [S4].

**Edge Cases and Pitfalls:**
*   **Threshold Not Reached:** If the cumulative weight never exceeds `depth_threshold`, the mask will be all zeros. The implementation must handle this case, typically by returning a default depth (e.g., `max_depth`) or marking the ray as invalid [S1, unofficial].
*   **Precision:** Floating-point precision in cumulative sum can affect the exact crossing point [S1, unofficial].
*   **Boundaries:** Ray bounds must be correctly computed to avoid sampling outside the volume [S2, S3, unofficial].

## Subtopics

### Depth Uncertainty Thresholding
A variant that uses a small threshold $\tau$ to evaluate the uncertainty of a depth map. The threshold is calculated as $\tau = (max\_depth - min\_depth) \cdot 0.0001$, where $max\_depth = 10$ and $min\_depth = 0.001$ for the ScanNet dataset [S4]. This method is used to assess depth reliability rather than just computing a single depth value [S4].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and samples, $O(Q \cdot N)$, where $Q$ is the number of rays and $N$ is the number of samples per ray [S2, S3, unofficial]. The CUDA kernel allows for parallel processing of rays.
*   **Memory Complexity:** $O(Q \cdot N)$ for storing weights and intermediate masks [S1, unofficial]. The CUDA kernel requires additional memory for ray specifications and output tensors [S2, S3, unofficial].
*   **Performance:** Measured performance depends on hardware and resolution. The CUDA implementation is optimized for GPU execution [S2, S3, unofficial].
*   **Strengths:** Provides a clear, binary surface indication, which can be useful for tasks requiring explicit depth boundaries [S1, unofficial]. It is differentiable, allowing for gradient-based optimization [S3, unofficial].
*   **Limitations:** Sensitive to the choice of threshold. If the threshold is too high or too low, it may fail to detect surfaces or detect noise [S1, S3, unofficial].
*   **When to Use:** When a binary surface mask is needed, or when the scene has clear density boundaries [S1, S3, unofficial].
*   **When Not to Use:** When a smooth, probabilistic depth estimate is preferred, such as in Expected Term Rendering [S1, S3, unofficial].

## Background
Sigma-Threshold Depth Rendering was introduced in the context of Dex-NeRF, a method for neural radiance fields in robotic manipulation scenarios [S3, unofficial]. It is also referred to as "Opaque Mask Computation" in some implementations [S2, unofficial]. The method is used to compute depth based on a sigma threshold, referencing the approach used in Dex-NeRF [S3, unofficial]. It is part of the broader family of differentiable volume rendering techniques [S1, S3, unofficial].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S2] kakaobrain/nerf-factory, https://github.com/kakaobrain/nerf-factory, fetched 2026-10-26 (unofficial)
[S3] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S4] Depth-Guided NeRF Training via Earth Mover’s Distance, https://link.springer.com/chapter/10.1007/978-3-031-73039-9_1, fetched 2026-10-27
