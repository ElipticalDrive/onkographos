# Extraction of Surface Points (Sobierajski)
Also called: Sobierajski's method, Visible boundary voxel extraction

## Overview
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It sets itself apart from sibling methods by using a 2D boundary tracing algorithm on axis-aligned slices to pre-filter voxels based on orthographic visibility, rather than performing per-face or per-voxel occlusion tests during the final render pass [S1, unofficial].

## How it works
**Inputs:** Volume data (density field); viewing directions (typically six orthographic views along $\pm x, \pm y, \pm z$) [S1, unofficial].
**Outputs:** A deduplicated list of surface points (boundary voxels) with attributes (coordinates, normal vectors) [S1, unofficial].

**Data Structures:**
*   **Boundary Voxel List:** A dynamic array or linked list storing extracted boundary voxels. Each entry contains:
    *   `coords`: 3D integer coordinates $(i, j, k)$ of the voxel.
    *   `normal`: 3D float vector $\mathbf{n}$ representing the surface normal at the boundary.
    *   `density`: Original density value (for cutting plane support) [S1, unofficial].

**Steps:**
1.  **Visibility Calculation:** For each of the $N$ viewing directions (typically 6 orthographic), perform a 2D boundary tracing algorithm on slices perpendicular to the corresponding coordinate axis [S1, unofficial].
    *   *Note:* The specific implementation of the 2D boundary tracing algorithm is not detailed in the sources [G]. It is assumed to identify voxels on the exterior boundary relative to the viewing direction.
2.  **Deduplication:** Merge the lists of boundary voxels from all viewing directions. Remove duplicate entries (voxels identified as boundary in multiple views) [S1, unofficial].
3.  **Attribute Assignment:** For each unique boundary voxel, compute and store the surface normal vector $\mathbf{n}$ and coordinates [S1, unofficial].
4.  **Backface Culling (Voxel Culling):** For a specific final viewing direction $\mathbf{v}$, iterate through the boundary voxel list. If the dot product of the surface normal $\mathbf{n}$ and the viewing direction $\mathbf{v}$ is positive, the voxel is a back face and is excluded from rendering [S1, unofficial]:
    $$ \mathbf{n} \cdot \mathbf{v} > 0 \implies \text{Cull} $$
    *   *Note:* This assumes $\mathbf{v}$ points from the camera to the scene, consistent with the parent article's orientation test [S1, unofficial].
5.  **Projection:** Project each remaining voxel onto the image plane. To avoid holes caused by adjacent voxels mapping to non-adjacent pixels, project each voxel onto multiple pixels depending on the viewing direction [S1, unofficial].
6.  **Shading:** Shade the projected boundary surface points according to lighting conditions [S1, unofficial].
7.  **Cutting Planes (Optional):** If cutting planes are used, render voxels intersected by the plane by projecting their original density values onto the image plane [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_visible_boundary_voxels(volume_data, viewing_directions):
    """
    Extracts visible boundary voxels using Sobierajski's method.
    
    Args:
        volume_data: 3D array of density values.
        viewing_directions: List of 3D vectors representing viewing directions.
        
    Returns:
        list of dicts: Each dict contains 'coords', 'normal', 'density'.
    """
    all_boundary_voxels = set()
    
    # Step 1 & 2: Visibility Calculation and Deduplication
    for direction in viewing_directions:
        # Perform 2D boundary tracing on slices perpendicular to direction
        # The specific tracing algorithm is not defined in sources [G]
        boundary_voxels = perform_2d_boundary_tracing(volume_data, direction)
        
        for voxel in boundary_voxels:
            # Store voxel key for deduplication (e.g., coordinates)
            all_boundary_voxels.add(voxel)
            
    # Step 3: Attribute Assignment
    surface_points = []
    for voxel in all_boundary_voxels:
        coords = voxel.coords
        normal = compute_surface_normal(volume_data, coords)
        density = volume_data[coords]
        surface_points.append({
            'coords': coords,
            'normal': normal,
            'density': density
        })
        
    return surface_points

def render_sobierajski(surface_points, final_view_direction, image_plane):
    """
    Renders the extracted surface points with backface culling and projection.
    
    Args:
        surface_points: List of surface point dicts from extract_visible_boundary_voxels.
        final_view_direction: 3D vector for the final camera view.
        image_plane: Target image buffer.
    """
    # Step 4: Backface Culling
    visible_points = []
    for point in surface_points:
        # Check if back face: n . v > 0 implies cull
        if dot(point['normal'], final_view_direction) > 0:
            continue
        visible_points.append(point)
        
    # Step 5: Projection
    for point in visible_points:
        # Project voxel onto multiple pixels to avoid holes
        # The exact projection mapping depends on viewing direction [G]
        project_onto_image(point['coords'], point['normal'], final_view_direction, image_plane)
        
    # Step 6: Shading
    apply_lighting(image_plane, final_view_direction)
```

## Subtopics

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It differs from the main method by sampling only a subset of boundary voxels rather than all visible ones, converting them to geometrical primitives (like crosslines perpendicular to the surface normal) for projection [S1, unofficial].

### Trimmed Voxel Lists
A surface rendering algorithm that uses 3D points as geometric primitives by projecting exactly one point per visible surface cell to ensure image coverage without subdivision [S2, unofficial]. It differs from Dividing Cubes by avoiding recursive subdivision; it achieves pixel coverage by projecting a single point per cell that can span multiple pixels (up to three) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the 2D boundary tracing on slices and the final projection. The number of voxels to be projected is reduced by backface culling [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of visible boundary voxels, which is typically much smaller than the total voxel count [S1, unofficial].
*   **Strengths:** Fast display method; efficient visibility calculation using 2D tracing; supports cutting planes [S1, unofficial].
*   **Limitations:** Holes can appear if adjacent voxels map to non-adjacent pixels (mitigated by multi-pixel projection); approximation of surface depends on the number of viewing directions [S1, unofficial].
*   **When to use:** When fast previewing or direct rendering of boundary surfaces is required, and the volume data has clear boundaries [S1, unofficial].

## Background
Proposed by Sobierajski [116] as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is cited in the context of medical imaging systems where cutting operations are important [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
