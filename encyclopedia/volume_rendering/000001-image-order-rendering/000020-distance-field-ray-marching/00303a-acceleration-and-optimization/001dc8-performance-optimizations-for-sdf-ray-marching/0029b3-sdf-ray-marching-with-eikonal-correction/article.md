# SDF Ray Marching with Eikonal Correction

Also called: Eikonal-corrected ray marching, distance field ray marching with step correction

## Overview
This method performs ray marching through a Signed Distance Field (SDF) to find surface intersections, specifically addressing the geometric error introduced by the Eikonal equation. While standard ray marching assumes the sampled distance value represents the true Euclidean distance to the nearest surface, the Eikonal equation causes these distances to elongate along non-cardinal directions relative to the grid axes [S1, unofficial]. This correction ensures that the step size accurately reflects the true distance, preventing overshooting or jittering artifacts when the ray direction is not aligned with the SDF grid's cardinal axes. It builds on [Performance Optimizations for SDF Ray Marching](1629-performance-optimizations-for-sdf-ray-marching.md) by adding a directional correction factor to the step calculation.

## How it works
**Inputs:**
- Ray origin $O$ and direction $D$ (normalized).
- SDF texture or field $SDF(x, y, z)$.
- Maximum ray march steps $N_{max}$ (typical value: 128, range 8–512) [S1, unofficial].
- Hit threshold distance $\epsilon$ (implicit in `hitThreshold()`).
- Progress multiplier $k$ (16.0 for ray origin march, 32.0 for shadow march) [S1, unofficial].
- Eikonal correction factor $C_{eikonal}$, computed from ray direction $D$.

**Outputs:**
- Boolean hit status (1.0 if hit, 0.0 if miss).
- Hit position (if hit).
- Accumulated progress/resolution metric.

**Data Structures:**
- The SDF is typically stored as a 3D texture or voxel grid. Access is via world-space coordinates mapped to texture coordinates.
- No additional complex data structures are required beyond the standard ray parameters and SDF lookup.

**Algorithm Steps:**
1. Initialize current ray distance $t = 0.0$ and progress accumulator $res = 1.0$.
2. Loop up to $N_{max}$ iterations:
   a. Calculate current world position: $P = O + D \cdot t$.
   b. Check bounds: If $P$ is out of the SDF volume, return 0.0 (miss).
   c. Sample SDF: $d = SDF(P)$.
   d. Check hit: If $d \le \epsilon$, set hit position to $P$, set $res = 0.0$, and return 1.0 (hit).
   e. Apply Eikonal correction: Clamp $d$ to a minimum positive value (e.g., $0.0001$) to avoid division by zero or zero-step stalls, then multiply by the correction factor: $d_{corrected} = \max(d, 0.0001) \cdot C_{eikonal}$.
   f. Update progress: $res = \min(res, k \cdot d_{corrected} / t)$ (if $t > 0$).
   g. Advance ray: $t = t + d_{corrected}$.
3. If the loop completes without hitting, return 1.0 (miss).

**Formula:**
The core step advancement formula is:
$$ t_{new} = t_{old} + \max(d, 0.0001) \cdot C_{eikonal} $$
where $d$ is the raw SDF value at the current position, and $C_{eikonal}$ is the correction factor derived from the ray direction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_eikonal_corrected(origin, direction, sdf_sample, max_steps=128, k=16.0, hit_threshold=0.001):
    """
    Ray march through an SDF with Eikonal correction.
    
    Args:
        origin: Vector3, ray origin.
        direction: Vector3, normalized ray direction.
        sdf_sample: Function, takes world position and returns float distance.
        max_steps: int, maximum number of march steps.
        k: float, progress/resolution multiplier.
        hit_threshold: float, distance below which a hit is registered.
        
    Returns:
        float: 1.0 if hit, 0.0 if miss.
    """
    t = 0.0
    res = 1.0
    
    # Pre-calculate correction factor based on direction
    # Note: The exact formula for calcEikonalCorrection is not provided in sources.
    # It is a function of the ray direction 'direction'.
    eikonal_fix = calc_eikonal_correction(direction) 
    
    for _ in range(max_steps):
        # Current position
        current_pos = origin + direction * t
        
        # Check bounds (implementation depends on SDF volume definition)
        if not is_within_sdf_bounds(current_pos):
            return 0.0
            
        # Sample SDF
        dist = sdf_sample(current_pos)
        
        # Check for hit
        if dist <= hit_threshold:
            # Hit detected
            return 1.0
            
        # Apply Eikonal correction
        # Clamp to small positive value to prevent zero-step or negative-step issues
        corrected_dist = max(dist, 0.0001) * eikonal_fix
        
        # Update progress metric (optional, for resolution tracking)
        if t > 0:
            res = min(res, k * corrected_dist / t)
            
        # Advance ray
        t += corrected_dist
        
    # Max steps reached without hit
    return 0.0

def calc_eikonal_correction(direction):
    """
    Calculates the correction factor to counteract distance elongation
    along non-cardinal directions due to the Eikonal equation.
    
    Note: The specific mathematical formula for this correction is not
    provided in the source material [S1, unofficial]. It is assumed to
    be a function that returns a scalar multiplier >= 1.0, depending
    on the alignment of 'direction' with the SDF grid axes.
    """
    # Placeholder for the actual correction logic.
    # In practice, this might involve trigonometric functions of the
    # direction vector components relative to the grid.
    # Since the source does not provide the formula, we return 1.0
    # as a neutral default, but in implementation, this must be
    # replaced with the correct geometric correction.
    return 1.0

def is_within_sdf_bounds(pos):
    """
    Checks if the position is within the SDF volume.
    Implementation depends on the specific SDF storage (e.g., AABB).
    """
    # Not given in the sources. Assume a simple AABB check.
    # return min_bounds <= pos <= max_bounds
    return True # Placeholder
```

**Parameters:**
- `max_steps`: 128 (typical), range 8–512 [S1, unofficial].
- `k`: 16.0 for ray origin march, 32.0 for shadow march [S1, unofficial].
- `eikonal_fix`: Calculated by `calcEikonalCorrection(dir)` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Distance Elongation:** The primary pitfall is that the Eikonal equation causes distance values to be larger than the true Euclidean distance along non-cardinal directions. This leads to larger step sizes than necessary, potentially skipping over thin features or causing jitter. The `eikonal_fix` factor corrects this [S1, unofficial].
- **Zero Distance:** If `dist` is 0 or negative (inside the surface), it is clamped to 0.0001 to prevent division by zero or infinite loops in progress calculation [S1, unofficial].
- **Out of Bounds:** Rays leaving the SDF volume must be handled explicitly to avoid sampling errors [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of ray march steps (up to `max_steps`). Each step involves one SDF sample and basic arithmetic.
- **Memory Complexity:** $O(1)$ auxiliary memory, plus the memory for the SDF texture itself.
- **Performance:** The correction factor calculation adds minimal overhead. The main cost is the SDF sampling. The correction improves accuracy, potentially reducing the number of steps needed for precise hits, but may increase steps if the correction factor is large (i.e., if the ray is highly non-cardinal).
- **Strengths:** Improves accuracy of ray marching in SDFs, especially for rays not aligned with grid axes. Prevents overshooting.
- **Limitations:** Requires a method to calculate the Eikonal correction factor, which is not detailed in the source. Performance depends on the efficiency of the SDF sampling.
- **When to use:** When high accuracy is required in SDF ray marching, particularly for non-axis-aligned rays.
- **When not to use:** If performance is critical and approximate results are acceptable, or if rays are always aligned with cardinal directions.

## Background
This method is part of the implementation of signed distance fields in Unity, as described in an unofficial GitHub repository by samuelbigos [S1, unofficial]. It addresses a known issue in SDF rendering where the Eikonal equation, used to compute the distance field, introduces directional bias in the distance values. This bias is corrected during ray marching to ensure accurate surface intersection detection.

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
