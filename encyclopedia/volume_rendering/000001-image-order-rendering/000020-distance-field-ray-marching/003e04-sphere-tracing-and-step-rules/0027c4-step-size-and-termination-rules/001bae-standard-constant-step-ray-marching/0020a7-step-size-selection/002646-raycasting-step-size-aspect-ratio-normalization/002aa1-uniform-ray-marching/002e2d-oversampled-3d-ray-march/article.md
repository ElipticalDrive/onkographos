# Oversampled 3D Ray March
Also called: slab tiling fix
## Overview
Oversampled 3D Ray March is a sampling strategy for distance-field ray marching that resolves discontinuities at Level-of-Detail (LOD) boundaries by ensuring every fine sample owns the slab it represents as its midpoint, clipped to the brick's segment [S1, unofficial]. It solves the problem of unprobed material at chunk exit faces caused by previous empty-space skip logic, which led to zero contribution and visual artifacts [S1, unofficial]. It sets itself apart from standard uniform marching by enforcing strict slab ownership and corrected exit-face probing, and from [NeRF Ray Marching Optimizations](9999-nerf-ray-marching-optimizations.md) by focusing on geometric tiling correctness rather than density-based acceleration [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Ray origin $O$, ray direction $D$, brick segment bounds $[0, len]$ along the ray, and LOD interface boundaries.
- Output: A sequence of sample positions and associated density/color values that tile exactly across LOD boundaries without gaps.

**Data Structures:**
- Sample Point: A 3D vector $(x, y, z)$ representing the position in world space.
- Slab: Defined by the interval $[t_{start}, t_{end}]$ along the ray parameter $t$.

**Steps:**
1. **Calculate Ray Segment:** Determine the intersection of the ray with the current brick's AABB to find the entry $t_{entry}$ and exit $t_{exit}$ parameters. The segment length is $len = t_{exit} - t_{entry}$.
2. **Determine Sample Count:** Calculate the number of fine samples $N$ required based on the desired spatial resolution or step size $s$. $N = \lceil len / s \rceil$.
3. **Compute Slab Thickness:** The thickness of each slab is $\Delta t = len / N$.
4. **Assign Slab Ownership:** For each sample index $i$ from $0$ to $N-1$:
   - The slab for sample $i$ is defined by the interval $[t_{entry} + i \cdot \Delta t, t_{entry} + (i+1) \cdot \Delta t]$.
   - The sample position $P_i$ is the midpoint of this slab:
     $$ P_i = O + D \cdot (t_{entry} + (i + 0.5) \cdot \Delta t) $$
   - This midpoint is clipped to the brick's segment to ensure it lies within $[t_{entry}, t_{exit}]$ [S1, unofficial].
5. **Correct Empty-Space Skip Probing:** When skipping empty space, the probe must extend one stride past each cube's exit face. This ensures that material in the last sub-stride is probed, preventing the cube from contributing zero and eliminating discontinuities at exit faces [S1, unofficial].
   - If using a stride $S_{stride}$, the probe position is $P_{probe} = P_{current} + D \cdot S_{stride}$.
   - The next iteration starts from this probe position, ensuring coverage of the boundary region [S1, unofficial].

**Parameters:**
- Step size $s$: Determined by the LOD level and brick size. Not given in the sources.
- Stride for empty-space skip: Typically one voxel width or level voxel size. The previous implementation strode 1.9 level voxels, which was incorrect [S1, unofficial].

**Edge Cases and Pitfalls:**
- **LOD Interfaces:** Slab thicknesses must tile $[0, len]$ exactly at LOD interfaces to avoid gaps or overlaps [S1, unofficial].
- **Exit Face Discontinuities:** Failing to probe past the exit face leaves material unprobed, causing zero contribution and visual discontinuities [S1, unofficial].
- **Precision:** Ensure floating-point precision is sufficient to handle small slab thicknesses at high LODs.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def oversampled_ray_march(ray_origin, ray_direction, brick_entry_t, brick_exit_t, step_size):
    """
    Performs oversampled 3D ray marching with correct slab tiling and exit-face probing.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray (normalized).
        brick_entry_t: float, t parameter where ray enters the brick.
        brick_exit_t: float, t parameter where ray exits the brick.
        step_size: float, desired distance between sample midpoints.
        
    Returns:
        list of 3D vectors, sample positions.
    """
    length = brick_exit_t - brick_entry_t
    if length <= 0:
        return []
    
    # Calculate number of samples
    num_samples = int(math.ceil(length / step_size))
    
    # Calculate slab thickness
    slab_thickness = length / num_samples
    
    samples = []
    for i in range(num_samples):
        # Calculate the midpoint of the slab for sample i
        t_mid = brick_entry_t + (i + 0.5) * slab_thickness
        
        # Clip to brick segment (though mathematically it should be within)
        t_clipped = max(brick_entry_t, min(t_mid, brick_exit_t))
        
        # Calculate sample position
        sample_pos = ray_origin + ray_direction * t_clipped
        samples.append(sample_pos)
        
    return samples

def correct_empty_space_skip(ray_origin, ray_direction, current_t, stride):
    """
    Calculates the next probe position for empty-space skipping, ensuring
    one stride past the exit face is probed.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        current_t: float, current t parameter.
        stride: float, distance to skip.
        
    Returns:
        float, next t parameter.
    """
    next_t = current_t + stride
    return next_t
```
## Subtopics

### Stable Linear Raymarching
A raymarching loop that uses stable, uniform linear marching between bounding bounds to eliminate artifacts like spots, missing clouds, noise, and banding, replacing unstable backing-up/bifurcated stepping loops [S2, unofficial]. It corrects empty space skipping using a robust positive-step bifurcated step size logic to handle coordinate jumping and infinite loops [S2, unofficial]. Choose this when rendering low-resolution volumes where unstable stepping causes visible artifacts [S2, unofficial]. It is a variant focused on stability in low-resolution contexts, whereas Oversampled 3D Ray March focuses on LOD tiling correctness [S1, S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$ where $N$ is the number of samples, same as standard ray marching, but with potentially more samples due to oversampling for tiling correctness [S1, unofficial].
- Memory: $O(1)$ auxiliary memory, as samples are generated on the fly [S1, unofficial].

**Measured Performance:**
- CPU replica worst-case two-cube split error: -3.99% to +0.68% [S1, unofficial].
- Browser A/B (HOA heart, pinned L4, 64 chunks, WebGPU): whole-image TV -2.7% x / -2.2% y [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Eliminates discontinuities at LOD boundaries and exit faces; ensures exact tiling of slabs [S1, unofficial].
- Limitations: Requires careful calculation of slab thickness and midpoint positioning; may increase sample count slightly [S1, unofficial].

**When to Use:**
- Use when rendering multi-LOD volumes where discontinuities at chunk boundaries are visible [S1, unofficial].
- Use when empty-space skipping causes artifacts due to unprobed material at exit faces [S1, unofficial].

**When Not to Use:**
- Not needed for single-level rendering, which takes an exact path and is bit-identical to previous methods [S1, unofficial].
- Not needed if high-resolution rendering does not exhibit LOD-related artifacts [S1, unofficial].
## Background
Originated in the context of streamed multi-LOD rendering fixes, specifically addressing ray-march ringing and slab tiling issues in the Niivue project [S1, unofficial]. It was developed to correct errors in empty-space skip logic that caused material to be unprobed at chunk exit faces [S1, unofficial]. The method was implemented and tested on both CPU and WebGPU backends, with measured improvements in visual consistency [S1, unofficial].
## Sources
[S1] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
[S2] Optimize cloud rendering with eroded Voronoi weather maps and stable linear raymarching, https://github.com/ricecake/boidsish/pull/679, fetched 2026-10-28 (unofficial)
