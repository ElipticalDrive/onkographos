# Radiance Cascades

Also called: Radiance Intervals, Suslik's Radiance Cascades

## Overview
Radiance Cascades is a global illumination technique developed for *Path of Exile 2* that approximates indirect lighting by sampling radiance along concentric shells (cascades) around a probe [S1, unofficial]. It addresses the need for real-time inter-reflections in dynamic scenes by trading spatial resolution for angular coverage, scaling the number of rays per probe with distance to maintain consistent angular sampling density [S1, unofficial]. It differs from standard voxel cone tracing by explicitly managing the trade-off between ray count and cascade radius to optimize storage and compute costs for indirect illumination [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Scene geometry and material properties (e.g., roughness for GGX models).
*   Direct lighting information.
*   Probe placement (Screen-space or World-space).
*   Cascade parameters: Level 0 ray count ($N_0$), branching factor ($N_{level}$), Level 0 interval radius ($R_0$), and total levels ($L$) [S1, unofficial].

**Outputs:**
*   Indirect radiance values stored in cascade shells for each probe.
*   Merged and unmerged cascade data to support rough surface reflections [S1, unofficial].

### Data Structures
**World Space Probes Layout:**
The storage cost is calculated based on the probe diameter ($D$), ray count per probe ($N_{ray}$), and bytes per ray ($B_{ray}$) [S1, unofficial].
*   `rayCount`: 6 (fixed in source examples) [S1, unofficial].
*   `bytesPerRay`: 16 bytes [S1, unofficial].
*   `Total Bytes`: $D^3 \times rayCount \times bytesPerRay \times 2$ [S1, unofficial].
*   Fields: `diameter`, `megabytes`, `MB per slice` [S1, unofficial].

**Cascade Parameters:**
*   `2i spacing (level 0)`: Initial interval spacing [S1, unofficial].
*   `2r raycount (level 0)`: Initial ray count $N_0$ [S1, unofficial].
*   `branching factor`: Scaling factor for ray count per level $N_{level}$ [S1, unofficial].
*   `interval radius (level 0)`: $R_0$ [S1, unofficial].
*   `levels`: Total number of cascade levels $L$ [S1, unofficial].

### Algorithm Steps
1.  **Initialize Cascade Levels:**
    For each cascade level $i$ from $0$ to $L-1$:
    *   Calculate inner radius $r_{in}(i)$ and width $w(i)$. In 2D, each level doubles the inner radius and width relative to the previous level [S1, unofficial].
    *   Calculate ray count $N(i)$. The number of rays doubles (or scales by $N_{level}$) per level [S1, unofficial].
    *   Calculate total rays. The total number of rays per probe decreases or is managed to balance cost [S1, unofficial].

2.  **Ray Distribution:**
    For each probe at level $i$, generate $N(i)$ rays distributed within the cascade shell [S1, unofficial].
    *   Offset rays to avoid overlaps using the formula:
        $$ \theta_k = \tau \times \frac{k + 0.5}{N(i)} $$
        where $\tau$ is the full circle angle ($2\pi$), $k$ is the ray index ($0 \le k < N(i)$), and $N(i)$ is the ray count for level $i$ [S1, unofficial].

3.  **Ray Marching / Sampling:**
    *   Trace rays from the probe center through the cascade shell [S1, unofficial].
    *   In 3D Screen-space probes with World-space intervals: Place probes on min and max depth to handle high depth complexity and thin objects [S1, unofficial].
    *   In 3D World-space probes: Use a ping-pong buffer at various spatial resolutions to compute storage costs and store data [S1, unofficial].
    *   Accumulate radiance along the rays. Parameters include `accumulate non-linearly` and `accumulation decay` [S1, unofficial].

4.  **Merge and Store:**
    *   Keep both merged and unmerged cascades. Merged cascades are used for smooth reflections, while unmerged data supports roughness-based reflections (e.g., GGX) [S1, unofficial].
    *   Store results in the World Space Probes structure or Screen-space buffer [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_radiance_cascades(probe_center, cascade_params, scene):
    """
    Computes radiance cascades for a single probe.
    
    Args:
        probe_center: 3D vector, position of the probe.
        cascade_params: Object containing:
            - N0: ray count at level 0
            - branching_factor: scaling factor for ray count per level
            - R0: interval radius at level 0
            - L: total number of levels
        scene: Object providing raymarch functionality.
        
    Returns:
        List of cascade levels, each containing radiance values.
    """
    cascades = []
    current_R = cascade_params.R0
    current_N = cascade_params.N0
    
    for level in range(cascade_params.L):
        # Calculate ray offsets to avoid overlaps
        rays = []
        for k in range(current_N):
            # Angle offset formula: angle = TAU * (rayIndex + 0.5) / rayCount
            angle = 2 * math.pi * (k + 0.5) / current_N
            # In 3D, direction generation would involve spherical coordinates
            # For 2D analogy, direction is purely angular
            direction = generate_direction(probe_center, angle, level)
            rays.append(direction)
        
        # Ray march through the cascade shell
        radiance_accumulation = 0.0
        for ray_dir in rays:
            # Trace ray through scene, accumulating radiance
            # Parameters: raymarch mipmap N on cascade level N, DDA, fixed step multiplier
            sample = raymarch(probe_center, ray_dir, scene, level)
            radiance_accumulation += sample.radiance
            
        # Store cascade level data
        cascades.append({
            'level': level,
            'radius': current_R,
            'ray_count': current_N,
            'radiance': radiance_accumulation / current_N if current_N > 0 else 0.0
        })
        
        # Update parameters for next level
        # Each level doubles the inner radius and width (2D case)
        current_R *= 2
        current_N *= cascade_params.branching_factor
        
    return cascades

def generate_direction(center, angle, level):
    """
    Generates a ray direction based on angle and level.
    Note: Specific 3D direction generation logic is not detailed in sources.
    This is a placeholder for spherical distribution.
    """
    # Placeholder for 3D direction generation
    # In 2D, this would be a 2D vector
    return Vector3(1, 0, 0) # [G] Placeholder direction
```

### Edge Cases and Pitfalls
*   **High Depth Complexity:** Thin objects may be missed if probes are not placed on min and max depth in screen-space probes [S1, unofficial].
*   **Roughness Models:** Merged cascades alone are insufficient for GGX or similar roughness models; unmerged data must be retained [S1, unofficial].
*   **Precision:** Accumulation decay and non-linear accumulation parameters affect precision in deep cascades [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of probes, cascade levels, and rays per level. Ray count scales with the branching factor, while radius scales exponentially.
*   **Memory Complexity:** Proportional to the number of probes and the storage cost per probe ($D^3 \times rayCount \times bytesPerRay \times 2$) [S1, unofficial].
*   **Strengths:** Efficient real-time indirect illumination; handles dynamic scenes; supports rough reflections via unmerged cascades [S1, unofficial].
*   **Limitations:** Approximate; may miss thin geometry without careful probe placement; requires tuning of branching factor and levels [S1, unofficial].
*   **When to Use:** Real-time global illumination in dynamic environments where caching is not feasible [S1, unofficial].
*   **When Not to Use:** Offline rendering or static scenes where precomputed radiance transfer is more efficient [S1, unofficial].

## Background
Radiance Cascades was developed by Alexander Sannikov (Suslik) for *Path of Exile 2* [S1, unofficial]. It is an evolution of earlier techniques like Radiance Intervals in 2D, extending the concept to 3D with screen-space and world-space probes [S1, unofficial]. The technique has been discussed in Exilecon talks and implemented in various demos and shaders [S1, unofficial].

## Sources
[S1] Radiance Cascades - tmpvar, https://tmpvar.com/poc/radiance-cascades/, fetched 2026-10-24 (unofficial)
