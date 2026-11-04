# Conservative Rasterization

Also called: Conservative Rasterization via Geometric Dilatation, Conservative Rasterization via MSAA, GPU Voxelization Conservative Rasterization, Triangle Expansion for Voxelization

## Overview
This method prevents holes in voxelized geometry caused by triangles smaller than the voxel grid resolution by rasterizing all fragments that "touch" the triangle [S1, S2, unofficial]. It achieves this by analytically expanding the triangle in clip space before rasterization, ensuring that any voxel potentially intersected by the triangle is covered [S2, unofficial]. Unlike hardware conservative rasterization (limited to NVIDIA) or Multi-Sample Anti-Aliasing (MSAA) activation [S1, unofficial], this geometric approach is hardware-agnostic but computationally more intensive, requiring a geometry shader for the expansion logic [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle defined by three vertices in clip space (`pos[0..2]`).
*   **Uniforms:**
    *   `halfPixel[3]`: `vec2` array representing half the pixel size in clip space for each of the three possible projection axes.
    *   `viewProject[3]`: Array of 3 `mat4` matrices for view-projection transformations.
    *   `viewProjectInverse[3]`: Array of 3 `mat4` matrices for inverse view-projection transformations.
*   **Outputs:**
    *   Fragment Shader: `FragPos` (world space position), `ProjectPos` (clip space position of the expanded vertex), `BoundingBox` (min/max xy in clip space of the original triangle, enlarged).
    *   Compute Buffer: Atomic counter updates for voxel occupancy.

**Data Structures:**
*   **Edge Planes:** `vec3 edgePlanes[3]`. Each vector $(A, B, C)$ represents the homogeneous plane equation $Ax + By + Cz = 0$ passing through the origin for each triangle edge.
*   **Triangle Plane:** `vec4 trianglePlane`. Represents the plane equation $Ax + By + Cz + D = 0$ containing the original triangle.
*   **Intersection Lines:** `vec3 intersection[3]`. Direction vectors for the lines formed by the intersection of offset edge planes.

**Numbered Steps:**

1.  **Select Projection Axis:** Determine the dominant axis of the triangle's face normal to select the appropriate projection matrix and half-pixel size. Calculate the normal $N = (pos[1] - pos[0]) \times (pos[2] - pos[0])$. Let $n_x, n_y, n_z$ be the absolute components.
    *   If $n_x > n_y$ and $n_x > n_z$, index $= 0$ (project onto YZ plane).
    *   Else if $n_y > n_x$ and $n_y > n_z$, index $= 1$ (project onto XZ plane).
    *   Else, index $= 2$ (project onto XY plane). [S2, unofficial]

2.  **Calculate Triangle Plane:** Compute the normal $N$ of the triangle in world space (or the space defined by the projection). Normalize it. Calculate $D = -P_0 \cdot N$. [S2, unofficial]
    $$ \text{trianglePlane} = (N_x, N_y, N_z, D) $$
    *Note: The source code uses `pos[0].xyz` which implies the vertices are in world space or the space where the plane equation is valid for the final voxelization step.* [S2, unofficial]

3.  **Fix Winding Order:** To prevent artifacts with back-facing triangles, ensure counter-clockwise winding relative to the view direction. Check the dot product of the triangle normal with the view direction $(0,0,1)$. If $N \cdot (0,0,1) < 0$, swap vertices 1 and 2. [S2, unofficial]

4.  **Calculate Homogeneous Edge Planes:** For each edge $i$ (connecting vertex $i$ to $i+1$, wrapping around), compute the plane passing through the origin using the cross product of the homogeneous vertex coordinates (xyw components).
    $$ \text{edgePlanes}[0] = (pos[0].xyw - pos[2].xyw) \times pos[2].xyw $$
    $$ \text{edgePlanes}[1] = (pos[1].xyw - pos[0].xyw) \times pos[0].xyw $$
    $$ \text{edgePlanes}[2] = (pos[2].xyw - pos[1].xyw) \times pos[1].xyw $$
    *Note: In orthographic projection, $w=1$, so this effectively computes the 2D line normal in clip space.* [S2, unofficial]

5.  **Dilate Edge Planes:** Offset the homogeneous planes to expand the triangle. Subtract the projection of the half-pixel size onto the plane's normal from the plane's z-component (distance).
    $$ \text{edgePlanes}[i].z \leftarrow \text{edgePlanes}[i].z - \text{dot}(\text{halfPixel}[\text{index}], |\text{edgePlanes}[i].xy|) $$
    *Note: $|\text{edgePlanes}[i].xy|$ denotes the absolute value of the x and y components.* [S2, unofficial]

6.  **Calculate Intersection Lines:** Compute the direction vector of the intersection line between two offset planes. The direction is the cross product of their normals. Normalize by dividing by the z-component.
    $$ \text{intersection}[0] = \text{edgePlanes}[0] \times \text{edgePlanes}[1] $$
    $$ \text{intersection}[1] = \text{edgePlanes}[1] \times \text{edgePlanes}[2] $$
    $$ \text{intersection}[2] = \text{edgePlanes}[2] \times \text{edgePlanes}[0] $$
    $$ \text{intersection}[i].x \leftarrow \text{intersection}[i].x / \text{intersection}[i].z $$
    $$ \text{intersection}[i].y \leftarrow \text{intersection}[i].y / \text{intersection}[i].z $$
    *Note: The z-component of the intersection vector is effectively normalized to 1.* [S2, unofficial]

7.  **Compute Dilated Vertices:** Calculate the new vertices of the dilated triangle. The x and y coordinates are taken from the intersection line vectors. The z coordinate is solved using the original triangle's plane equation:
    $$ z_i = -( \text{intersection}[i].x \cdot \text{trianglePlane}.x + \text{intersection}[i].y \cdot \text{trianglePlane}.y + \text{trianglePlane}.w ) / \text{trianglePlane}.z $$
    $$ \text{pos}[i].xyz = (\text{intersection}[i].x, \text{intersection}[i].y, z_i) $$
    *Pitfall: If $\text{trianglePlane}.z == 0$, the triangle is parallel to the view direction, leading to division by zero. The algorithm should return early or handle this degenerate case.* [S2, unofficial]

8.  **Transform and Output:** Transform the dilated vertices back to world space using the inverse view-projection matrix. Pass the clip-space positions to the fragment shader.
    $$ \text{voxelPos} = \text{viewProjectInverse}[\text{index}] \times \text{pos}[i] $$
    $$ \text{FragPos} = \text{voxelPos}.xyz $$
    $$ \text{ProjectPos} = \text{pos}[i].xyz $$
    Emit the vertex. [S2, unofficial]

9.  **Fragment Shader Culling:** In the fragment shader, discard fragments outside the bounding box of the original triangle (enlarged by half-pixel) to remove artifacts caused by the expansion.
    $$ \text{if } (\text{ProjectPos}.x < \text{BoundingBox}.x \lor \text{ProjectPos}.y < \text{BoundingBox}.y \lor \text{ProjectPos}.x > \text{BoundingBox}.z \lor \text{ProjectPos}.y > \text{BoundingBox}.w) \text{ discard} $$
    *Note: `BoundingBox` is calculated in the geometry shader as the min/max xy of the original triangle, enlarged by `halfPixel`.* [S2, unofficial]

10. **Voxel Counting:** Map the world space position (`FragPos`) to voxel coordinates and atomically increment the count.
    $$ x = \lfloor (\text{FragPos}.x - \text{boxMin}.x) / \text{step} \rfloor $$
    $$ y = \lfloor (\text{FragPos}.y - \text{boxMin}.y) / \text{step} \rfloor $$
    $$ z = \lfloor (\text{FragPos}.z - \text{boxMin}.z) / \text{step} \rfloor $$
    $$ \text{index} = y \cdot (\text{resolution}.z \cdot \text{resolution}.x) + z \cdot \text{resolution}.x + x $$
    $$ \text{atomicAdd}(\text{cnts}[\text{index}], 1) $$ [S2, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Geometry Shader Logic (Simplified)

def select_projection_axis(pos):
    """Selects the dominant axis of the face normal."""
    p1 = pos[1].xyz - pos[0].xyz
    p2 = pos[2].xyz - pos[0].xyz
    face_normal = cross(p1, p2)
    n_dx = abs(face_normal.x)
    n_dy = abs(face_normal.y)
    n_dz = abs(face_normal.z)
    
    if n_dx > n_dy and n_dx > n_dz:
        return 0
    elif n_dy > n_dx and n_dy > n_dz:
        return 1
    else:
        return 2

def calc_aabb(pos, half_pixel):
    """Calculates the bounding box of the triangle in clip space, enlarged by half-pixel."""
    min_xy = min(pos[0].xy, min(pos[1].xy, pos[2].xy))
    max_xy = max(pos[0].xy, max(pos[1].xy, pos[2].xy))
    # Enlarge by half-pixel
    min_xy -= half_pixel
    max_xy += half_pixel
    return vec4(min_xy.x, min_xy.y, max_xy.x, max_xy.y)

def conservative_rasterization_expand(pos, half_pixel, view_project_inv, project_index):
    """
    Expands the triangle in clip space to ensure all touching voxels are covered.
    pos: Array of 3 vec4 (clip space vertices)
    half_pixel: vec2 (half pixel size in clip space)
    view_project_inv: mat4 (inverse view-projection matrix)
    project_index: int (0, 1, or 2)
    Returns: List of (FragPos, ProjectPos, BoundingBox) for each vertex
    """
    
    # 1. Calculate Triangle Plane in World Space (assuming pos.xyz is world space for plane calc)
    # Note: Source code uses pos[0].xyz for plane calculation, implying input might be world space 
    # or the geometry shader receives world space vertices. 
    # However, gl_Position is clip space. The source code mixes spaces. 
    # We follow the source: pos[0].xyz is used for plane normal.
    
    triangle_normal = normalize(cross(pos[1].xyz - pos[0].xyz, pos[2].xyz - pos[0].xyz))
    triangle_plane_w = -dot(pos[0].xyz, triangle_normal)
    
    # 2. Fix Winding Order
    if dot(triangle_normal, vec3(0.0, 0.0, 1.0)) < 0.0:
        temp = pos[2]
        pos[2] = pos[1]
        pos[1] = temp
        
    # Pitfall: Check for degenerate plane
    if abs(triangle_normal.z) < 1e-6:
        return [] # Return empty to skip processing
        
    # 3. Calculate Homogeneous Edge Planes (using clip space xyw)
    edge_planes = [vec3(0.0)] * 3
    edge_planes[0] = cross(pos[0].xyw - pos[2].xyw, pos[2].xyw)
    edge_planes[1] = cross(pos[1].xyw - pos[0].xyw, pos[0].xyw)
    edge_planes[2] = cross(pos[2].xyw - pos[1].xyw, pos[1].xyw)
    
    # 4. Dilate Edge Planes
    for i in range(3):
        # Subtract projection of half-pixel onto plane's normal (xy components)
        dilate_dist = dot(half_pixel, abs(vec2(edge_planes[i].x, edge_planes[i].y)))
        edge_planes[i].z -= dilate_dist
        
    # 5. Calculate Intersection Lines
    intersection = [vec3(0.0)] * 3
    intersection[0] = cross(edge_planes[0], edge_planes[1])
    intersection[1] = cross(edge_planes[1], edge_planes[2])
    intersection[2] = cross(edge_planes[2], edge_planes[0])
    
    # Normalize by z-component
    for i in range(3):
        if abs(intersection[i].z) > 1e-6:
            intersection[i].x /= intersection[i].z
            intersection[i].y /= intersection[i].z
        else:
            # Degenerate case, handle appropriately (e.g., keep as is or skip)
            pass
            
    # 6. Compute Dilated Vertices
    dilated_pos = [vec4(0.0)] * 3
    for i in range(3):
        # Solve for z using triangle plane equation: Ax + By + Cz + D = 0
        # z = -(Ax + By + D) / C
        z_val = -(intersection[i].x * triangle_normal.x + intersection[i].y * triangle_normal.y + triangle_plane_w) / triangle_normal.z
        dilated_pos[i] = vec4(intersection[i].x, intersection[i].y, z_val, 1.0)
        
    # 7. Transform to World Space and Prepare Outputs
    outputs = []
    bounding_box = calc_aabb(pos, half_pixel) # Use original pos for bounding box
    
    for i in range(3):
        # Transform dilated vertex to world space
        voxel_pos = view_project_inv * dilated_pos[i]
        frag_pos = voxel_pos.xyz
        project_pos = dilated_pos[i].xyz # Clip space position for culling
        
        outputs.append({
            'FragPos': frag_pos,
            'ProjectPos': project_pos,
            'BoundingBox': bounding_box
        })
        
    return outputs

# Fragment Shader Logic
def fragment_shader(FragPos, ProjectPos, BoundingBox, boxMin, step, resolution, cnts):
    """
    FragPos: vec3 (World space position)
    ProjectPos: vec3 (Clip space position)
    BoundingBox: vec4 (min_x, min_y, max_x, max_y in clip space)
    """
    
    # Culling
    if (ProjectPos.x < BoundingBox.x or 
        ProjectPos.y < BoundingBox.y or 
        ProjectPos.x > BoundingBox.z or 
        ProjectPos.y > BoundingBox.w):
        discard()
        
    # Voxel Index Calculation
    x = int((FragPos.x - boxMin.x) / step)
    y = int((FragPos.y - boxMin.y) / step)
    z = int((FragPos.z - boxMin.z) / step)
    
    index = y * (resolution.z * resolution.x) + z * resolution.x + x
    
    # Atomic Add
    atomic_add(cnts[index], 1)
    
    return vec4(0.0, 1.0, 1.0, 1.0) # Color output
```

## Operating characteristics
*   **Time Complexity:** The expansion logic is performed per triangle in the geometry shader. It involves a constant number of vector operations (cross products, dot products, normalizations). The fragment shader performs a constant number of comparisons and an atomic add.
*   **Memory Complexity:** Requires storage for the voxel count buffer (`int cnts[]`) and uniform matrices.
*   **Performance:** The geometric dilatation approach is slower than hardware conservative rasterization (if available) but faster than other software expansion techniques [S1, unofficial]. MSAA activation is described as "not too expensive" and producing few noticeable artifacts, but the geometric dilatation method is more explicit and controllable [S1, S2, unofficial].
*   **Strengths:** Hardware-agnostic (works on any GPU with geometry shaders), prevents holes effectively for small triangles, explicit control over the dilation amount.
*   **Limitations:** Requires a geometry shader, which can be a bottleneck on some hardware. Computationally more intensive than simple rasterization. Degenerate cases (triangle parallel to view) must be handled.
*   **When to use:** When hardware conservative rasterization is not available (e.g., non-NVIDIA GPUs) and MSAA is insufficient or undesirable. When precise control over voxel coverage is needed.
*   **When not to use:** When performance is critical and hardware conservative rasterization is available. When the geometry is large and the overhead of the geometry shader is prohibitive.

## Background
This method is a software implementation of conservative rasterization specifically tailored for GPU voxelization. It was popularized in the context of voxel cone tracing and real-time global illumination [S1, S2, unofficial]. The technique of geometric dilatation in clip space is detailed in "The Basics of GPU Voxelization" and related blog posts [S2, unofficial]. It addresses the limitation of traditional rasterization, which ignores fragments that do not strictly intersect the triangle's interior, leading to holes in the voxel grid for small or thin triangles [S1, unofficial].

## Sources
[S1] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S2] 体素化Voxelization：基于GPU的三维体素化 - YangWC's Blog, https://yangwc.com/2019/06/11/Voxelization/, fetched 2026-10-30 (unofficial)
