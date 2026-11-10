# GVDB Raycasting with Binary Search Surface Tracing

Also called: GVDB Shading and Screen-Space Flow/Depth, custom_iso_kernel shading, flow computation, raySurfaceTrilinearBrickCustom

## Overview
This method performs image-order raycasting on a GVDB sparse voxel dataset to render an isosurface by tracing rays from the camera, marching through the volume using fixed steps, and refining the first surface intersection via binary search [S1, unofficial]. It distinguishes itself from sibling methods by integrating surface shading, screen-space flow computation, and ambient occlusion into a single kernel that operates directly on GVDB's brick-based sparse structure, rather than relying on pre-computed meshes or uniform grids [S1, unofficial]. It builds on [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md).

## How it works
**Inputs:**
- GVDB sparse voxel dataset (`gvdb`), containing leaf nodes and atlas offsets.
- View matrices for current and next frames (for flow computation).
- Light direction (`lightDir`) and eye direction (`eyedir`).
- Parameters: `SCN_PSTEP` (march step size), `SCN_THRESH` (isosurface threshold), `binarySearchSteps` (refinement iterations, typically 10), `specularExponent` (shininess), `viewport` (pixel bounds).

**Outputs:**
- Per-pixel hit point, surface normal, color (shaded), screen-space flow vector, depth, and ambient occlusion value.

**Data Structures:**
- GVDB Leaf Node: Contains texture data for a brick of voxels. Accessed via `getNodeAtPoint`.
- Atlas Offset: 3D vector (`o`) representing the base coordinate of the current brick in the global volume space.
- Ray: Defined by origin `rpos` and direction `rdir` in world space.

**Steps:**

1. **Ray Initialization:** For each pixel within `viewport`, compute ray origin `rpos` and direction `rdir` from view matrices [S1, unofficial].

2. **Ray Marching:** Call `raySurfaceTrilinearBrickCustom` to march the ray [S1, unofficial].
   - Initialize `tCurrent` to 0.
   - Loop while `tCurrent` is within volume bounds and node is valid:
     - Get current leaf node and atlas offset `o` at `rpos + tCurrent * rdir` [S1, unofficial].
     - Convert world position to index coordinates $p$ using:
       $$ p = \frac{\text{world\_pos} - v_{min}}{v_{del}[0]} $$
       where $v_{min}$ is node minimum bounds and $v_{del}[0]$ is voxel size [S1, unofficial].
     - Sample texture using `tex3D` at $p + o$ [S1, unofficial].
     - If sample value $\ge SCN\_THRESH$, break loop (surface hit detected) [S1, unofficial].
     - Advance ray: `tCurrent += SCN_PSTEP` [S1, unofficial].
     - Check bounds: if $p.x/y/z < 0$ or $\ge gvdb->res[0]$, or node is null, terminate [S1, unofficial].

3. **Binary Search Refinement:** Upon threshold hit at `tCurrent`, refine the intersection [S1, unofficial].
   - Set `tLower = tCurrent - SCN_PSTEP` and `tUpper = tCurrent` [S1, unofficial].
   - Loop `binarySearchSteps` times:
     - Compute `tMiddle = 0.5 * (tLower + tUpper)` [S1, unofficial].
     - Compute position $p = pStart + tMiddle * dir$ [S1, unofficial].
     - Sample texture at $p + o$ [S1, unofficial].
     - If sample $\ge SCN\_THRESH$, set `tUpper = tMiddle` (hit is inside, move upper bound down) [S1, unofficial].
     - Else, set `tLower = tMiddle` (hit is outside, move lower bound up) [S1, unofficial].
   - Final hit point: $p = pStart + tLower * dir$ [S1, unofficial].

4. **Surface Property Computation:**
   - Compute normal `norm` using `getGradient` at the hit point [S1, unofficial].
   - Compute base color `color` using `getColorF` [S1, unofficial].

5. **Shading:**
   - Start with `ambientColor` [S1, unofficial].
   - Add diffuse term: `diffuseColor * abs(dot(norm, lightDir))` [S1, unofficial].
   - Compute reflection vector $R = \text{normalize}(\text{reflect3}(lightDir, norm))$ [S1, unofficial].
   - Add specular term:
     $$ \text{specular} = c_{spec} \cdot \frac{\alpha + 2}{6.82} \cdot \max(0, R \cdot E)^\alpha $$
     where $c_{spec}$ is specular color, $\alpha$ is `specularExponent`, $R$ is reflection vector, $E$ is eye direction [S1, unofficial].

6. **Flow Computation:**
   - Project hit point to screen space using current view matrix to get `screenCurrent` [S1, unofficial].
   - Project hit point to screen space using next view matrix to get `screenNext` [S1, unofficial].
   - Compute flow:
     $$ \text{flow} = 0.5 \cdot (\text{screenCurrent} - \text{screenNext}) $$
     where coordinates are normalized [S1, unofficial].

7. **Depth and AO:**
   - Compute depth from `screenCurrent.z` [S1, unofficial].
   - Compute ambient occlusion using `computeAmbientOcclusion` [S1, unofficial].

8. **Output:** Write hit point, normal, color, flow, depth, and AO to output buffer [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gvdb_ray_surface_tracing(gvdb, view_matrix_curr, view_matrix_next, light_dir, eye_dir, 
                             scn_pstep, scn_thresh, binary_search_steps, specular_exponent, 
                             viewport, ambient_color, diffuse_color, specular_color):
    # Initialize output buffers
    hit_points = []
    normals = []
    colors = []
    flows = []
    depths = []
    aos = []

    for y in range(viewport.minY, viewport.maxY):
        for x in range(viewport.minX, viewport.maxX):
            # 1. Ray Initialization
            ray_origin, ray_dir = compute_ray_from_view(view_matrix_curr, x, y)
            
            # 2. Ray Marching
            t_current = 0.0
            hit_found = False
            hit_point = None
            hit_node = None
            hit_offset = None
            
            while t_current < gvdb.max_distance:
                world_pos = ray_origin + t_current * ray_dir
                node, offset = get_node_at_point(gvdb, world_pos)
                
                if node is None:
                    break
                
                # Convert world pos to index coordinates
                p = (world_pos - node.v_min) / node.v_del[0]
                
                # Check bounds
                if p.x < 0 or p.x >= node.res[0] or \
                   p.y < 0 or p.y >= node.res[0] or \
                   p.z < 0 or p.z >= node.res[0]:
                    break
                
                # Sample texture
                sample_val = tex3d(node.texture, p.x + offset.x, p.y + offset.y, p.z + offset.z)
                
                if sample_val >= scn_thresh:
                    hit_found = True
                    hit_point = world_pos
                    hit_node = node
                    hit_offset = offset
                    break
                
                t_current += scn_pstep
            
            if not hit_found:
                continue # No hit for this pixel
            
            # 3. Binary Search Refinement
            t_lower = t_current - scn_pstep
            t_upper = t_current
            
            for i in range(binary_search_steps):
                t_middle = 0.5 * (t_lower + t_upper)
                p_mid = ray_origin + t_middle * ray_dir
                
                # Convert to index coordinates for sampling
                p_mid_idx = (p_mid - hit_node.v_min) / hit_node.v_del[0]
                
                sample_val = tex3d(hit_node.texture, p_mid_idx.x + hit_offset.x, 
                                   p_mid_idx.y + hit_offset.y, p_mid_idx.z + hit_offset.z)
                
                if sample_val >= scn_thresh:
                    t_upper = t_middle
                else:
                    t_lower = t_middle
            
            # Final hit point
            final_point = ray_origin + t_lower * ray_dir
            
            # 4. Surface Property Computation
            normal = get_gradient(hit_node.texture, final_point, hit_offset)
            base_color = get_color_f(hit_node.texture, final_point, hit_offset)
            
            # 5. Shading
            color = ambient_color
            
            # Diffuse
            diff_term = diffuse_color * abs(dot(normal, light_dir))
            color += diff_term
            
            # Specular
            reflection = normalize(reflect3(light_dir, normal))
            spec_term = specular_color * ((specular_exponent + 2) / 6.82) * pow(max(0.0, dot(reflection, eye_dir)), specular_exponent)
            color += spec_term
            
            # 6. Flow Computation
            screen_curr = project_to_screen(view_matrix_curr, final_point)
            screen_next = project_to_screen(view_matrix_next, final_point)
            flow = 0.5 * (screen_curr - screen_next)
            
            # 7. Depth and AO
            depth = screen_curr.z
            ao = compute_ambient_occlusion(gvdb, final_point, normal)
            
            # 8. Output
            hit_points.append(final_point)
            normals.append(normal)
            colors.append(color)
            flows.append(flow)
            depths.append(depth)
            aos.append(ao)

    return hit_points, normals, colors, flows, depths, aos
```

## Subtopics

### GVDB Sparse Voxel Structure
GVDB uses a hierarchical sparse voxel octree with brick-based leaf nodes to store volume data efficiently. This method leverages this structure for fast traversal and sampling, avoiding the memory overhead of dense grids. It is chosen when dealing with large, sparse volumes where memory efficiency and dynamic updates are critical. Link: [GVDB Sparse Voxel Octree](link-to-gvdb-sparse-voxel-octree.md).

### Screen-Space Flow
This technique computes motion vectors by projecting the same 3D point into screen space using consecutive view matrices. It is used for temporal anti-aliasing, motion blur, or frame interpolation. It is chosen when temporal coherence is needed without storing explicit velocity buffers. Link: [Screen-Space Flow](link-to-screen-space-flow.md).

## Operating characteristics
- **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of pixels in the viewport and $M$ is the average number of steps per ray (marching + binary search). The binary search adds a constant factor of $\log_2(\text{initial\_step\_size} / \text{desired\_precision})$.
- **Memory Complexity**: $O(V)$, where $V$ is the number of active voxels in the GVDB structure. The kernel itself uses minimal stack memory per thread.
- **Measured Performance**: Performance depends heavily on the sparsity of the volume and the distribution of the isosurface. Dense regions require more marching steps, while sparse regions may terminate early. The binary search adds negligible overhead compared to marching.
- **Scaling**: Scales well with resolution due to the independent nature of pixel processing. Scales with volume size via the sparse structure, but traversal cost increases with depth.
- **Strengths**: Efficient memory usage for sparse volumes; accurate surface refinement; integrated shading and flow computation.
- **Limitations**: Requires GVDB data structure; binary search assumes monotonicity across the step boundary; performance degrades with highly complex, non-monotonic surfaces within a single step.
- **When to Use**: Real-time rendering of large, sparse volumes where surface accuracy and temporal effects (flow) are needed.
- **When Not to Use**: Dense volumes where uniform grid traversal is faster; volumes with non-monotonic isosurfaces within small steps; when pre-computed meshes are available.

## Background
This method originates from the GVDB (GPU Voxel Database) project, specifically from the `custom_iso_kernel` implementation used in adaptive sampling and reconstruction for volume visualization [S1, unofficial]. It was developed to provide efficient, real-time isosurface rendering on sparse voxel data with integrated shading and temporal effects. It is used in interactive volume visualization tools and scientific visualization applications requiring dynamic updates and high performance.

## Sources
[S1] Learning adaptive sampling and reconstruction for Volume Visualization, https://github.com/shamanDevel/AdaptiveSampling, fetched 2026-10-27 (unofficial)
