# GVDB Raycasting with Binary Search Surface Tracing

Also called: GVDB Shading and Screen-Space Flow/Depth, SDF Raycast with Binary Search Refinement, custom_iso_kernel shading, flow computation, raySurfaceTrilinearBrickCustom

## Overview
A GPU-based raycasting method that traces rays through a GVDB sparse voxel dataset to find the first intersection with an isosurface defined by a Signed Distance Field (SDF) channel [S1, unofficial][S2, unofficial]. It employs a coarse grid-based ray march followed by a binary search refinement to locate the surface, then computes surface normals, shading, and screen-space flow [S1, unofficial][S2, unofficial]. It is distinguished from sibling methods by its specific integration with the GVDB sparse voxel architecture, utilizing brick-based traversal and explicit SDF gradient computation for shading, rather than raw voxel intensity or algebraic surface approximations [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** GVDB sparse voxel dataset containing an SDF channel (`VoxelBuffer::CHANNEL_SDF`), view matrices (current and next for flow), light direction, eye direction, isosurface threshold (`SCN_THRESH`), and rendering parameters (viewport, binary search steps, specular exponent) [S1, unofficial][S2, unofficial].
*   **Output:** Per-pixel buffers containing hit position, surface normal, color (ambient + diffuse + specular), screen-space flow vector, and depth [S2, unofficial].

**Data Structures:**
*   **VolumeSampler:** A functor wrapping a reference to `VoxelData`. It provides an operator `()` that takes a 3D integer position `pos` and retrieves the SDF value from the SDF channel using `data.get_voxel(pos, VoxelBuffer::CHANNEL_SDF, defval)`, where `defval.f` is a constant representing a value far outside the surface (e.g., `constants::SDF_FAR_OUTSIDE`) [S1, unofficial].
*   **GVDB Node:** Represents a sparse voxel brick. Contains bounds (`v_min`, `v_del`) and an atlas offset for texture sampling [S2, unofficial].

**Parameters:**
*   `SCN_THRESH`: Isosurface threshold. Sampling values $\ge$ this indicate volume interior [S2, unofficial].
*   `SCN_PSTEP`: Fixed step size for coarse ray marching, aligned to brick boundaries [S2, unofficial].
*   `binarySearchSteps`: Number of binary search iterations for refinement. Typical value: 10 [S2, unofficial].
*   `specularExponent`: Shininess exponent for Blinn-Phong specular highlight [S2, unofficial].
*   `viewport`: Valid pixel range `[minX, minY, maxX, maxY]` [S2, unofficial].
*   `normal_enabled`: Boolean flag to enable/disable normal calculation [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Initialization:** For each pixel in the `viewport`, compute ray origin $\mathbf{o}$ and direction $\mathbf{d}$ from the view matrices [S2, unofficial].
2.  **Coarse Marching:** Call `raySurfaceTrilinearBrickCustom`.
    *   Get the leaf node and atlas offset for the current ray position [S2, unofficial].
    *   Convert world position $\mathbf{p}_{world}$ to index coordinates $p$ using:
        $$ p = \frac{\mathbf{p}_{world} - \mathbf{v}_{min}}{\mathbf{v}_{del}[0]} $$
        where $\mathbf{v}_{min}$ is node minimum bounds and $\mathbf{v}_{del}[0]$ is voxel size [S2, unofficial].
    *   Step the ray by `SCN_PSTEP`. At each step, sample the SDF using `tex3D` at the offset position.
    *   **Pitfall:** Voxels influenced by marching cubes are centered on their lower corner, extending 0.5 units. To prevent missing voxels due to interpolation artifacts, offset the ray sampling position by $(0.5, 0.5, 0.5)$ [S1, unofficial].
    *   If `tex3D` value $\ge$ `SCN_THRESH`, a hit is detected. Record `hit_distance_prev` (previous distance) and `hit_distance` (current distance) [S1, unofficial].
    *   If the ray exceeds volume bounds ($p.x/y/z < 0$ or $\ge$ `gvdb->res[0]`) or hits an invalid node, terminate and return no hit [S2, unofficial].
3.  **Binary Search Refinement:** If a hit was detected and `binarySearchSteps > 0`:
    *   Define interval $[t_{lower}, t_{upper}] = [\text{hit\_distance\_prev}, \text{hit\_distance}]$ [S2, unofficial].
    *   Let $\mathbf{p}_{start}$ be the ray origin at the start of the coarse step, and $\mathbf{dir}$ be the ray direction [S2, unofficial].
    *   **Adjust Start Point:** If the SDF at $\mathbf{p}_{start} + t_{lower} \cdot \mathbf{dir}$ is negative (below surface), step back by 0.5 units up to 4 times until the SDF is positive [S1, unofficial].
    *   **Adjust End Point:** If the SDF at $\mathbf{p}_{start} + t_{upper} \cdot \mathbf{dir}$ is positive (above surface), step forward by 0.5 units up to 4 times until the SDF is negative [S1, unofficial].
    *   **Loop:** For `i` from 0 to `binarySearchSteps - 1`:
        *   Calculate midpoint distance: $t_{middle} = 0.5 \cdot (t_{lower} + t_{upper})$ [S2, unofficial].
        *   Evaluate position: $\mathbf{p}_{mid} = \mathbf{p}_{start} + t_{middle} \cdot \mathbf{dir}$ [S2, unofficial].
        *   Sample SDF at $\mathbf{p}_{mid}$ (with 0.5 offset) [S2, unofficial].
        *   If SDF $\ge$ `SCN_THRESH` (inside), set $t_{upper} = t_{middle}$ [S2, unofficial].
        *   Else (outside), set $t_{lower} = t_{middle}$ [S2, unofficial].
    *   **Final Position:** Select the distance $t_{final}$ corresponding to the endpoint with the smaller absolute SDF value (typically $t_{lower}$ if the interval brackets the surface correctly) [S1, unofficial].
    *   Compute final hit point: $\mathbf{p}_{hit} = \mathbf{p}_{start} + t_{final} \cdot \mathbf{dir}$ [S2, unofficial].
4.  **Normal Computation:** If `normal_enabled`:
    *   Compute the gradient $\nabla f$ at $\mathbf{p}_{hit}$ using `get_interpolated_raw_sdf_gradient` [S1, unofficial].
    *   Normalize the gradient to get the surface normal $\mathbf{n}$ [S1, unofficial].
5.  **Shading:**
    *   **Ambient:** Start with `ambientColor` [S2, unofficial].
    *   **Diffuse:** Add `diffuseColor * max(0.0, \mathbf{n} \cdot \mathbf{l})`, where $\mathbf{l}$ is the light direction [S2, unofficial].
    *   **Specular:** Compute reflection vector $\mathbf{R} = \text{normalize}(\text{reflect}(\mathbf{l}, \mathbf{n}))$ [S2, unofficial].
        *   Compute eye direction $\mathbf{E}$ (from hit point to camera) [S2, unofficial].
        *   Add specular term:
            $$ c_{spec} \cdot \frac{\alpha + 2}{6.82} \cdot \max(0, \mathbf{R} \cdot \mathbf{E})^\alpha $$
            where $c_{spec}$ is specular color and $\alpha$ is `specularExponent` [S2, unofficial]. Note: The constant $6.82$ is $2 \times 3.41$ [S2, unofficial].
6.  **Flow and Depth:**
    *   Project $\mathbf{p}_{hit}$ into screen space using current view matrix to get $\mathbf{s}_{current}$ and next view matrix to get $\mathbf{s}_{next}$ [S2, unofficial].
    *   Compute flow:
        $$ \text{flow} = 0.5 \cdot (\mathbf{s}_{current} - \mathbf{s}_{next}) $$
        [S2, unofficial].
    *   Compute depth from $\mathbf{s}_{current}.z$ [S2, unofficial].
7.  **Output:** Write hit point, normal, color, flow, and depth to the output buffer [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gvdb_raytrace_binary_search(ray_origin, ray_dir, gvdb_data, params):
    """
    ray_origin: float3
    ray_dir: float3
    gvdb_data: GVDB dataset with SDF channel
    params: object containing SCN_THRESH, SCN_PSTEP, binarySearchSteps, specularExponent, etc.
    """
    t_current = 0.0
    p_start = ray_origin
    
    # Coarse Marching
    hit_found = False
    t_lower = 0.0
    t_upper = 0.0
    
    while True:
        # Check bounds
        p_world = p_start + t_current * ray_dir
        if not is_inside_volume(p_world, gvdb_data):
            break
            
        # Get node and offset
        node = get_node_at_point(p_world, gvdb_data)
        if node is None:
            break
            
        # Convert to index coordinates
        p_idx = (p_world - node.v_min) / node.v_del[0]
        
        # Sample SDF with 0.5 offset to account for voxel influence region
        sdf_val = sample_sdf_with_offset(p_idx + float3(0.5, 0.5, 0.5), node)
        
        if sdf_val >= params.SCN_THRESH:
            t_lower = t_current - params.SCN_PSTEP
            t_upper = t_current
            hit_found = True
            break
            
        t_current += params.SCN_PSTEP
        
    if not hit_found:
        return None, None, None, None, None

    # Binary Search Refinement
    # Adjust start point if it's inside (SDF >= threshold means inside, but we want outside for lower bound logic in some implementations, 
    # however notes say: adjust start pos0 if SDF is negative (below surface) by stepping back. 
    # Note: SDF < 0 is outside, SDF > 0 is inside. SCN_THRESH is the isosurface.
    # The code snippet says: if tex3D >= SCN_THRESH (inside), tUpper = tMiddle.
    # So tLower is outside, tUpper is inside.
    
    # Adjust t_lower (outside) if it's actually inside? 
    # Notes say: adjust start point pos0 if its SDF is negative (below surface) by stepping back 0.5 units up to 4 times until SDF is positive.
    # This implies pos0 corresponds to t_lower. If SDF at t_lower is negative (outside), we step back? 
    # Wait, if SDF is negative, it's outside. We want t_lower to be outside. 
    # Let's follow the code snippet logic strictly:
    # tLower = tCurrent - SCN_PSTEP; tUpper = tCurrent;
    # Loop: tMiddle = 0.5 * (tLower + tUpper); p = pStart + tMiddle * dir;
    # if tex3D >= SCN_THRESH: tUpper = tMiddle; else: tLower = tMiddle;
    
    # Pre-adjustment based on notes:
    # "adjust the start point pos0 if its SDF is negative (below surface) by stepping back 0.5 units up to 4 times until SDF is positive."
    # This seems contradictory to standard bisection if pos0 is the lower bound. 
    # However, we will implement the adjustment as described in notes for robustness.
    
    pos0 = p_start + t_lower * ray_dir
    for _ in range(4):
        if sample_sdf_with_offset((pos0 - node.v_min)/node.v_del[0] + float3(0.5,0.5,0.5), node) < 0: # Negative SDF
            pos0 -= ray_dir * 0.5
        else:
            break
            
    pos1 = p_start + t_upper * ray_dir
    for _ in range(4):
        if sample_sdf_with_offset((pos1 - node.v_min)/node.v_del[0] + float3(0.5,0.5,0.5), node) > 0: # Positive SDF (inside)
            pos1 += ray_dir * 0.5
        else:
            break
            
    for i in range(params.binarySearchSteps):
        t_middle = 0.5 * (t_lower + t_upper)
        p_mid = p_start + t_middle * ray_dir
        sdf_mid = sample_sdf_with_offset((p_mid - node.v_min)/node.v_del[0] + float3(0.5,0.5,0.5), node)
        
        if sdf_mid >= params.SCN_THRESH:
            t_upper = t_middle
            pos1 = p_mid
        else:
            t_lower = t_middle
            pos0 = p_mid
            
    # Select final distance based on smaller absolute SDF
    sdf_lower = sample_sdf_with_offset((p_start + t_lower * ray_dir - node.v_min)/node.v_del[0] + float3(0.5,0.5,0.5), node)
    sdf_upper = sample_sdf_with_offset((p_start + t_upper * ray_dir - node.v_min)/node.v_del[0] + float3(0.5,0.5,0.5), node)
    
    if abs(sdf_lower) < abs(sdf_upper):
        t_final = t_lower
        p_hit = p_start + t_lower * ray_dir
    else:
        t_final = t_upper
        p_hit = p_start + t_upper * ray_dir
        
    # Normal Computation
    normal = None
    if params.normal_enabled:
        grad = get_interpolated_raw_sdf_gradient(p_hit, gvdb_data)
        normal = normalize(grad)
        
    # Shading
    ambient = params.ambientColor
    diffuse = params.diffuseColor * max(0.0, dot(normal, params.lightDir))
    
    R = normalize(reflect(params.lightDir, normal))
    E = normalize(camera_position - p_hit)
    specular = params.specularColor * ((params.specularExponent + 2) / 6.82) * pow(max(0.0, dot(R, E)), params.specularExponent)
    
    color = ambient + diffuse + specular
    
    # Flow and Depth
    screen_current = project_to_screen(p_hit, params.current_view_matrix)
    screen_next = project_to_screen(p_hit, params.next_view_matrix)
    flow = 0.5 * (screen_current - screen_next)
    depth = screen_current.z
    
    return p_hit, normal, color, flow, depth
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the number of coarse steps plus binary search iterations. The coarse step size `SCN_PSTEP` determines $N$. Binary search adds a constant factor (typically 10 iterations) [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** $O(1)$ per pixel for the kernel state, plus the memory required for the GVDB sparse voxel structure [S1, unofficial][S2, unofficial].
*   **Performance:** Fast for sparse datasets due to GVDB's sparse traversal. The binary search provides high precision without excessive sampling [S1, unofficial][S2, unofficial].
*   **Strengths:** Robust surface detection with SDF, efficient sparse voxel traversal, accurate normals via SDF gradient, supports screen-space flow for temporal effects [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires SDF channel in GVDB. Coarse marching may miss very thin features if `SCN_PSTEP` is too large. Precision depends on `binarySearchSteps` [S1, unofficial][S2, unofficial].

## Background
This method is part of the Godot Voxel plugin ecosystem, specifically leveraging the GVDB (GPU Voxel Database) sparse voxel architecture [S1, unofficial]. It is used for real-time isosurface rendering in voxel-based environments, providing shading and flow information for visual effects [S1, unofficial][S2, unofficial].

## Sources
[S1] Zylann/godot_voxel, https://github.com/Zylann/godot_voxel, fetched 2026-10-26 (unofficial)
[S2] Learning adaptive sampling and reconstruction for Volume Visualization, https://github.com/shamanDevel/AdaptiveSampling, fetched 2026-10-27 (unofficial)
