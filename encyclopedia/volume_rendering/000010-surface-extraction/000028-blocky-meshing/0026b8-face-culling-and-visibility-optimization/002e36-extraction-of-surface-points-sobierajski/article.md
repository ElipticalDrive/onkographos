# Extraction of Surface Points (Sobierajski)
Also called: Sobierajski's method, Visible boundary voxel extraction

## Overview
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It sets itself apart from sibling methods like [Face Culling](8735-face-culling.md) by operating in object space to generate a pre-filtered list of surface points based on viewing directions, rather than determining face visibility per voxel during mesh generation. This approach reduces the rendering load by culling back-facing voxels and projecting them to avoid holes, serving as a direct volume rendering acceleration technique [S1, unofficial].

## How it works
**Inputs:** Volume data (density values), set of viewing directions (typically six orthographic views along coordinate axes), lighting parameters, cutting plane definitions.
**Outputs:** A list of unique boundary voxels with attributes (coordinates, normal vectors) passed to the rendering engine.

**Data Structures:**
*   `BoundaryVoxelList`: A dynamic list or hash set storing extracted boundary voxels.
*   `VoxelAttributes`: For each voxel, stores 3D coordinates $(x, y, z)$ and normal vector $\mathbf{n}$.
*   `VisibilityGrid`: Implicitly handled via 2D boundary tracing on slices.

**Steps:**
1.  **Visibility Determination:** For each of the specified viewing directions (e.g., $\pm x, \pm y, \pm z$), determine which boundary voxels are potentially visible.
2.  **2D Boundary Tracing:** For each coordinate axis, perform visibility calculations using a 2D boundary tracing algorithm on slices perpendicular to that axis [S1, unofficial]. This identifies the outermost boundary voxels along the viewing ray.
3.  **List Generation and Deduplication:** Collect all identified boundary voxels into a single list. Remove duplicate elements (voxels visible from multiple directions) to ensure each surface point is processed once [S1, unofficial].
4.  **Attribute Assignment:** For each unique voxel in the list, compute and store necessary attributes, including its 3D coordinates and the surface normal vector $\mathbf{n}$ [S1, unofficial].
5.  **Voxel Culling (Backface Removal):** For a specific viewing direction, reduce the voxel list size by culling back-facing voxels. Calculate the dot product between the surface normal $\mathbf{n}$ and the viewing direction vector $\mathbf{v}$. If $\mathbf{n} \cdot \mathbf{v} > 0$, the voxel is a back face and is removed from the rendering set [S1, unofficial].
6.  **Projection and Hole Avoidance:** Project each remaining voxel onto the image plane. To avoid holes caused by adjacent voxels mapping to non-adjacent pixels, project each voxel onto several pixels depending on the viewing direction and voxel size [S1, unofficial].
7.  **Rendering:** Pass the set of surface points to the rendering engine. Shade the projected boundary surface points according to lighting conditions. If cutting planes are used, render voxels intersected by the cutting plane by projecting their original density values onto the image plane [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_visible_boundary_voxels(volume, viewing_directions, lighting, cutting_planes):
    """
    Extracts visible boundary voxels using Sobierajski's method.
    volume: 3D array of density values
    viewing_directions: list of 3D vectors (e.g., [1,0,0], [-1,0,0], ...)
    lighting: lighting parameters for shading
    cutting_planes: list of plane definitions
    """
    all_boundary_voxels = set() # Use set to handle deduplication
    
    # Step 1 & 2: Determine visibility and trace boundaries
    for direction in viewing_directions:
        # Perform 2D boundary tracing on slices perpendicular to direction
        # This identifies boundary voxels visible from this direction
        visible_voxels = trace_boundary_2d(volume, direction)
        all_boundary_voxels.update(visible_voxels)
        
    # Step 3 & 4: Deduplicate and assign attributes
    surface_points = []
    for voxel in all_boundary_voxels:
        coords = voxel.position
        normal = compute_normal(volume, coords) # Compute surface normal
        surface_points.append({
            'coords': coords,
            'normal': normal,
            'density': volume[coords]
        })
        
    # Step 5: Voxel Culling (Backface Removal) for each viewing direction
    # Note: In a full pipeline, this might be done per-view or globally.
    # Here we assume global culling based on average or specific view.
    # For simplicity, we cull based on a representative viewing direction or 
    # the final view direction.
    final_view_direction = viewing_directions[0] # Example: use first direction
    rendered_points = []
    for point in surface_points:
        dot_product = dot(point['normal'], final_view_direction)
        if dot_product <= 0: # Front-facing or tangent
            rendered_points.append(point)
            
    # Step 6 & 7: Projection and Rendering
    image = create_image_plane()
    for point in rendered_points:
        # Project voxel to multiple pixels to avoid holes
        projected_pixels = project_onto_pixels(point['coords'], point['normal'], final_view_direction)
        for pixel in projected_pixels:
            shade_pixel(pixel, point['density'], lighting)
            
    # Handle cutting planes
    for plane in cutting_planes:
        intersecting_voxels = get_intersecting_voxels(volume, plane)
        for voxel in intersecting_voxels:
            project_density_to_image(voxel, plane, image)
            
    return image
```

**Edge Cases and Pitfalls:**
*   **Holes:** Adjacent voxels may map to non-adjacent pixels, creating visual holes. Mitigated by projecting each voxel onto several pixels [S1, unofficial].
*   **Duplicate Voxels:** A voxel visible from multiple directions will be detected multiple times. Deduplication is essential [S1, unofficial].
*   **Backface Culling:** Incorrect normal calculation or viewing direction alignment can lead to incorrect culling. The condition $\mathbf{n} \cdot \mathbf{v} > 0$ defines a back face [S1, unofficial].

## Subtopics

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It differs from the main method by extracting only a subset of boundary voxels rather than all visible ones. The extracted surface samples are converted to geometrical primitives like crosslines perpendicular to the surface normal and projected onto the image plane [S1, unofficial]. Choose this when rendering speed is prioritized over geometric completeness for preview purposes.

## Operating characteristics
*   **Time Complexity:** Depends on the number of voxels and viewing directions. 2D boundary tracing is efficient per slice. Deduplication and culling add overhead. Overall, faster than full volume rendering for display purposes [S1, unofficial].
*   **Memory Complexity:** Stores a list of boundary voxels, which is significantly smaller than the full volume data [S1, unofficial].
*   **Strengths:** Fast display method; direct volume rendering approach allows easy implementation of cutting planes; reduces rendering load by culling back faces and duplicates [S1, unofficial].
*   **Limitations:** Approximate visibility (six orthographic views may not capture all visible voxels); holes can appear if not projected onto multiple pixels; less geometrically precise than mesh-based methods [S1, unofficial].
*   **When to Use:** Medical imaging systems requiring fast previewing and cutting operations; scenarios where rendering speed is critical and approximate surface representation is acceptable [S1, unofficial].
*   **When Not to Use:** When high-fidelity geometric meshes are required; when lighting and shading depend on precise surface normals for every pixel.

## Background
Proposed by Sobierajski [116] as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is cited in the context of medical imaging systems where cutting operations and simultaneous rendering of shaded isosurfaces and cross-sectional slices are important [S1, unofficial]. The method is part of a broader category of hybrid acceleration methods, though it is distinct from shear-warp factorization [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
