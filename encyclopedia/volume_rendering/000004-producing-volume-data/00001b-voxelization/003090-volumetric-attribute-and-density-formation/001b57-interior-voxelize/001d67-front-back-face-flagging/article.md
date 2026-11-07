# Front/Back Face Flagging

Also called: Voxel Surface Orientation

## Overview
Front/Back Face Flagging assigns a boolean orientation flag to voxels intersected by triangle surfaces, indicating whether the triangle is front-facing relative to a specified filling direction [S1]. It distinguishes interior from exterior voxels during mesh filling by tracking surface traversal direction [S1].

## How it works
**Inputs:**
- A triangular mesh defined by vertices $V = \{v_0, v_1, \dots\}$ [S1].
- A target voxel grid with resolution $R_x \times R_y \times R_z$ and bounds [S1].
- A filling direction vector $\vec{d}$ (typically $(0, 0, 1)$) [S1].

**Outputs:**
- A voxel array where each voxel $V_{ijk}$ contains:
  - `fill`: Integer or boolean flag indicating if the voxel intersects the mesh surface [S1].
  - `front`: Boolean flag indicating if the intersecting triangle is front-facing relative to $\vec{d}$ [S1].
  - `position`: 3 floats representing the voxel center [S1].

**Data Structures:**
The voxel is represented as a struct or object with fields:
- `fill`: 1 bit (or integer) [S1].
- `front`: 1 bit (or boolean) [S1].
- `position`: 3 floats [S1].

**Steps:**
1. **Bounding Box Culling:** For each triangle $T$ with vertices $a, b, c$, compute its Axis-Aligned Bounding Box (AABB) [S1]. Iterate through all voxels whose AABBs intersect $T$'s AABB [S1].
2. **Triangle Normal Calculation:** Compute the unnormalized normal $\vec{n}_{raw}$ using the cross product of two edge vectors:
   $$ \vec{n}_{raw} = (b - a) \times (c - a) $$
   Normalize to get $\vec{n}$:
   $$ \vec{n} = \frac{\vec{n}_{raw}}{\|\vec{n}_{raw}\|} $$ [S1]
3. **Front/Back Determination:** Determine if the triangle is front-facing relative to the filling direction $\vec{d}$ [S1]. A triangle is considered front-facing if the dot product of its normal and the filling direction is less than or equal to zero:
   $$ \text{is\_front} = (\vec{n} \cdot \vec{d}) \le 0 $$ [S1]
4. **Voxel Intersection Check:** For each candidate voxel, check if the triangle intersects the voxel's AABB using a plane-AABB intersection test [S1]. Define a plane $P$ with normal $\vec{n}$ and distance $D = \vec{n} \cdot a$ [S1]. Let $C$ be the voxel center and $\Delta x, \Delta y, \Delta z$ be the voxel extents from the center [S1]. The projected radius is $r = |n_x| \Delta x + |n_y| \Delta y + |n_z| \Delta z$ [S1]. The signed distance from the plane to the center is $s = \vec{n} \cdot C - D$ [S1]. Intersection occurs if $|s| \le r$ [S1].
5. **Flag Update:** If the voxel is intersected:
   - If `voxel.fill` was previously 0 (unfilled), set `voxel.front = is_front` [S1].
   - If `voxel.fill` was already 1 (already intersected by another triangle), update the front flag to prioritize back-facing triangles using a logical AND: `voxel.front = voxel.front & is_front` [S1].
   - Set `voxel.fill = 1` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def front_back_face_flagging(mesh_triangles, voxel_grid, fill_direction):
    """
    mesh_triangles: List of Triangle objects, each with vertices a, b, c.
    voxel_grid: 3D array of Voxel objects, initialized to fill=0, front=False.
    fill_direction: Vector3 (e.g., (0, 0, 1)).
    """
    for tri in mesh_triangles:
        # 1. Calculate Normal
        edge1 = tri.b - tri.a
        edge2 = tri.c - tri.a
        normal_raw = cross(edge1, edge2)
        normal = normalize(normal_raw)
        
        # 2. Determine Front/Back
        is_front = dot(normal, fill_direction) <= 0.0
        
        # 3. Get Triangle AABB bounds
        min_x = min(tri.a.x, tri.b.x, tri.c.x)
        max_x = max(tri.a.x, tri.b.x, tri.c.x)
        min_y = min(tri.a.y, tri.b.y, tri.c.y)
        max_y = max(tri.a.y, tri.b.y, tri.c.y)
        min_z = min(tri.a.z, tri.b.z, tri.c.z)
        max_z = max(tri.a.z, tri.b.z, tri.c.z)
        
        # 4. Iterate over intersecting voxels
        for x in range(voxel_grid.x_start(min_x), voxel_grid.x_end(max_x)):
            for y in range(voxel_grid.y_start(min_y), voxel_grid.y_end(max_y)):
                for z in range(voxel_grid.z_start(min_z), voxel_grid.z_end(max_z)):
                    voxel = voxel_grid[x, y, z]
                    
                    # 5. Check Triangle-Voxel Intersection (Plane-AABB)
                    center = voxel.aabb.center
                    extents = voxel.aabb.max - center
                    r = abs(normal.x) * extents.x + abs(normal.y) * extents.y + abs(normal.z) * extents.z
                    s = dot(normal, center) - dot(normal, tri.a)
                    
                    if abs(s) <= r:
                        # 6. Flag Update
                        if (voxel.fill & 1) == 0:
                            voxel.front = is_front
                        else:
                            voxel.front = voxel.front and is_front
                        
                        voxel.fill = 1
                        voxel.position = center
                        voxel_grid[x, y, z] = voxel
```

**Parameters:**
- `fill_direction`: The direction from which the mesh is considered to be "filled" [S1]. Typical value: $(0, 0, 1)$ for Z-axis filling [S1].

**Edge Cases and Pitfalls:**
- **Degenerate Triangles:** Triangles with zero area (collinear vertices) produce zero-length normals [G]. The normalization step must handle this, typically by skipping such triangles or treating them as non-intersecting [G].
- **Precision Errors:** Floating-point inaccuracies in dot product calculations may cause boundary cases where $\vec{n} \cdot \vec{d} \approx 0$ [G]. Using a small epsilon for the comparison $(\vec{n} \cdot \vec{d}) \le \epsilon$ can help stabilize results [G].
- **Non-Watertight Meshes:** This method does not inherently guarantee correct interior detection for open meshes [G]. It relies on the subsequent filling algorithm to use the flags correctly [G]. If a voxel is intersected by multiple triangles, the AND logic ensures that if *any* intersecting triangle is back-facing, the flag becomes back-facing [G]. This assumes that back-facing triangles represent the "inner" boundary relative to the fill direction [G].

## Subtopics

### Inside-Outside Voxelization
Determines whether a grid cell is inside or outside a solid obstacle using stencil buffer operations, inspired by stencil shadow volumes [S2, unofficial]. It renders the input triangle mesh into each slice of a 3D texture using an orthogonal projection [S2, unofficial]. A stencil buffer is initialized to zero, and stencil operations increment for back faces and decrement for front faces (with wrapping) [S2, unofficial]. Interior cells receive a nonzero stencil value [S2, unofficial]. This method depends on having one back face for every front face, making it best suited to watertight closed meshes [S2, unofficial]. It is often used in conjunction with velocity voxelization, where cells interior but next to the boundary are tagged by the velocity algorithm rather than this one [S2, unofficial]. Parameters include setting the far clip plane to infinity and the near clip plane to match the depth of the current slice [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Simple to implement; explicitly tracks orientation relative to a global vector, allowing subsequent filling algorithms to resolve ambiguity by prioritizing back-facing surfaces [S1].
- **Limitations:** Requires explicit intersection checks for each triangle-voxel pair; performance degrades with high-resolution grids and complex meshes; does not inherently resolve interior/exterior for open meshes without a subsequent filling pass [G].
- **When to Use:** When directional filling is required, or when dealing with meshes that may not be perfectly watertight but have a clear "inside" relative to a specific direction [G].
- **When Not to Use:** For perfectly watertight meshes where stencil-based methods [Inside-Outside Voxelization](1067-stochastic-ray-casting-voxelization.md) are faster and more robust; for scenes requiring high-performance real-time voxelization where CPU-based intersection checks are a bottleneck [G].

## Background
The method originates from CPU-based voxelization implementations for Unity graphics programming, specifically described in the context of real-time voxelization techniques [S1]. It is used in applications requiring dynamic voxelization of meshes for effects like fluid simulation, soft body dynamics, or volumetric lighting, where the interior of a mesh must be determined efficiently [G]. The approach is part of a broader family of voxelization algorithms that seek to convert polygonal meshes into voxel grids for further processing in physics or rendering pipelines [G].

## Sources
[S1] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
