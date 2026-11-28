# Hardware Ray Tracing Global Illumination

## Overview
Hardware Ray Tracing Global Illumination is a real-time, one-bounce diffuse global illumination technique that leverages hardware-accelerated ray tracing cores to compute indirect lighting [S1, unofficial]. It approximates the indirect irradiance at a surface point by cosine-sampling the hemisphere around the surface normal, tracing short rays against a voxel grid, and gathering irradiance from hits or skylight when rays escape [S1, unofficial]. It differs from sibling methods like Voxel Cone Tracing by performing explicit per-pixel ray queries rather than cone-based approximations, trading higher per-sample computational overhead for improved geometric accuracy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Surface position $o$, surface normal $n$, camera direction $v$, and a precomputed voxel grid representing the scene geometry and lighting [S1, unofficial].
*   **Output:** Accumulated indirect irradiance contribution to the pixel color [S1, unofficial].

**Data Structures:**
The method relies on a voxel grid where each voxel (or "brick") contains metadata [S1, unofficial]. The specific internal layout of the voxel data is not detailed in the sources, but the access pattern involves querying for the first solid, non-transparent voxel along a ray [S1, unofficial]. The system uses a `RayQuery` object (standard in DXR/Vulkan RT) to manage traversal state [S1, unofficial].

**Parameters:**
*   `GI_RAYS`: Number of fresh samples per frame. Value: 1 [S1, unofficial].
*   `GI_DIST`: Bounce ray reach in voxels. Value: 40.0 [S1, unofficial].
*   `GI_STRENGTH`: Multiplier for GI intensity. Value: 1.0 [S1, unofficial].
*   `GI_SKY`: Soft skylight fill factor. Value: 0.32 [S1, unofficial].

**Algorithm Steps:**
1.  **Tangent Frame Construction:** Construct a local tangent frame from the surface normal $n$ to facilitate hemisphere sampling [S1, unofficial].
    *   Select an up vector: `up = select(z, x, abs(n.z) > 0.9)` [S1, unofficial].
    *   Compute tangent: `tang = normalize(cross(n, up))` [S1, unofficial].
    *   Compute bitangent: `bitan = cross(n, tang)` [S1, unofficial].
2.  **Ray Sampling Loop:** For each ray $k$ in `GI_RAYS`:
    *   Generate two random numbers $r_1, r_2$ from a hash of the pixel position and a seed [S1, unofficial].
    *   Compute a cosine-weighted hemisphere direction $d_{local}$:
        $$ rr = \sqrt{r_1} $$
        $$ \phi = 2\pi r_2 $$
        $$ d_{local} = (rr \cos \phi, rr \sin \phi, \sqrt{\max(0, 1 - r_1)}) $$
    *   Transform to world space:
        $$ dir = \text{normalize}(tang \cdot d_{local}.x + bitan \cdot d_{local}.y + n \cdot d_{local}.z) $$
3.  **Ray Tracing (`gi_trace`):**
    *   Initialize a `RayQuery` with origin $o$, direction $dir$, and hit range $[0.02, \text{GI\_DIST}]$ [S1, unofficial].
    *   Iterate through ray query candidates [S1, unofficial].
    *   If a candidate is an Axis-Aligned Bounding Box (AABB), retrieve its brick index $bi$ [S1, unofficial].
    *   Check if the brick is active using `rt_brick_active(bi)` [S1, unofficial]. If inactive, continue to the next candidate [S1, unofficial].
    *   Call `resolve_brick` with `skip_transparent = true` to find the first solid, non-transparent voxel within the brick [S1, unofficial].
    *   If a hit is found at distance $t \ge 0$, generate an intersection in the ray query: `rayQueryGenerateIntersection(&rq, t)` [S1, unofficial].
    *   Update the best hit distance `best_t` if $t < \text{best\_t}$ [S1, unofficial].
4.  **Irradiance Accumulation (`rt_gather_indirect`):**
    *   If a hit is recorded with voxel $hv$, normal $hn$, and brick $hbi$:
        *   Retrieve the albedo or color of the hit voxel [S1, unofficial].
        *   Accumulate light: `light += hit_color * GI_STRENGTH` [S1, unofficial].
    *   If no hit is recorded (ray escapes):
        *   Accumulate skylight: `light += GI_SKY` [S1, unofficial].
    *   The final contribution is typically scaled by the cosine of the angle between the ray direction and the surface normal, though the source notes imply the cosine-weighted sampling handles the distribution, and the accumulation sums the gathered values [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Parameters from sources
GI_RAYS = 1
GI_DIST = 40.0
GI_STRENGTH = 1.0
GI_SKY = 0.32

def rt_gather_indirect(o: vec3, n: vec3) -> vec3:
    # 1. Tangent frame construction
    up = vec3(0.0, 0.0, 1.0) if abs(n.z) > 0.9 else vec3(1.0, 0.0, 0.0)
    tang = normalize(cross(n, up))
    bitan = cross(n, tang)

    total_irradiance = vec3(0.0)

    # 2. Ray sampling loop
    for k in range(GI_RAYS):
        # Generate randoms from hash of position and seed
        r1, r2 = hash_randoms(o, k)

        # Cosine-weighted hemisphere direction
        rr = math.sqrt(r1)
        phi = 2.0 * math.pi * r2
        dl_x = rr * math.cos(phi)
        dl_y = rr * math.sin(phi)
        dl_z = math.sqrt(max(0.0, 1.0 - r1))

        dir = normalize(tang * dl_x + bitan * dl_y + n * dl_z)

        # 3. Trace ray
        hit = gi_trace(o, dir)

        # 4. Accumulate
        if hit is not None:
            total_irradiance += hit.color * GI_STRENGTH
        else:
            total_irradiance += vec3(GI_SKY)

    return total_irradiance

def gi_trace(o: vec3, dir: vec3) -> HitRecord | None:
    # Initialize ray query with origin, direction, and hit range [0.02, GI_DIST]
    best_t = GI_DIST
    best_hit = None

    # Iterate through ray query candidates
    for candidate in ray_query_candidates(o, dir, min_t=0.02, max_t=GI_DIST):
        if candidate.is_aabb():
            bi = candidate.brick_index()
            if not rt_brick_active(bi):
                continue

            # Resolve brick for first solid, non-transparent voxel
            t = resolve_brick(bi, skip_transparent=True)

            if t >= 0 and t < best_t:
                best_t = t
                best_hit = get_hit_info(candidate, t)

    return best_hit

# Helper operations
def hash_randoms(position: vec3, seed: int) -> tuple[float, float]:
    # Returns two pseudo-random floats in [0, 1) derived from position and seed
    h = hash((position.x, position.y, position.z, seed))
    return (h % 10000 / 10000.0, (h >> 16) % 10000 / 10000.0)

def ray_query_candidates(o: vec3, dir: vec3, min_t: float, max_t: float):
    # Yields intersection candidates from the hardware/software ray query engine
    # within the specified hit range. Sources do not specify the traversal implementation details.
    pass

def rt_brick_active(bi: int) -> bool:
    # Checks if the voxel brick at index bi is populated and valid.
    # Sources do not specify the brick metadata layout.
    pass

def resolve_brick(bi: int, skip_transparent: bool) -> float:
    # Returns the distance to the first solid, non-transparent voxel in the brick.
    # Sources do not specify the voxel traversal logic.
    pass

def get_hit_info(candidate, t: float) -> HitRecord:
    # Extracts hit voxel color, normal, and brick index from the ray query state.
    # Sources do not specify the hit record extraction logic.
    pass
```

## Subtopics

### [Dynamic Diffuse Global Illumination (DDGI)](8142-dynamic-diffuse-global-illumination-ddgi.md)
DDGI extends this concept by maintaining persistent probe data that adapts to geometry changes over time, offering $O(1)$ rendering complexity but with slightly lower quality than Voxel Cone Tracing and no specular support.

### [Temporal GI Accumulation with Reprojection](15376-temporal-gi-accumulation-with-reprojection.md)
Not given in the sources.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$ where $N$ is the number of pixels/samples and $R$ is the number of rays per sample (typically 1) [S1, unofficial]. The cost is dominated by the hardware ray traversal [S1, unofficial].
*   **Memory Complexity:** Depends on the voxel grid size [S1, unofficial]. The method itself requires minimal per-pixel state, storing only the accumulated irradiance [S1, unofficial].
*   **Performance:** Performance scales with the number of active bricks and the depth of the ray traversal [S1, unofficial]. Hardware RT cores provide significant acceleration over software ray tracing [S1, unofficial].
*   **Strengths:** High geometric accuracy for indirect lighting; handles complex occlusion and shadowing naturally via ray tracing; simple to implement on modern hardware [S1, unofficial].
*   **Limitations:** Limited to one-bounce illumination (diffuse GI); noisy with low sample counts (mitigated by temporal accumulation); no specular reflections; performance can degrade with deep voxel grids or high `GI_DIST` values [S1, unofficial].
*   **When to use:** Real-time applications requiring accurate diffuse global illumination where hardware RT is available [S1, unofficial].
*   **When not to use:** Applications requiring specular GI, multi-bounce lighting, or running on hardware without RT cores [S1, unofficial].

## Background
This method is derived from techniques demonstrated in the `voxelG` project, which implements real-time global illumination using hardware ray tracing and voxel grids [S1, unofficial]. It represents a shift from traditional rasterization-based GI approximations (like VCT) towards explicit ray tracing for indirect lighting calculations [S1, unofficial].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
