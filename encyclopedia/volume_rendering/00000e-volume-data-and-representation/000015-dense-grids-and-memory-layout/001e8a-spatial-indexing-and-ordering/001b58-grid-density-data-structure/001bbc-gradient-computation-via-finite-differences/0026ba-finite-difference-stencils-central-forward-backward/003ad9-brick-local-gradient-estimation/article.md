# Brick-local Gradient Estimation

Also called: Central Differences within a Brick, Clamped Central Differences for Bricks

## Overview
Brick-local gradient estimation computes the isosurface gradient for volume rendering by applying central differences strictly within the bounds of a single data brick [S1, unofficial]. It solves the problem of indirection overhead and boundary errors when accessing neighbor voxels that may reside in adjacent bricks or lie outside the valid sampleable region [S1, unofficial]. Unlike global central difference methods that require cross-brick lookups or padding, this method clamps sample coordinates to the brick's internal `sampleMin` and `sampleMax` bounds, ensuring all accesses remain local to the current brick's memory slot [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field $V$ stored in a brick structure; the brick's spatial scale factor $h$ (often `brick.lodScale`); the current voxel position $\mathbf{p}$ (integer indices relative to the brick).
*   **Output:** A 3D gradient vector $\mathbf{G} = (G_x, G_y, G_z)$.

**Data Structures:**
The method operates on a `brick` object containing:
*   `sampleMin`: A 3D integer vector defining the inclusive lower bound of valid sample indices within the brick.
*   `sampleMax`: A 3D integer vector defining the inclusive upper bound of valid sample indices within the brick.
*   `lodScale`: A scalar $h$ representing the voxel spacing at the current level of detail.
*   `data`: The array of scalar values $V$.

**Algorithm Steps:**
For each axis $i \in \{x, y, z\}$, the gradient component $G_i$ is computed using a helper function `brickSlope`.

1.  **Define Offset Vector:** Create an offset vector $\mathbf{o}_i$ with magnitude $h$ along axis $i$ and zero elsewhere. For example, for the x-axis: $\mathbf{o}_x = (h, 0, 0)$.
2.  **Calculate Sample Positions:** Determine the high and low sample indices for the central difference stencil:
    $$ \mathbf{p}_{hi} = \text{clamp}(\mathbf{p} + \mathbf{o}_i, \text{sampleMin}, \text{sampleMax}) $$
    $$ \mathbf{p}_{lo} = \text{clamp}(\mathbf{p} - \mathbf{o}_i, \text{sampleMin}, \text{sampleMax}) $$
    Here, `clamp` restricts each coordinate of the vector to the range $[\text{sampleMin}_k, \text{sampleMax}_k]$ for each dimension $k$.
3.  **Compute Effective Span:** Calculate the projected distance between the clamped positions along the direction of the offset to handle boundary cases where the stencil is truncated:
    $$ \text{span} = \max \left( \frac{\mathbf{o}_i \cdot (\mathbf{p}_{hi} - \mathbf{p}_{lo})}{h}, 0.001 \right) $$
    The term $\mathbf{o}_i \cdot (\mathbf{p}_{hi} - \mathbf{p}_{lo})$ simplifies to $h \cdot (\mathbf{p}_{hi,i} - \mathbf{p}_{lo,i})$ because $\mathbf{o}_i$ is non-zero only in dimension $i$. The denominator $h$ normalizes this to voxel units. The minimum value $0.001$ prevents division by zero.
4.  **Sample Values:** Retrieve the scalar values at the clamped positions using affine interpolation if necessary (e.g., `sampleAtlasAffine`), or direct indexing if the positions are integer-aligned:
    $$ V_{hi} = V(\mathbf{p}_{hi}) $$
    $$ V_{lo} = V(\mathbf{p}_{lo}) $$
5.  **Compute Slope:** Calculate the gradient component:
    $$ G_i = \frac{V_{hi} - V_{lo}}{\text{span}} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_brick_local_gradient(brick, voxel_pos):
    """
    Computes the gradient at voxel_pos within a single brick.
    
    Args:
        brick: Object containing 'data' (3D array), 'sampleMin' (vec3), 
               'sampleMax' (vec3), and 'lodScale' (float h).
        voxel_pos: Tuple (x, y, z) integer indices.
    
    Returns:
        Tuple (gx, gy, gz) of float gradient components.
    """
    h = brick.lodScale
    sample_min = brick.sampleMin
    sample_max = brick.sampleMax
    data = brick.data
    
    gx, gy, gz = 0.0, 0.0, 0.0
    
    # Define axis offsets
    offsets = [
        (h, 0.0, 0.0),
        (0.0, h, 0.0),
        (0.0, 0.0, h)
    ]
    
    for i, (dx, dy, dz) in enumerate(offsets):
        # Step 2: Clamp positions
        # Calculate potential high and low indices
        px, py, pz = voxel_pos
        
        # High position: voxel + offset
        hi_x = clamp(px + (1 if dx > 0 else -1), sample_min.x, sample_max.x)
        hi_y = clamp(py + (1 if dy > 0 else -1), sample_min.y, sample_max.y)
        hi_z = clamp(pz + (1 if dz > 0 else -1), sample_min.z, sample_max.z)
        
        # Low position: voxel - offset
        lo_x = clamp(px - (1 if dx > 0 else -1), sample_min.x, sample_max.x)
        lo_y = clamp(py - (1 if dy > 0 else -1), sample_min.y, sample_max.y)
        lo_z = clamp(pz - (1 if dz > 0 else -1), sample_min.z, sample_max.z)
        
        # Step 3: Compute effective span
        # The dot product of offset and (hi - lo) is simply h * (hi_axis - lo_axis)
        # because offset is non-zero only in one dimension.
        # We use the absolute difference in indices scaled by h, but the formula 
        # in source uses dot product directly.
        # hi - lo vector
        diff_x = hi_x - lo_x
        diff_y = hi_y - lo_y
        diff_z = hi_z - lo_z
        
        # Dot product: dx*diff_x + dy*diff_y + dz*diff_z
        # Since only one component of (dx,dy,dz) is non-zero (equal to +/- h),
        # this equals h * (diff_axis).
        dot_product = dx * diff_x + dy * diff_y + dz * diff_z
        
        # Span calculation
        span = dot_product / h
        if span < 0.001:
            span = 0.001
            
        # Step 4: Sample values
        # Assuming direct indexing for integer voxel positions. 
        # If affine sampling is needed, replace with sampleAtlasAffine.
        val_hi = data[hi_x, hi_y, hi_z]
        val_lo = data[lo_x, lo_y, lo_z]
        
        # Step 5: Compute slope
        slope = (val_hi - val_lo) / span
        
        if i == 0:
            gx = slope
        elif i == 1:
            gy = slope
        else:
            gz = slope
            
    return (gx, gy, gz)

def clamp(value, min_val, max_val):
    if value < min_val:
        return min_val
    elif value > max_val:
        return max_val
    else:
        return value
```

**Parameters:**
*   `0.001`: Minimum span value to prevent division by zero. [S1, unofficial]
*   `sampleMin`/`sampleMax`: Defined by the brick's geometry and LOD level. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Boundary Truncation:** At the edges of a brick, the central difference stencil is truncated. The `span` calculation dynamically adjusts the denominator to reflect the actual distance between the sampled points, preventing artificially large gradients [S1, unofficial].
*   **Zero Span:** If a voxel is at the boundary and both neighbors are clamped to the same value (e.g., at a corner where all neighbors are clamped to the same voxel), the span would be zero. The minimum threshold `0.001` handles this [S1, unofficial].

## Subtopics

### Gradient Cache
A caching mechanism applied on a per-brick basis to store computed gradients, avoiding redundant calculations for gradients computed in the derivative-first manner [S2].

*   **Definition:** Stores previously computed gradient vectors for voxels within a brick to reuse them for subsequent rays or samples [S2].
*   **Data Structures:**
    1.  **Gradient Cache:** A 3D array of gradient vectors, sized equal to the number of voxels in the brick [S2].
    2.  **Bit List:** A bitmask or boolean array of the same size as the cache, indicating whether the gradient at each voxel has already been computed and stored [S2].
*   **Steps:**
    1.  When a gradient is required for a voxel, check the corresponding bit in the bit list [S2].
    2.  If the bit indicates the gradient is not in the cache, compute it using the brick-local estimation method and store it in the cache [S2].
    3.  After processing all rays for the current brick, flush the cache (clear the bit list) before moving to the next brick [S2].
*   **Pitfalls:**
    *   **Border Recalculation:** Gradients on the border of a brick may be calculated up to eight times (once for each adjacent brick that shares the boundary), though this is rare and causes negligible performance loss [S2].
    *   **Perspective Divergence:** High speedup is primarily observed with orthogonal projection. In perspective rendering, rays diverge for distant bricks, increasing the distance between neighboring rays and lowering the cache reuse ratio [S2].
*   **Performance:** Average speedup factor of approximately 2.2 [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel for gradient computation, assuming constant-time array access and clamping. With the Gradient Cache, the amortized cost decreases for rays that traverse multiple voxels within the same brick, particularly under orthogonal projection.
*   **Memory Complexity:** $O(N)$ per brick for the gradient cache, where $N$ is the number of voxels in the brick. This adds significant memory overhead compared to on-the-fly computation.
*   **Strengths:** Eliminates cross-brick memory indirection, improving cache locality and reducing latency in the hot loop [S1, unofficial]. Handles brick boundaries robustly without external padding [S1, unofficial].
*   **Limitations:** The Gradient Cache variant is less effective in perspective rendering due to reduced spatial coherence of rays across distant bricks [S2]. The clamping strategy may introduce slight inaccuracies in gradient magnitude at brick boundaries compared to a global stencil with proper padding, though the span correction mitigates this [S1, unofficial].
*   **When to Use:** Use when rendering large out-of-core datasets divided into bricks where memory bandwidth and cache locality are critical bottlenecks [S1, unofficial]. Use the Gradient Cache variant when rendering orthogonal views or when the scene allows for high ray coherence within bricks [S2].

## Background
The method was introduced in the context of interactive volume rendering with ray tracing, specifically to optimize the gradient computation step in derivative-first rendering approaches [S2]. The specific implementation details involving `brickSlope` and clamping to `sampleMin`/`sampleMax` are documented in the Kiln renderer project, which targets web-based, out-of-core volume rendering [S1, unofficial]. The concept of caching gradients per brick was proposed by Grimm et al. [GBKG04] and discussed in the Eurographics State of the Art Report on Interactive Volume Rendering [S2].

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
