# Conservative Rasterization

Also called: Conservative Raster via MSAA, Conservative Rasterization for Voxelization, Conservative Rasterization via Geometric Dilatation, Conservative Rasterization via MSAA, GPU Voxelization Conservative Rasterization, Hertel et al. Conservative Rasterization, MSAA render target voxelization, Triangle Expansion for Voxelization

## Overview
Conservative rasterization is a technique used during voxelization to prevent artifacts where small triangles (smaller than the voxel grid resolution) are ignored completely, resulting in holes in the voxelized geometry [S3, unofficial]. It ensures that all voxels potentially intersected by a triangle are covered by rasterization [S4, unofficial]. Two primary implementations exist: **Geometric Dilatation**, which explicitly enlarges triangles in clip space within the geometry shader [S1, S4, unofficial], and **MSAA Approximation**, which leverages Multi-Sample Anti-Aliasing to invoke the pixel shader if any subsample is covered, approximating conservative behavior without explicit edge expansion [S2, unofficial]. The MSAA approach is often preferred for its simplicity and lack of redundant voxels compared to edge extension [S2, unofficial].

## How it works
This section details the **Geometric Dilatation** method, as it provides a deterministic, algorithmic definition. The MSAA method is described as a parameterized alternative in Operating characteristics.

### Inputs and Outputs
*   **Input:** A triangle defined by three vertices $P_0, P_1, P_2$ in clip space (or view space, depending on pipeline stage).
*   **Output:** A dilated triangle with vertices $P'_0, P'_1, P'_2$ in clip space, which is then transformed to world space for voxelization.
*   **Uniforms:**
    *   `halfPixel[3]`: Array of `vec2` representing half the pixel size in clip space, indexed by the selected view projection [S4, unofficial].
    *   `viewProject[3]`: Array of 3 `mat4` matrices for view-projection transformations [S4, unofficial].
    *   `viewProjectInverse[3]`: Array of 3 `mat4` matrices for inverse view-projection transformations [S4, unofficial].

### Data Structures
*   **Edge Planes:** Three `vec3` values representing the homogeneous plane equations for the triangle edges.
*   **Triangle Plane:** One `vec4` representing the plane equation of the original triangle.
*   **Intersection Lines:** Three `vec3` values representing the direction vectors of the intersection lines between offset edge planes.
*   **Bounding Box:** `vec4` containing min/max x/y coordinates in clip space, used in the fragment shader for culling [S4, unofficial].

### Numbered Steps

1.  **Select Projection Axis:** Determine the dominant axis of the triangle's face normal to select the appropriate orthographic projection matrix from `viewProject` [S4, unofficial]. Let this index be `projectIndex`.
2.  **Compute Triangle Plane:** Calculate the normal $N = (A, B, C)$ of the triangle in world/view space using the cross product of edge vectors:
    $$ N = \text{normalize}((P_1 - P_0) \times (P_2 - P_0)) $$
    Compute the plane constant $D$:
    $$ D = -P_0 \cdot N $$
    The plane equation is $Ax + By + Cz + D = 0$ [S4, unofficial].
3.  **Ensure Consistent Winding:** Check if the triangle is back-facing relative to the camera. If the z-component of the normal is negative ($N_z < 0$), swap vertices $P_1$ and $P_2$ to ensure counter-clockwise winding [S4, unofficial].
4.  **Compute Homogeneous Edge Planes:** For each edge $i$ (connecting $P_i$ to $P_{i+1}$, with $P_3=P_0$), compute the homogeneous plane passing through the origin. The normal $(A_i, B_i, C_i)$ is found via the cross product of the homogeneous vertex coordinates (using $x, y, w$ components):
    $$ \text{edgePlanes}[i] = \text{cross}(P_{i+1}.xyw - P_i.xyw, P_i.xyw) $$
    Note: In orthographic projection, $w=1$, so this represents a 2D line in clip space [S4, unofficial].
5.  **Dilate Edge Planes:** Offset each homogeneous plane to expand the triangle. Subtract the projection of the half-pixel size onto the plane's normal from the plane's $z$-component (distance):
    $$ \text{edgePlanes}[i].z \leftarrow \text{edgePlanes}[i].z - \text{dot}(\text{halfPixel}[\text{projectIndex}], |\text{edgePlanes}[i].xy|) $$
    [S4, unofficial].
6.  **Compute Intersection Lines:** Calculate the direction vector of the intersection line between adjacent offset planes. The direction is the cross product of their normals. Normalize by dividing by the $z$-component:
    $$ \text{intersection}[i] = \text{cross}(\text{edgePlanes}[i], \text{edgePlanes}[i+1]) $$
    $$ \text{intersection}[i] \leftarrow \text{intersection}[i] / \text{intersection}[i].z $$
    [S4, unofficial].
7.  **Compute Dilated Vertices:** The $x$ and $y$ coordinates of the new vertices are taken from the intersection line vectors. The $z$ coordinate is solved using the original triangle's plane equation:
    $$ z_i = -( \text{intersection}[i].x \cdot A + \text{intersection}[i].y \cdot B + D ) / C $$
    If $C = 0$ (triangle plane parallel to view direction), the algorithm returns early or handles the singularity [S4, unofficial].
    $$ P'_i = (\text{intersection}[i].x, \text{intersection}[i].y, z_i) $$
    [S4, unofficial].
8.  **Transform and Output:** Transform the dilated vertices $P'_i$ back to world space using `viewProjectInverse[projectIndex]` for voxelization. Pass the clip-space positions to the fragment shader [S4, unofficial].

### Pseudocode (Geometry Shader)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: gl_in[3] (vec4 positions in clip space)
# Uniforms: halfPixel[3] (vec2), viewProjectInverse[3] (mat4)

def conservative_rasterization_geometry_shader(gl_in):
    # 1. Select projection axis based on face normal
    p1 = gl_in[1].xyz - gl_in[0].xyz
    p2 = gl_in[2].xyz - gl_in[0].xyz
    face_normal = cross(p1, p2)
    
    n_dx = abs(face_normal.x)
    n_dy = abs(face_normal.y)
    n_dz = abs(face_normal.z)
    
    if n_dx > n_dy and n_dx > n_dz:
        project_index = 0
    elif n_dy > n_dx and n_dy > n_dz:
        project_index = 1
    else:
        project_index = 2
        
    # 2. Compute Triangle Plane (in world/view space, assuming input is world/view)
    # Note: Source [S4, unofficial] computes normal from xyz components. 
    # If input is clip space, transformation may be needed first. 
    # Assuming standard pipeline where GS receives transformed vertices or handles projection.
    # Based on [S4, unofficial] code: pos is treated as having xyz for plane calc.
    pos = [gl_in[0].xyz, gl_in[1].xyz, gl_in[2].xyz]
    
    triangle_normal = normalize(cross(pos[1] - pos[0], pos[2] - pos[0]))
    triangle_d = -dot(pos[0], triangle_normal)
    
    # 3. Ensure consistent winding (counter-clockwise relative to camera)
    if dot(triangle_normal, vec3(0.0, 0.0, 1.0)) < 0.0:
        # Swap vertex 1 and 2
        temp = pos[2]
        pos[2] = pos[1]
        pos[1] = temp
        
    # Handle degenerate case: plane parallel to view direction
    if abs(triangle_normal.z) < 1e-6:
        return # Skip or handle as needed

    # 4. Compute Homogeneous Edge Planes
    # Use clip space positions for edge planes if projecting orthographically in clip space
    # Source [S4, unofficial] uses pos.xyw. Assuming pos here refers to clip space positions gl_in[i].xyzw
    clip_pos = [gl_in[0], gl_in[1], gl_in[2]]
    
    edge_planes = [vec3(0.0), vec3(0.0), vec3(0.0)]
    for i in range(3):
        p_curr = clip_pos[i]
        p_next = clip_pos[(i + 1) % 3]
        # Cross product of homogeneous coordinates (x, y, w)
        # vec3 cross(vec3 a, vec3 b)
        edge_planes[i] = cross(vec3(p_next.x - p_curr.x, p_next.y - p_curr.y, p_next.w - p_curr.w), 
                               vec3(p_curr.x, p_curr.y, p_curr.w))
                               
    # 5. Dilate Edge Planes
    half_pix = halfPixel[project_index]
    for i in range(3):
        # Subtract projection of half-pixel onto plane's xy normal from z component
        edge_planes[i].z -= dot(half_pix, abs(vec2(edge_planes[i].x, edge_planes[i].y)))
        
    # 6. Compute Intersection Lines
    intersection = [vec3(0.0), vec3(0.0), vec3(0.0)]
    for i in range(3):
        p1 = edge_planes[i]
        p2 = edge_planes[(i + 1) % 3]
        intersection[i] = cross(p1, p2)
        # Normalize by z-component
        if abs(intersection[i].z) > 1e-6:
            intersection[i] /= intersection[i].z
        else:
            intersection[i] = vec3(0.0) # Degenerate
            
    # 7. Compute Dilated Vertices
    dilated_pos = [vec3(0.0), vec3(0.0), vec3(0.0)]
    for i in range(3):
        # x, y from intersection line
        # z from triangle plane equation: Ax + By + Cz + D = 0 => z = -(Ax + By + D)/C
        ix = intersection[i].x
        iy = intersection[i].y
        iz = -(ix * triangle_normal.x + iy * triangle_normal.y + triangle_d) / triangle_normal.z
        dilated_pos[i] = vec3(ix, iy, iz)
        
    # 8. Transform and Emit
    for i in range(3):
        # Transform dilated clip-space vertex to world space for voxelization
        # Note: Source [S4, unofficial] applies viewProjectInverse to the dilated clip-space pos
        world_pos = viewProjectInverse[project_index] * vec4(dilated_pos[i], 1.0)
        
        # Set outputs for Fragment Shader
        # FragPos (world space), ProjectPos (clip space)
        emit_vertex(world_pos.xyz, dilated_pos[i])

def emit_vertex(world_pos, clip_pos):
    gl_Position = vec4(clip_pos, 1.0) # Or appropriate clip space setup
    # Pass world pos to FS
    # In GLSL: FragPos = world_pos; ProjectPos = clip_pos;
    pass
```

### Fragment Shader Culling
To remove artifacts caused by the expansion (extra voxels at corners), the fragment shader discards fragments outside the bounding box of the original triangle [S4, unofficial].
*   **Input:** `FragPos` (world space), `ProjectPos` (clip space), `BoundingBox` (min/max xy in clip space).
*   **Logic:**
    ```python
    if (ProjectPos.x < BoundingBox.x or ProjectPos.y < BoundingBox.y or 
        ProjectPos.x > BoundingBox.z or ProjectPos.y > BoundingBox.w):
        discard
    ```
    [S4, unofficial].

### MSAA Approximation Alternative
Instead of geometric dilatation, enable 8xMSAA on the render state [S2, unofficial].
1.  Bind an MSAA render target [S2, unofficial].
2.  Rasterize primitives. The pixel shader is invoked only once per pixel if any subsample is covered [S2, unofficial].
3.  This approximates conservative rasterization by covering most of the pixel area [S2, unofficial].
4.  Skip primitive-voxel intersection tests required in conventional rasterization to remove extra voxels [S2, unofficial].

### Parameters
*   **MSAA Sample Count:** Typically 8x is used [S2, unofficial].
*   **Half-Pixel Size:** Determined by the resolution of the render target/voxel grid [S4, unofficial].

### Edge Cases and Pitfalls
*   **Holes between subsamples:** MSAA method may miss primitives lying exactly between subsamples. Mitigated by ensuring the voxel grid is fine/high-resolution enough relative to primitive sizes [S2, unofficial].
*   **Back-facing artifacts:** Winding order must be consistent. If the normal's z-component is negative, swap vertices to ensure counter-clockwise winding [S4, unofficial].
*   **Degenerate Planes:** If the triangle plane is parallel to the view direction ($N_z = 0$), division by zero occurs in z-calculation. The code returns early [S4, unofficial].
*   **Memory Cost:** MSAA method requires binding an MSAA render target, increasing video memory usage [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Geometric dilatation adds $O(1)$ per-triangle overhead in the geometry shader for plane calculations. MSAA adds negligible CPU overhead but increases GPU memory bandwidth and VRAM usage due to multi-sampling [S2, unofficial].
*   **Memory Complexity:** MSAA requires additional VRAM for the MSAA render target [S2, unofficial]. Geometric dilatation requires minimal extra memory (uniforms and registers) [S4, unofficial].
*   **Performance:** MSAA voxelization is considered reasonable in most cases, particularly on Maxwell or higher GPU architectures where official conservative rasterization functions are available [S2, unofficial]. Geometric dilatation is slower than hardware MSAA approximation but more accurate than conventional rasterization [S3, unofficial].
*   **Strengths:** Prevents holes in voxelized geometry for thin surfaces or small triangles [S1, S3, unofficial]. MSAA approach produces no redundant voxels compared to edge extension [S2, unofficial].
*   **Limitations:** MSAA is not genuine conservative rasterization and may have holes if resolution is insufficient [S2, unofficial]. Geometric dilatation is more complex to implement and slower than hardware MSAA [S3, unofficial]. Hardware conservative rasterization is limited to NVIDIA GPUs [S3, unofficial].
*   **When to use:** Use when voxel grid resolution is comparable to or coarser than triangle sizes, or when hole-free voxelization is critical. Use MSAA on modern NVIDIA GPUs; use Geometric Dilatation on other hardware or when deterministic results are required.

## Background
*   **Origin:** The geometric dilatation technique is attributed to Hertel et al. from the paper "A hybrid gpu rendering pipeline for alias-free hard shadows" (Eurographics 2009) [S1, unofficial].
*   **History:** Conservative rasterization was introduced to solve aliasing and hole artifacts in shadow mapping and voxelization [S1, unofficial]. The MSAA approximation method was popularized in GPU voxelization tutorials as a simpler alternative to explicit edge expansion [S2, unofficial].
*   **Usage:** Used in GPU-based voxelization pipelines for ray tracing, light injection, and opacity voxelization [S3, unofficial].

## Sources
[S1] A hybrid gpu rendering pipline for alias-free hard shadows, http://wwwcg.in.tum.de/research/research/publications/2009/a-hybrid-gpu-rendering-pipeline-for-alias-free-hard-shadows.html, fetched 2026-10-26 (unofficial)
[S2] The Basics of GPU Voxelization, https://www.cnblogs.com/hustztz/p/7575729.html, fetched 2026-10-27 (unofficial)
[S3] Voxel Cone Tracing, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S4] 体素化Voxelization：基于GPU的三维体素化, https://yangwc.com/2019/06/11/Voxelization/, fetched 2026-10-30 (unofficial)
