# Brick-local Gradient Estimation

Also called: Central Differences within a Brick, Clamped Central Differences for Bricks

## Overview
Brick-local gradient estimation computes the 3D gradient vector for each voxel in a scalar volume using central differences, specifically optimized for brick-based out-of-core rendering architectures [S1, unofficial][S2]. It avoids indirection lookups in the hot rendering loop by computing gradients relative to the current brick's atlas slot, clamping sample positions to the brick's valid sampleable range to handle boundaries [S1, unofficial]. Unlike global gradient maps that require pre-computation of the entire volume, this method computes gradients on-the-fly or per-brick, trading off some boundary consistency for reduced memory bandwidth and simpler data structures [S1, unofficial][S2].

## How it works
**Inputs:**
- `voxel`: The 3D integer coordinates $(x, y, z)$ of the current voxel within the brick.
- `brick`: The current brick object containing metadata such as `lodScale` (step size $h$), `sampleMin`, and `sampleMax` (clamping bounds).
- `sampleAtlasAffine`: A function that samples the scalar value from the brick's texture/atlas using affine filtering at a given 3D position.

**Outputs:**
- `gradient`: A 3D vector $(g_x, g_y, g_z)$ representing the gradient at `voxel`.

**Data Structures:**
- `brick`: Contains fields:
  - `lodScale`: A scalar $h$ representing the step size in finest-level voxels for the current LOD.
  - `sampleMin`: A `vec3f` defining the minimum valid sample coordinate within the brick.
  - `sampleMax`: A `vec3f` defining the maximum valid sample coordinate within the brick.

**Algorithm Steps:**
1. Initialize `gradient` as a zero vector.
2. For each axis $i \in \{x, y, z\}$:
   a. Define the offset vector `offset` with $h$ in the $i$-th component and 0 elsewhere.
   b. Compute the gradient component using the `brickSlope` function:
      $$ g_i = \text{brickSlope}(\text{voxel}, \text{offset}, h, \text{brick}) $$
   c. Assign $g_i$ to the corresponding component of `gradient`.
3. Return `gradient`.

**Helper Function: `brickSlope`**
Given `voxel`, `offset`, `h`, and `brick`:
1. Calculate the high and low sample positions by adding/subtracting the offset and clamping to the brick's valid range:
   $$ \text{hi} = \text{clamp}(\text{voxel} + \text{offset}, \text{brick.sampleMin}, \text{brick.sampleMax}) $$
   $$ \text{lo} = \text{clamp}(\text{voxel} - \text{offset}, \text{brick.sampleMin}, \text{brick.sampleMax}) $$
2. Calculate the span in finest-level voxels to ensure a non-zero denominator and correct magnitude near boundaries:
   $$ \text{span} = \max\left(\frac{\text{dot}(\text{hi} - \text{lo}, \text{offset})}{h}, 0.001\right) $$
3. Sample the scalar values at the clamped positions:
   $$ v_{\text{hi}} = \text{sampleAtlasAffine}(\text{hi}, \dots) $$
   $$ v_{\text{lo}} = \text{sampleAtlasAffine}(\text{lo}, \dots) $$
4. Return the slope:
   $$ \text{slope} = \frac{v_{\text{hi}} - v_{\text{lo}}}{\text{span}} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_brick_local_gradient(voxel, brick):
    """
    Computes the 3D gradient for a voxel within a brick using clamped central differences.
    
    Args:
        voxel (vec3i): Integer coordinates of the voxel within the brick.
        brick (Brick): Object containing lodScale, sampleMin, sampleMax, and sampleAtlasAffine.
        
    Returns:
        vec3f: The computed gradient vector.
    """
    gradient = vec3f(0.0, 0.0, 0.0)
    h = brick.lodScale
    
    # Define unit vectors for each axis scaled by h
    offsets = [
        vec3f(h, 0.0, 0.0),
        vec3f(0.0, h, 0.0),
        vec3f(0.0, 0.0, h)
    ]
    
    for i in range(3):
        offset = offsets[i]
        gradient[i] = brick_slope(voxel, offset, h, brick)
        
    return gradient

def brick_slope(voxel, offset, h, brick):
    """
    Computes the slope along a specific axis for a voxel in a brick.
    
    Args:
        voxel (vec3i): Current voxel coordinates.
        offset (vec3f): Offset vector for the difference calculation.
        h (float): Step size (lodScale).
        brick (Brick): Brick metadata.
        
    Returns:
        float: The computed slope.
    """
    # Clamp sample positions to the brick's valid sample range
    hi = clamp(voxel + offset, brick.sampleMin, brick.sampleMax)
    lo = clamp(voxel - offset, brick.sampleMin, brick.sampleMax)
    
    # Calculate span in finest-level voxels
    # dot(hi - lo, offset) gives the distance along the offset direction
    # Dividing by h normalizes it, but we take max with 0.001 to avoid division by zero
    # and correct magnitude when hi == lo (e.g., at boundaries)
    span = max(dot(hi - lo, offset) / h, 0.001)
    
    # Sample values using affine filtering
    val_hi = sample_atlas_affine(hi, brick)
    val_lo = sample_atlas_affine(lo, brick)
    
    return (val_hi - val_lo) / span

def clamp(v, min_val, max_val):
    """Clamps vector v component-wise between min_val and max_val."""
    return vec3f(
        max(min(v.x, max_val.x), min_val.x),
        max(min(v.y, max_val.y), min_val.y),
        max(min(v.z, max_val.z), min_val.z)
    )

def dot(v1, v2):
    """Dot product of two 3D vectors."""
    return v1.x * v2.x + v1.y * v2.y + v1.z * v2.z

def sample_atlas_affine(pos, brick):
    """
    Samples the scalar value from the brick's atlas using affine filtering.
    Implementation details depend on the specific graphics API (e.g., WebGL2, Vulkan).
    This is a placeholder for the actual texture sampling operation.
    """
    # [G] In a real implementation, this would call a texture sampler function
    # provided by the graphics API, using the brick's texture handle and pos.
    # For the purpose of this algorithm description, we assume it returns a float.
    pass
```

## Subtopics

### Gradient Cache
A caching mechanism on a per-brick basis to store computed gradients, avoiding redundant calculations for gradients computed in the derivative-first manner [S2]. It uses two data structures: the gradient cache itself and a bit list indicating if a particular gradient has already been computed and is in the cache [S2]. The cache size equals the number of voxels in the brick [S2]. When a gradient needs to be computed, the corresponding presence bit is checked; if not set, the gradient is calculated and stored [S2]. After processing the brick, the cache is flushed for the next brick [S2]. Gradients on the border of a brick are calculated up to eight times (once per adjacent brick), but this is rare and causes no significant performance loss [S2]. An average speedup factor of approximately 2.2 can be expected [S2]. However, this high speedup is only expected with orthogonal projection; in perspective rendering, ray divergence for distant bricks increases neighbor distance, lowering cache reuse ratio [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel for gradient computation, assuming constant-time texture sampling and clamping. With the gradient cache, the average time is reduced by a factor of approximately 2.2 under orthogonal projection [S2].
- **Memory Complexity:** $O(N)$ per brick, where $N$ is the number of voxels in the brick, for the gradient cache and bit list [S2].
- **Measured Performance:** Average speedup of 2.2x with gradient cache under orthogonal projection [S2]. Performance degrades in perspective rendering due to lower cache reuse [S2].
- **Strengths:** Avoids indirection lookups in the hot loop [S1, unofficial]; reduces redundant calculations via caching [S2]; handles brick boundaries gracefully via clamping [S1, unofficial].
- **Limitations:** Gradients at brick boundaries may be inconsistent if not cached/shared [S1, unofficial]; cache effectiveness drops significantly in perspective rendering [S2]; requires additional memory for cache and bit list [S2].
- **When to use:** When rendering brick-based volumes with orthogonal projection and memory bandwidth is a bottleneck [S2].
- **When not to use:** When perspective rendering is required and cache reuse is low [S2]; when global gradient consistency is critical and pre-computation is feasible [S2].

## Background
Brick-local gradient estimation was developed as part of interactive volume rendering techniques for out-of-core datasets, specifically to optimize gradient computation in ray casting [S1, unofficial][S2]. The method leverages the spatial locality of rays within a brick to minimize memory access overhead [S1, unofficial]. The gradient cache variant was proposed by Grimm et al. [GBKG04] and discussed in the context of interactive volume rendering with ray tracing by Marmitt et al. [S2]. It is used in systems like Kiln for webGPU-native out-of-core volume rendering [S1, unofficial].

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
