# Locality-preserving Densification

Also called: Z-ordering for Gaussians

## Overview
Locality-preserving densification addresses the memory access throttling caused by standard Gaussian densification, where new primitives are appended to the end of parameter buffers, causing spatially close Gaussians to be stored far apart in memory [S1, unofficial]. By reordering the Gaussian set based on 3D spatial proximity using Z-ordering (Morton order), it ensures that neighboring Gaussians in space are also close in memory, thereby improving memory access coalescing [S1, unofficial]. It sets itself apart from sibling methods like [Score-based Densification](4705-score-based-densification.md) or [Adaptive Density Control](9188-adaptive-density-control.md) by focusing on data layout optimization rather than geometric splitting criteria, though it is typically applied in conjunction with them [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of $N$ Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$, where each Gaussian $g_i$ has a 3D mean position $\mathbf{p}_i \in \mathbb{R}^3$.
*   **Output:** A permutation of the Gaussian indices such that the reordering preserves spatial locality.

**Data Structures:**
*   **Gaussian Buffer:** A contiguous array of Gaussian parameters. Each Gaussian $g_i$ is stored with fields including mean position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and spherical harmonics coefficients.
*   **Z-order Key:** A 64-bit integer derived from the 3D coordinates of the Gaussian mean.

**Steps:**
1.  **Trigger Check:** Determine if the current training step $t$ satisfies the condition for Z-ordering. The method is applied only when densification is active and at regular intervals.
2.  **Coordinate Normalization:** Normalize the 3D mean positions $\mathbf{p}_i$ of all Gaussians into a canonical unit cube $[0, 1]^3$. This requires knowing the bounding box of the scene or the current Gaussian distribution. Let $\mathbf{p}'_i = (x'_i, y'_i, z'_i)$ be the normalized coordinates.
3.  **Z-order Encoding:** For each Gaussian $i$, compute the Morton code $m_i$ from the normalized coordinates. The Morton code interleaves the bits of the integer representations of the normalized coordinates. For a 64-bit integer, if the coordinates are quantized to 21 bits each (total 63 bits), the interleaving is performed as follows:
    $$ m_i = \text{interleave}(x'_i, y'_i, z'_i) $$
    Where $\text{interleave}$ maps the $k$-th bit of $x, y, z$ to the $3k, 3k+1, 3k+2$ bits of the result.
4.  **Sorting:** Sort the Gaussians based on their Morton codes $m_i$. This can be done using a radix sort or a standard comparison sort on the 64-bit integers.
5.  **Buffer Reordering:** Permute the Gaussian parameters in the GPU buffer according to the sorted order. This involves moving the entire Gaussian structure (mean, covariance, opacity, SH coefficients) to maintain data integrity.

**Parameters:**
*   **Frequency:** Z-ordering is applied every 5000 iterations [S1, unofficial].
*   **Quantization Bits:** Not explicitly given in the sources, but standard Morton ordering for 3D coordinates typically uses 21 bits per axis for 64-bit integers [G].

**Edge Cases and Pitfalls:**
*   **Atomic Contention:** Z-ordering performs effectively only when used with the per-Gaussian backward pass. If the original backward pass with numerous atomic operations is used, Z-ordering heightens atomic contention [S1, unofficial].
*   **Small Gaussian Counts:** Repeated Z-ordering slows down training when the number of Gaussians is small due to a massive increase in atomic contention in the alpha blending backward pass (false sharing) [S1, unofficial].
*   **Bounding Box:** Accurate normalization requires a consistent bounding box. If the scene bounds change significantly, the Morton codes may not reflect relative locality correctly [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_z_ordering(gaussians, step, interval=5000):
    """
    Reorders Gaussians in the buffer using Z-ordering (Morton order)
    to improve memory coalescing.
    
    Args:
        gaussians: List of Gaussian objects with attribute 'mean' (vec3).
        step: Current training step.
        interval: Frequency of Z-ordering application.
        
    Returns:
        None (modifies gaussians in place or returns new order).
    """
    # Step 1: Trigger Check
    if step % interval != 0:
        return

    # Step 2: Coordinate Normalization
    # Determine bounding box of current Gaussians
    min_x = min(g.mean.x for g in gaussians)
    max_x = max(g.mean.x for g in gaussians)
    min_y = min(g.mean.y for g in gaussians)
    max_y = max(g.mean.y for g in gaussians)
    min_z = min(g.mean.z for g in gaussians)
    max_z = max(g.mean.z for g in gaussians)
    
    # Handle degenerate cases where min == max
    dx = max_x - min_x if max_x != min_x else 1.0
    dy = max_y - min_y if max_y != min_y else 1.0
    dz = max_z - min_z if max_z != min_z else 1.0
    
    # Normalize to [0, 1]
    normalized_coords = []
    for g in gaussians:
        x = (g.mean.x - min_x) / dx
        y = (g.mean.y - min_y) / dy
        z = (g.mean.z - min_z) / dz
        normalized_coords.append((x, y, z))
        
    # Step 3: Z-order Encoding
    # Quantize to 21 bits (0 to 2^21 - 1)
    bits = 21
    scale = (1 << bits) - 1
    
    morton_codes = []
    for x, y, z in normalized_coords:
        ix = int(x * scale) & ((1 << bits) - 1)
        iy = int(y * scale) & ((1 << bits) - 1)
        iz = int(z * scale) & ((1 << bits) - 1)
        code = interleave_bits_3d(ix, iy, iz)
        morton_codes.append(code)
        
    # Step 4: Sorting
    # Create index array and sort based on morton_codes
    indices = list(range(len(gaussians)))
    indices.sort(key=lambda i: morton_codes[i])
    
    # Step 5: Buffer Reordering
    # Permute the gaussians list according to indices
    reordered_gaussians = [gaussians[i] for i in indices]
    
    # Update the global gaussian buffer with reordered_gaussians
    # Note: In implementation, this involves GPU memory copy or pointer swap
    update_gaussian_buffer(reordered_gaussians)

def interleave_bits_3d(x, y, z):
    """
    Interleaves the bits of three 21-bit integers into a 64-bit integer.
    """
    # Simple bit interleaving for 21 bits
    # This is a simplified version; production code may use lookup tables or bit hacks
    result = 0
    for i in range(21):
        result |= ((x >> i) & 1) << (3 * i)
        result |= ((y >> i) & 1) << (3 * i + 1)
        result |= ((z >> i) & 1) << (3 * i + 2)
    return result
```

## Operating characteristics
*   **Time Complexity:** The sorting step dominates, typically $O(N \log N)$ or $O(N)$ with radix sort. The computational cost is approximately 4 ms per million Gaussians [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the Morton codes and the permutation indices.
*   **Measured Performance:** Reduces warp divergence and cache misses, resulting in faster training when scenes contain many Gaussians [S1, unofficial]. It outperforms load-balanced approaches for Gaussian/tile instances, which can cause uncoalesced memory accesses that become a bigger bottleneck as primitives increase [S1, unofficial].
*   **Strengths:** Improves memory access patterns, reduces cache misses, and accelerates training for large-scale scenes.
*   **Limitations:** Ineffective or detrimental for small Gaussian counts due to false sharing and atomic contention. Requires the per-Gaussian backward pass to avoid heightened atomic contention [S1, unofficial].
*   **When to Use:** Use when training scenes with a large number of Gaussians (e.g., >1M) and using the per-Gaussian backward pass.
*   **When Not to Use:** Avoid for small scenes or when using the original backward pass with atomic operations.

## Background
Locality-preserving densification, also known as Z-ordering for Gaussians, was introduced in the context of optimizing 3D Gaussian Splatting training. It addresses the memory layout issues identified in the Faster-GS framework [S1, unofficial]. The method leverages Z-ordering (Morton order), a space-filling curve technique, to map multi-dimensional data to one dimension while preserving locality [S1, unofficial]. This approach is particularly relevant in the context of differentiable volume rendering and inverse rendering, where memory bandwidth is a critical bottleneck.

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
