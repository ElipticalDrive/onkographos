# Hardware Ray Tracing Global Illumination


## Overview
Hardware Ray Tracing Global Illumination approximates one-bounce diffuse global illumination by cosine-sampling the hemisphere around a surface normal, tracing short rays via hardware RT cores, and gathering indirect irradiance from hits or skylight if rays escape [S1, unofficial]. It differs from sibling methods like Voxel Cone Tracing by using explicit ray queries against voxel bricks rather than cone-based approximations, and from Precomputed Lighting by computing the indirect term per-frame rather than baking it offline [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Surface position $o$, surface normal $n$, camera direction $v$, and a voxel grid data structure containing active bricks and solid voxels.
*   **Output:** Indirect irradiance contribution to the pixel color.

**Data Structures:**
*   **GiProbe:** Stores SH-L1 coefficients for irradiance. Fields: `sh_r`, `sh_g`, `sh_b` (each `vec4<f32>`), `sh_dist`, `sh_dist2`, `info` (world grid coord + sample age), `st_r`, `st_g`, `st_b`, `st_dist`, `st_dist2`, `st_info` (staging accumulator) [S1, unofficial].
*   **Ray Query:** Hardware abstraction for tracing. Supports `RayDesc` with origin, direction, and hit range $[0.02, \text{GI\_DIST}]$ [S1, unofficial].
*   **Brick/Voxel:** Voxel grid is organized into bricks. Each brick has an active flag. Solid voxels are identified by `resolve_brick` with `skip_transparent = true` [S1, unofficial].

**Parameters:**
*   `GI_RAYS`: Number of fresh samples per frame. Value: 1 [S1, unofficial].
*   `GI_DIST`: Bounce ray reach in voxels. Value: 40.0 [S1, unofficial].
*   `GI_STRENGTH`: Multiplier for GI intensity. Value: 1.0 [S1, unofficial].
*   `GI_SKY`: Soft skylight fill factor. Value: 0.32 [S1, unofficial].

**Steps:**
1.  **Tangent Frame Construction:** Compute tangent and bitangent vectors from normal $n$ [S1, unofficial]:
    $$ \text{up} = \begin{cases} \mathbf{z} & \text{if } |n_z| > 0.9 \\ \mathbf{x} & \text{otherwise} \end{cases} $$
    $$ \mathbf{tang} = \text{normalize}(\mathbf{n} \times \text{up}) $$
    $$ \mathbf{bitan} = \mathbf{n} \times \mathbf{tang} $$
2.  **Ray Generation:** For each ray $k$ in `GI_RAYS`, generate two random numbers $r_1, r_2$ from a hash of position and seed [S1, unofficial]. Compute cosine-weighted hemisphere direction [S1, unofficial]:
    $$ rr = \sqrt{r_1} $$
    $$ \phi = 2\pi r_2 $$
    $$ \mathbf{dl} = (rr \cos \phi, rr \sin \phi, \sqrt{\max(0, 1 - r_1)}) $$
    $$ \mathbf{dir} = \text{normalize}(\mathbf{tang} \cdot \mathbf{dl}_x + \mathbf{bitan} \cdot \mathbf{dl}_y + \mathbf{n} \cdot \mathbf{dl}_z) $$
3.  **Ray Tracing (`gi_trace`):** Initialize ray query with origin $o$, direction $\mathbf{dir}$, and hit range $[0.02, \text{GI\_DIST}]$ [S1, unofficial]. Loop through candidates:
    *   If candidate is an AABB, get brick index $bi$. If `rt_brick_active(bi)` is false, continue [S1, unofficial].
    *   Call `resolve_brick(bi, skip_transparent=true)` to find the first solid, non-transparent voxel [S1, unofficial].
    *   If hit ($t \ge 0$), generate intersection `rayQueryGenerateIntersection(&rq, t)` and update `best_t` if $t < \text{best\_t}$ [S1, unofficial].
4.  **Irradiance Accumulation:**
    *   If a hit is found, accumulate light based on the hit voxel's albedo and the sun term [S1, unofficial].
    *   If no hit is found (ray escapes), accumulate skylight using `GI_SKY` [S1, unofficial].
    *   Multiply accumulated irradiance by `GI_STRENGTH` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rt_gather_indirect(o: vec3, n: vec3, v: vec3) -> vec3:
    # 1. Tangent frame
    up = vec3(0, 0, 1) if abs(n.z) > 0.9 else vec3(1, 0, 0)
    tang = normalize(cross(n, up))
    bitan = cross(n, tang)
    
    acc = vec3(0.0)
    
    # 2. Ray generation and tracing
    for k in range(GI_RAYS):
        r1, r2 = hash_randoms(o, k)
        rr = sqrt(r1)
        phi = 2 * PI * r2
        dl = vec3(rr * cos(phi), rr * sin(phi), sqrt(max(0.0, 1.0 - r1)))
        dir = normalize(tang * dl.x + bitan * dl.y + n * dl.z)
        
        # 3. Trace
        hit_voxel, hit_normal, hit_brick, hit_local = gi_trace(o, dir)
        
        if hit_voxel is not None:
            # Hit: accumulate based on albedo and sun term
            # [G] Albedo and sun term are assumed to be retrieved from hit_voxel data
            albedo = get_albedo(hit_voxel)
            sun_term = get_sun_irradiance(hit_normal)
            acc += albedo * sun_term
        else:
            # Escape: accumulate skylight
            acc += vec3(GI_SKY)
            
    return acc * GI_STRENGTH

def gi_trace(o: vec3, dir: vec3) -> tuple:
    best_t = GI_DIST
    best_hit = None
    
    # [G] Ray query initialization and iteration is hardware-specific.
    # This pseudocode abstracts the loop over ray query candidates.
    for candidate in ray_query_candidates(o, dir, 0.02, GI_DIST):
        if candidate.is_aabb():
            bi = candidate.brick_index()
            if not rt_brick_active(bi):
                continue
            
            t = resolve_brick(bi, skip_transparent=True)
            if t >= 0 and t < best_t:
                best_t = t
                best_hit = candidate.get_hit_info(t)
                
    return best_hit
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$ where $N$ is the number of pixels and $R$ is `GI_RAYS`. With `GI_RAYS=1`, it is $O(N)$ per frame [S1, unofficial].
*   **Memory Complexity:** Depends on the voxel grid and probe grid storage. Probes store SH-L1 coefficients (4 floats per channel) [S1, unofficial].
*   **Performance:** Real-time on hardware with RT cores. Performance scales with voxel grid density and ray query efficiency [S1, unofficial].
*   **Strengths:** Accurate one-bounce diffuse GI, handles complex geometry via ray tracing, no precomputation required [S1, unofficial].
*   **Limitations:** Single bounce only, noisy with low sample counts, requires hardware RT support [S1, unofficial].
*   **When to use:** Real-time scenes requiring accurate diffuse GI without precomputation, on hardware with RT cores [S1, unofficial].
*   **When not to use:** Scenes requiring multiple bounces, hardware without RT cores, or where precomputed lighting is preferred [S1, unofficial].

## Background
Originated in the `voxelG` project by mstampfli, this method leverages hardware ray tracing cores for efficient voxel-based global illumination [S1, unofficial]. It is part of a family of real-time approximations that use spatial discretization and hardware acceleration to solve the radiative transfer equation for participating media [S1, unofficial].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
