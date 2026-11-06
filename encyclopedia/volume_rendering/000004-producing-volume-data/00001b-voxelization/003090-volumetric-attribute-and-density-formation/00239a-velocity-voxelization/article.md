# Velocity Voxelization

Also called: Boundary Velocity Voxelization, Slice-based Velocity Voxelization

## Overview
Velocity Voxelization computes the instantaneous velocity of an obstacle at each grid cell containing part of the obstacle's boundary, producing a velocity field for fluid simulation [S1, unofficial]. It differs from standard occupancy voxelization by explicitly interpolating vertex velocities across the intersection segments of mesh triangles and grid slices, outputting thickened quads to populate the velocity attributes of the volume [S1, unofficial]. It is best suited for watertight closed meshes to ensure consistent front/back face handling [S1, unofficial].

## How it works
**Inputs:**
- A watertight mesh defined by vertices $\mathbf{p}$.
- Vertex positions from the previous frame $\mathbf{p}^{n-1}$ and current frame $\mathbf{p}^n$.
- Time step $\Delta t$.
- A 3D grid defined by origin, resolution, and voxel size.
- Orthographic camera parameters aligned with the slicing axis.

**Outputs:**
- A 3D texture (or voxel grid) where boundary cells contain interpolated velocity vectors $\mathbf{v}$.

**Data Structures:**
- **Vertex Buffer:** Stores $\mathbf{p}^{n-1}$ and $\mathbf{p}^n$ for each vertex.
- **3D Texture:** Stores velocity vectors (e.g., `vec3`) at each grid index.

**Steps:**

1.  **Compute Per-Vertex Velocity:**
    Calculate the instantaneous velocity $\mathbf{v}_i$ for each vertex $i$ using the forward difference approximation [S1, unofficial]:
    $$ \mathbf{v}_i = \frac{\mathbf{p}_i^n - \mathbf{p}_i^{n-1}}{\Delta t} $$
    [G] Note: The source text contains a typo in the formula description ($\mathbf{p}_i^{n+1}$) but defines inputs as previous and current frames. The standard forward difference uses current minus previous.

2.  **Iterate Grid Slices:**
    Render the mesh once for each slice of the grid along the chosen axis (e.g., Z-axis). For each slice $k$, set the orthographic camera's near plane to the slice's position [S1, unofficial].

3.  **Compute Triangle-Slice Intersection:**
    For each triangle in the mesh, intersect its three edges with the current slice plane.
    - If exactly two edge-slice intersections are found, they define a segment.
    - If the intersection is a point, triangle, or empty, no velocity quad is generated for this triangle in this slice [S1, unofficial].

4.  **Interpolate Velocity at Segment Endpoints:**
    Let the two intersection points be $\mathbf{q}_1$ and $\mathbf{q}_2$. These points lie on specific triangle edges. Interpolate the velocity at $\mathbf{q}_1$ and $\mathbf{q}_2$ using the barycentric coordinates of the intersection points along the respective triangle edges and the precomputed vertex velocities $\mathbf{v}_i$ [S1, unofficial].

5.  **Generate Thickened Quad:**
    Create a quad representing the segment $\mathbf{q}_1 \mathbf{q}_2$ thickened to cover the grid cells.
    - **Offset Distance $w$:** Equal to the diagonal length of one texel in the slice plane. If voxel size is $s$, then $w = s\sqrt{2}$ [S1, unofficial].
    - **Offset Direction:** The projection of the triangle's normal $\mathbf{n}$ onto the slice plane. Let $\mathbf{u}$ be the unit vector in this direction.
    - **Quad Vertices:**
      $$ \mathbf{c}_1 = \mathbf{q}_1 + w \mathbf{u} $$
      $$ \mathbf{c}_2 = \mathbf{q}_1 - w \mathbf{u} $$
      $$ \mathbf{c}_3 = \mathbf{q}_2 + w \mathbf{u} $$
      $$ \mathbf{c}_4 = \mathbf{q}_2 - w \mathbf{u} $$
    - Assign the interpolated velocities $\mathbf{v}(\mathbf{q}_1)$ to $\mathbf{c}_1, \mathbf{c}_2$ and $\mathbf{v}(\mathbf{q}_2)$ to $\mathbf{c}_3, \mathbf{c}_4$ [S1, unofficial].

6.  **Rasterize Quad:**
    Output the quad as a two-triangle strip (since geometry shaders cannot output quads directly). Rasterize into the 3D texture using additive blending or logic operations to accumulate velocity values [S1, unofficial]. Linear interpolation across the quad distributes the velocity values to the covered grid cells [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def velocity_voxelization(mesh, p_prev, p_curr, dt, grid, slice_axis):
    # Precompute vertex velocities
    v_vertices = (p_curr - p_prev) / dt
    
    # Initialize velocity volume
    vol = zeros(grid.resolution, dtype=vec3)
    
    # Determine slice planes
    slice_positions = linspace(grid.origin[slice_axis], 
                               grid.origin[slice_axis] + grid.extent[slice_axis], 
                               grid.resolution[slice_axis])
    
    # Offset vector magnitude (diagonal of one texel)
    voxel_size = grid.voxel_size
    w = voxel_size * sqrt(2)
    
    for z_idx, z_pos in enumerate(slice_positions):
        # For each triangle in mesh
        for tri in mesh.triangles:
            # Get vertices and their velocities
            v0, v1, v2 = tri.vertices
            sv0, sv1, sv2 = v_vertices[v0], v_vertices[v1], v_vertices[v2]
            
            # Intersect edges with slice plane z = z_pos
            # Edge 0-1
            p_a, t_a = intersect_edge_plane(v0, v1, z_pos)
            # Edge 1-2
            p_b, t_b = intersect_edge_plane(v1, v2, z_pos)
            # Edge 2-0
            p_c, t_c = intersect_edge_plane(v2, v0, z_pos)
            
            # Collect valid intersections
            intersections = []
            if p_a is not None: intersections.append((p_a, sv0, sv1, t_a))
            if p_b is not None: intersections.append((p_b, sv1, sv2, t_b))
            if p_c is not None: intersections.append((p_c, sv2, sv0, t_c))
            
            # Must have exactly two intersections to form a segment
            if len(intersections) != 2:
                continue
                
            # Sort intersections to ensure consistent ordering (optional but good for quad winding)
            intersections.sort(key=lambda x: x[0][slice_axis])
            
            q1, sv_start1, sv_end1, t1 = intersections[0]
            q2, sv_start2, sv_end2, t2 = intersections[1]
            
            # Interpolate velocity at q1 and q2
            v1_val = sv_start1 + t1 * (sv_end1 - sv_start1)
            v2_val = sv_start2 + t2 * (sv_end2 - sv_start2)
            
            # Compute offset direction: projection of triangle normal onto slice plane
            tri_normal = cross(v1 - v0, v2 - v0)
            tri_normal = normalize(tri_normal)
            
            # Project normal onto slice plane (remove component along slice_axis)
            offset_dir = tri_normal - dot(tri_normal, unit_vector(slice_axis)) * unit_vector(slice_axis)
            if length(offset_dir) < 1e-6:
                # Degenerate case: normal is parallel to slice axis
                # Use arbitrary perpendicular vector in slice plane
                offset_dir = arbitrary_perpendicular(tri_normal, slice_axis)
            offset_dir = normalize(offset_dir)
            
            # Generate quad vertices
            c1 = q1 + w * offset_dir
            c2 = q1 - w * offset_dir
            c3 = q2 + w * offset_dir
            c4 = q2 - w * offset_dir
            
            # Rasterize quad into volume at z_idx
            # Using additive blending for velocity accumulation
            rasterize_quad_into_volume(vol, c1, c2, c3, c4, v1_val, v2_val, z_idx)

    return vol
```

## Subtopics

### [Geometry Shader for Velocity Voxelization](3083-geometry-shader-for-velocity-voxelization.md)
Computes obstacle velocity at each grid cell containing part of the obstacle's boundary by projecting 3D triangle intersections onto 2D slice planes. It differs from standard voxelization by outputting a 1-voxel wide quadrilateral for each intersecting triangle edge, explicitly encoding the interpolated velocity vector at the intersection point. This allows the subsequent rasterization or accumulation step to directly populate velocity fields without post-processing interpolation.

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of triangles and $S$ is the number of slices (grid resolution along the slicing axis). Each triangle is tested against each slice plane.
- **Memory Complexity:** $O(V)$, where $V$ is the number of voxels in the 3D grid, to store the velocity field.
- **Strengths:** Directly computes boundary velocities; efficient on GPU using geometry shaders; handles moving meshes naturally via vertex buffer updates.
- **Limitations:** Requires watertight meshes for consistent results; performance degrades with high grid resolution; sensitive to degenerate triangles or normals parallel to the slicing axis.
- **When to use:** Fluid simulation where boundary velocity is required; real-time applications where GPU acceleration is available.
- **When not to use:** Non-watertight meshes; static scenes where velocity is zero; offline rendering where higher accuracy methods might be preferred.

## Background
Originated in the context of real-time 3D fluid simulation, specifically for coupling solid obstacle motion with fluid dynamics. Described in "Real-Time Simulation and Rendering of 3D Fluids" [S1, unofficial]. The method leverages GPU geometry shaders to efficiently generate velocity data for boundary voxels.

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
