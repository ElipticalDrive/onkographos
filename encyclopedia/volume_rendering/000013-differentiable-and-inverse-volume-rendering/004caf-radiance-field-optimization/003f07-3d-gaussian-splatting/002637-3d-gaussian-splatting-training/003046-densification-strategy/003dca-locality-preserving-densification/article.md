# Locality-preserving Densification

Also called: Z-ordering for Gaussians

## Overview
Locality-preserving densification addresses the memory access throttling caused by standard Gaussian densification, where newly added primitives are appended to the end of parameter buffers, causing spatially close Gaussians to be stored far apart in memory [S1, unofficial]. By reordering the Gaussian population based on 3D spatial proximity using Z-ordering (Morton order), it ensures that neighboring Gaussians in space are also adjacent in memory, thereby improving memory access coalescing [S1, unofficial]. This method sets itself apart from sibling strategies like Score-based Densification by focusing on the memory layout and hardware efficiency rather than the geometric criteria for splitting or cloning primitives [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of $N$ Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$, where each Gaussian $g_k$ has a 3D mean position $\mu_k \in \mathbb{R}^3$. The method operates on the indices of the parameter buffers.
*   **Output:** A permutation of the Gaussian indices such that the spatial locality of the Gaussians is preserved in the linear memory layout.

**Data Structures:**
*   **Gaussian Buffer:** A contiguous array of Gaussian parameters. The method only reorders the indices or pointers to these buffers; it does not modify the internal structure of the Gaussian primitives themselves.
*   **Z-order Index:** A scalar value derived from the 3D position of each Gaussian, used for sorting.

**Steps:**
1.  **Trigger Check:** Determine if the current training iteration $t$ is a multiple of the Z-ordering frequency $T_{\text{zorder}}$. If not, skip this step [S1, unofficial].
2.  **Compute Z-order Keys:** For each Gaussian $g_k$ with mean $\mu_k = (x, y, z)$, compute its Z-order (Morton) code $z_k$. This involves interleaving the bits of the quantized integer coordinates of $\mu_k$ [S1, unofficial].
    *   Let $x', y', z'$ be the integer coordinates of $\mu_k$ after quantization and offsetting to non-negative values.
    *   $z_k = \text{interleave}(x', y', z')$.
3.  **Sort:** Sort the Gaussians based on their Z-order keys $z_k$ in ascending order. This produces a new permutation $\pi$ of the indices $\{1, \dots, N\}$ [S1, unofficial].
4.  **Reorder Buffers:** Reorder all parameter buffers (means, covariances, opacities, spherical harmonics coefficients) according to the permutation $\pi$. This ensures that Gaussians with similar Z-order codes (and thus similar 3D locations) are stored contiguously in memory [S1, unofficial].

**Parameters:**
*   **Z-ordering Frequency ($T_{\text{zorder}}$):** The number of iterations between Z-ordering applications. The recommended value is 5000 iterations [S1, unofficial].
*   **Quantization Range:** The range of coordinates to be quantized for Z-ordering. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_z_ordering(gaussians, current_iteration, z_order_interval):
    """
    Reorders the Gaussian buffers based on 3D spatial locality using Z-ordering.
    
    Args:
        gaussians: A list or array of Gaussian objects, each containing a mean position 'mean' (vec3).
        current_iteration: The current training iteration number.
        z_order_interval: The frequency (in iterations) at which to apply Z-ordering.
        
    Returns:
        None (modifies gaussians in place or returns reordered buffers).
    """
    # Step 1: Check if Z-ordering should be applied
    if current_iteration % z_order_interval != 0:
        return

    n = len(gaussians)
    if n == 0:
        return

    # Step 2: Compute Z-order keys for each Gaussian
    # Note: Quantization details are not specified in the source [G]
    # Assuming a standard Morton order implementation for 3D coordinates
    z_keys = []
    for g in gaussians:
        # Quantize and offset coordinates to non-negative integers
        # This step depends on the specific coordinate space and precision used
        x, y, z = g.mean
        # Placeholder for quantization logic [G]
        # x_int, y_int, z_int = quantize(x), quantize(y), quantize(z)
        # Compute Morton code by interleaving bits
        # z_key = morton_interleave(x_int, y_int, z_int)
        # For the purpose of this pseudocode, we assume a function 'compute_z_order' exists
        z_key = compute_z_order(g.mean)
        z_keys.append(z_key)

    # Step 3: Sort Gaussians based on Z-order keys
    # Create a list of indices and sort them based on z_keys
    indices = list(range(n))
    indices.sort(key=lambda i: z_keys[i])

    # Step 4: Reorder all Gaussian parameters according to the sorted indices
    # This ensures spatially close Gaussians are adjacent in memory
    reordered_gaussians = [gaussians[i] for i in indices]
    
    # Update the global Gaussian buffer with the reordered list
    # In a real implementation, this would involve moving memory or updating pointers
    gaussians = reordered_gaussians
```

**Edge Cases and Pitfalls:**
*   **Atomic Contention:** Z-ordering performs effectively only when used with the per-Gaussian backward pass. If used with the original backward pass, numerous atomic operations heighten atomic contention, negating the benefits [S1, unofficial].
*   **Small Gaussian Counts:** Repeated Z-ordering slows down training when the number of Gaussians is small due to a massive increase in atomic contention in the alpha blending backward pass (false sharing) [S1, unofficial].
*   **Computational Cost:** The computational cost of Z-ordering is approximately 4 ms per million Gaussians [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time Complexity:** The sorting step dominates the complexity, typically $O(N \log N)$ where $N$ is the number of Gaussians. The Z-order computation is $O(N)$. The source reports a fixed cost of approximately 4 ms per million Gaussians [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the Z-order keys and the permutation indices. No additional memory is required for the Gaussian parameters themselves as they are reordered in place or via pointer swapping.

**Measured Performance:**
*   The method reduces warp divergence and cache misses, resulting in faster training when scenes contain many Gaussians [S1, unofficial].
*   It is most beneficial in scenes with a high number of Gaussians (e.g., >1.3M for indoor, >3.8M for outdoor scenes in Mip-NeRF360) [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Improves memory access coalescing, reduces warp divergence, and decreases cache misses. It is computationally efficient relative to the training time savings in large scenes.
*   **Limitations:** Not beneficial for small scenes or early training stages due to overhead and atomic contention. Requires the use of a per-Gaussian backward pass to avoid increased atomic contention [S1, unofficial].

**When to Use:**
*   Use when training large scenes with millions of Gaussians and using a per-Gaussian backward pass.
*   Apply every 5000 iterations to balance the overhead with the benefits of improved memory locality [S1, unofficial].

**When Not to Use:**
*   Do not use with the original backward pass due to atomic contention issues [S1, unofficial].
*   Avoid frequent application (e.g., every iteration) as it yields diminishing returns and slows down training, especially with fewer Gaussians [S1, unofficial].

## Background
Locality-preserving densification, also known as Z-ordering for Gaussians, was introduced in the context of optimizing 3D Gaussian Splatting training. It addresses the memory layout issues that arise during the densification process, where new Gaussians are added to the end of parameter buffers, leading to uncoalesced memory accesses [S1, unofficial]. The method leverages Z-ordering (Morton order) to spatially sort Gaussians, ensuring that neighboring Gaussians in 3D space are also close in memory [S1, unofficial]. This approach was presented as part of the Faster-GS framework, which aims to improve the optimization speed of 3D Gaussian Splatting [S1, unofficial].

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
