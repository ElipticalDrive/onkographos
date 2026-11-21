# Parallelization for Backpropagation
Also called: Per-splat Backpropagation
## Overview
Parallelization for Backpropagation is a training optimization for 3D Gaussian Splatting that replaces the standard per-pixel backward pass with a per-splat (per-Gaussian) approach to accelerate gradient computation [S1]. While the standard method computes gradients by iterating over pixels and accumulating contributions from visible Gaussians using atomic operations, this method iterates over Gaussians and updates the gradients of the pixels they cover [S1]. This shift addresses the performance bottleneck of atomic contention in the per-pixel approach, where splats contributing to arbitrary numbers of pixels require frequent global memory synchronization [S1]. It yields a 4–5× reduction in training time while maintaining numerical equivalence to the original backpropagation [S1].

## How it works
The method restructures the backward pass to operate on fixed-size buckets of Gaussians, typically aligned with GPU warp sizes (e.g., 32 Gaussians), to minimize warp stalls and false sharing [S1].

**Inputs and Outputs:**
- Input: The set of $N$ 3D Gaussians $\mathcal{G}$, the camera parameters for the current view, and the gradient of the loss with respect to the rendered pixel colors $\frac{\partial \mathcal{L}}{\partial \mathbf{c}}$.
- Output: The accumulated gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all Gaussian parameters $\theta$ (position, covariance, opacity, spherical harmonics coefficients).

**Data Structures:**
- **Gaussian Bucket:** A contiguous block of 32 Gaussians (one warp) processed in parallel.
- **Gradient Buffer:** A pre-allocated buffer for $\frac{\partial \mathcal{L}}{\partial \theta}$ for the current bucket, initialized to zero.
- **Pixel Gradient Cache:** Temporary storage for $\frac{\partial \mathcal{L}}{\partial \mathbf{c}}$ values, accessed via the rasterization order.

**Steps:**
1. **Sort and Rasterize:** Perform the forward pass to determine the depth-sorted order of Gaussians for each pixel. Store the list of visible Gaussians per pixel or, more efficiently for the backward pass, the list of pixels covered by each Gaussian (or use a tile-based approach where Gaussians are projected to screen space) [S1].
2. **Bucket Assignment:** Partition the $N$ Gaussians into buckets of size $W=32$ (warp size).
3. **Parallel Gradient Accumulation:** For each bucket:
   a. Initialize local gradient accumulators for the parameters of the Gaussians in the bucket.
   b. For each Gaussian $g_i$ in the bucket:
      i. Compute the Jacobian of the rendering equation with respect to $g_i$'s parameters. This involves the derivative of the alpha-blending function.
      ii. Identify the set of pixels $P_i$ covered by $g_i$.
      iii. For each pixel $p \in P_i$, retrieve the pre-computed pixel loss gradient $\frac{\partial \mathcal{L}}{\partial \mathbf{c}_p}$.
      iv. Compute the contribution to the gradient of $g_i$ using the chain rule: $\frac{\partial \mathcal{L}}{\partial \theta_{g_i}} += \frac{\partial \mathbf{c}_p}{\partial \theta_{g_i}} \cdot \frac{\partial \mathcal{L}}{\partial \mathbf{c}_p}$.
   c. Write the accumulated gradients for the bucket to global memory.
4. **Optimization:** Use numerically equivalent but faster formulations for the Jacobian computations, avoiding redundant operations [S1].

**Formulas:**
The core update for a Gaussian $g$ with parameters $\theta_g$ is:
$$ \frac{\partial \mathcal{L}}{\partial \theta_g} = \sum_{p \in \text{covered}(g)} \frac{\partial \mathcal{L}}{\partial \mathbf{c}_p} \frac{\partial \mathbf{c}_p}{\partial \theta_g} $$
Where $\frac{\partial \mathbf{c}_p}{\partial \theta_g}$ is the derivative of the alpha-blended color at pixel $p$ with respect to the Gaussian's parameters. The per-splat approach computes this sum by iterating over $g$ and then $p$, whereas the per-pixel approach iterates over $p$ and then $g$, requiring atomic adds for $\frac{\partial \mathcal{L}}{\partial \theta_g}$ when multiple Gaussians contribute to the same pixel.

**Parameters:**
- Bucket size: 32 (warp size) [S1].

**Edge Cases:**
- **Empty Buckets:** If $N$ is not a multiple of 32, the last bucket may contain fewer Gaussians; handle gracefully.
- **Precision:** The method claims numerical equivalence, so standard float32 precision is maintained [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_splat_backward(Gaussians, pixel_gradients, camera_params):
    """
    Gaussians: List of Gaussian objects with parameters theta.
    pixel_gradients: Array of shape (H, W, 3) containing dL/dc.
    camera_params: Camera intrinsics and extrinsics.
    """
    # Pre-compute which pixels each Gaussian covers (or use tile-based rasterization)
    # This is typically done during the forward pass or a lightweight projection step
    gaussian_pixel_map = compute_gaussian_pixel_coverage(Gaussians, camera_params)
    
    # Initialize global gradient buffer
    global_gradients = zero_like(Gaussians)
    
    # Process in warps of 32
    num_warps = (len(Gaussians) + 31) // 32
    
    for w in range(num_warps):
        start_idx = w * 32
        end_idx = min(start_idx + 32, len(Gaussians))
        bucket = Gaussians[start_idx:end_idx]
        
        # Local accumulators for this warp
        local_grads = zero_like(bucket)
        
        # Parallel execution over the warp (conceptually)
        for i, g in enumerate(bucket):
            # Get pixels covered by this Gaussian
            covered_pixels = gaussian_pixel_map[g.id]
            
            # Compute Jacobian dC/dTheta for this Gaussian
            # This involves the derivative of the alpha blending equation
            jacobian = compute_jacobian(g, camera_params)
            
            # Accumulate gradients from all covered pixels
            for p in covered_pixels:
                # dL/dTheta_g += dL/dC_p * dC_p/dTheta_g
                # Note: dC_p/dTheta_g is a vector/matrix depending on parameter type
                local_grads[i] += jacobian * pixel_gradients[p]
        
        # Write back to global memory
        for i, g in enumerate(bucket):
            global_gradients[start_idx + i] = local_grads[i]
            
    return global_gradients
```

## Subtopics

### [Per-Gaussian Backward Pass](9590-per-gaussian-backward-pass.md)
The Per-Gaussian Backward Pass is a backward pass optimization for 3D Gaussian Splatting that parallelizes gradient computation over Gaussians rather than pixels to reduce atomic contention. It addresses the performance bottleneck of standard pixel-parallel backpropagation, where splats contributing to arbitrary numbers of pixels require frequent atomic operations for accumulation. By operating on fixed-size buckets of 32 Gaussians (one GPU warp), it significantly reduces warp stalls and false sharing associated with global memory synchronization.

## Operating characteristics
- **Time Complexity:** The method reduces training time by 4–5× compared to the standard per-pixel backpropagation [S1].
- **Memory Complexity:** Requires storing the mapping of Gaussians to pixels or vice versa, which may increase memory usage slightly compared to the standard approach, but the reduction in atomic operations reduces memory bandwidth pressure [S1].
- **Strengths:** Eliminates atomic contention, reduces warp stalls, and improves GPU utilization by aligning computation with warp boundaries [S1].
- **Limitations:** Requires careful management of the Gaussian-to-pixel mapping, which may be complex to implement efficiently [S1].
- **When to use:** When training speed is a critical bottleneck and the hardware supports efficient warp-level parallelism [S1].

## Background
The method was introduced in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" (2024) as part of a broader effort to optimize the 3D Gaussian Splatting training pipeline [S1]. It addresses the scalability issues of the original 3DGS method, which suffered from slow training times due to the inefficiencies of the per-pixel backward pass [S1].

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
