# Brick-local Gradient Estimation
Also called: Central Differences within a Brick, Clamped Central Differences for Bricks

## Overview
Brick-local Gradient Estimation computes the volume gradient at a sample point by applying central finite differences using data strictly within the current brick's atlas slot, thereby avoiding expensive global coordinate lookups or indirection during ray casting [S1, unofficial]. It differs from standard [Central Difference Stencils](9968-central-difference-stencils.md) by handling brick boundaries via clamping and span normalization, ensuring gradient continuity and correct magnitude even when the stencil crosses into adjacent bricks or falls outside the sampleable range.

## How it works
**Inputs:**
- `voxel`: The 3D position of the current sample point in finest-level voxel coordinates.
- `brick`: The current brick structure containing `sampleMin`, `sampleMax`, and `lodScale` (the step size $h$).
- `sampleAtlasAffine`: A function that trilinearly interpolates the scalar field value from the brick's texture atlas at a given 3D position.

**Outputs:**
- $\nabla V = (G_x, G_y, G_z)$: The gradient vector at `voxel`.

**Data Structures:**
- `brick`: Contains `sampleMin` (vec3), `sampleMax` (vec3), and `lodScale` (vec3, typically isotropic $h$).
- `sampleAtlasAffine(p)`: Returns $V(p)$ via trilinear interpolation.

**Steps:**
1. Initialize gradient vector $\mathbf{G} = (0, 0, 0)$.
2. For each axis $i \in \{x, y, z\}$:
   a. Define offset vector $\mathbf{o}_i$ with $h_i$ in dimension $i$ and $0$ elsewhere.
   b. Compute high and low sample positions:
      $$ \mathbf{p}_{hi} = \text{clamp}(\mathbf{voxel} + \mathbf{o}_i, \mathbf{b}_{min}, \mathbf{b}_{max}) $$
      $$ \mathbf{p}_{lo} = \text{clamp}(\mathbf{voxel} - \mathbf{o}_i, \mathbf{b}_{min}, \mathbf{b}_{max}) $$
      where $\mathbf{b}_{min} = \text{brick.sampleMin}$ and $\mathbf{b}_{max} = \text{brick.sampleMax}$.
   c. Calculate the effective span in finest-level voxels:
      $$ s_i = \max \left( \frac{\mathbf{o}_i \cdot (\mathbf{p}_{hi} - \mathbf{p}_{lo})}{h_i}, 0.001 \right) $$
      This step ensures the denominator is non-zero and corrects the slope magnitude when clamping reduces the effective stencil width (e.g., near brick faces).
   d. Compute the gradient component:
      $$ G_i = \frac{\text{sampleAtlasAffine}(\mathbf{p}_{hi}) - \text{sampleAtlasAffine}(\mathbf{p}_{lo})}{s_i} $$
3. Return $\mathbf{G} = (G_x, G_y, G_z)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_brick_local_gradient(voxel, brick):
    """
    Computes the gradient of the volume scalar field at voxel position
    using brick-local central differences with clamping.
    
    Args:
        voxel: vec3, sample position in finest-level voxel coordinates.
        brick: dict/struct with keys:
            - sampleMin: vec3, minimum sampleable position.
            - sampleMax: vec3, maximum sampleable position.
            - lodScale: vec3, step size h for each axis.
            
    Returns:
        vec3, the gradient vector (Gx, Gy, Gz).
    """
    G = vec3(0.0, 0.0, 0.0)
    h = brick.lodScale
    b_min = brick.sampleMin
    b_max = brick.sampleMax
    
    # Define axes offsets
    offsets = [
        vec3(h.x, 0.0, 0.0),
        vec3(0.0, h.y, 0.0),
        vec3(0.0, 0.0, h.z)
    ]
    
    for i in range(3):
        offset = offsets[i]
        
        # Clamp sample positions to the brick's sampleable range
        p_hi = clamp(voxel + offset, b_min, b_max)
        p_lo = clamp(voxel - offset, b_min, b_max)
        
        # Calculate effective span in finest-level voxels
        # Dot product of offset and displacement gives the projected distance along the axis
        # Divide by h to normalize to voxel units
        # Use max with 0.001 to prevent division by zero
        span = max(dot(offset, p_hi - p_lo) / h[i], 0.001)
        
        # Compute gradient component
        # sampleAtlasAffine performs trilinear interpolation on the brick's texture
        val_hi = sampleAtlasAffine(p_hi, brick)
        val_lo = sampleAtlasAffine(p_lo, brick)
        
        G[i] = (val_hi - val_lo) / span
        
    return G

def clamp(v, min_val, max_val):
    """Clamps vector v component-wise to [min_val, max_val]."""
    return vec3(
        max(min(v.x, max_val.x), min_val.x),
        max(min(v.y, max_val.y), min_val.y),
        max(min(v.z, max_val.z), min_val.z)
    )

def dot(a, b):
    """Dot product of two vec3s."""
    return a.x * b.x + a.y * b.y + a.z * b.z

def sampleAtlasAffine(pos, brick):
    """
    Trilinearly interpolates the scalar field value at pos from the brick's atlas.
    Implementation details depend on the specific texture format (e.g., WebGL2, Vulkan).
    For the purpose of this algorithm, it returns V(pos).
    """
    # [G] Placeholder for actual texture sampling logic.
    # In a real implementation, this would use glTexelFetch or similar,
    # performing interpolation if pos is not on integer voxel centers.
    return interpolate_trilinear(brick.atlas, pos)
```

**Edge Cases and Pitfalls:**
- **Brick Boundaries:** When `voxel` is near the edge of the brick, `p_hi` or `p_lo` may be clamped to `b_max` or `b_min`. The span calculation $s_i$ automatically adjusts for this reduced stencil width, preventing gradient blow-up or incorrect magnitude [S1, unofficial].
- **Zero Span:** If `voxel` is exactly at a boundary and both `p_hi` and `p_lo` clamp to the same point, the span would be zero. The `max(..., 0.001)` guard prevents division by zero [S1, unofficial].
- **Precision:** Using `lodScale` (finest-level step) for span normalization ensures consistency across LOD levels, but requires careful handling of floating-point precision in `sampleAtlasAffine` if the atlas is stored at a lower resolution [S1, unofficial].

## Subtopics

### [Gradient Cache](10335-gradient-cache.md)
A per-brick caching mechanism that stores computed volume gradients to avoid redundant finite difference calculations during ray casting. It differs from precomputing gradients for the entire volume by requiring only modest, volume-size-independent memory, while offering higher speedups than cell-local caching by reusing gradients across multiple cells within a brick.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it involves a fixed number of arithmetic operations and two texture lookups per axis (6 total).
- **Memory Complexity:** $O(1)$ auxiliary memory, as it does not require storing gradient data structures beyond the current brick's texture atlas.
- **Performance:** Significantly faster than global gradient computation due to reduced memory indirection and cache locality within the brick [S1, unofficial].
- **Strengths:** Avoids boundary checks and global coordinate transformations; maintains gradient continuity across brick boundaries via clamping and span normalization.
- **Limitations:** Gradient accuracy may degrade near brick boundaries if the clamping significantly alters the effective stencil width; not suitable for high-order derivative approximations.
- **When to Use:** Real-time volume rendering where performance is critical and brick-based data structures are used.
- **When Not to Use:** When high-accuracy gradients are required across large continuous volumes without brick artifacts, or when using non-brick-based data structures.

## Background
Brick-local Gradient Estimation is a technique developed for out-of-core volume rendering systems that use brick-based data structures to manage large datasets. It was popularized by implementations such as Kiln, a WebGPU-native out-of-core volume renderer for multi-GB datasets, which leverages this method to optimize gradient computation during ray casting [S1, unofficial]. The method addresses the performance bottleneck of global gradient lookups by localizing the computation to the current brick's texture atlas.

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
