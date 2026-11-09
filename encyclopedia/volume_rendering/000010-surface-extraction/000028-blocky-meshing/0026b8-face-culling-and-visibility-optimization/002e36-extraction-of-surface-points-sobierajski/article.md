# Extraction of Surface Points (Sobierajski)

Also called: Sobierajski's method, Visible boundary voxel extraction

## Overview
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It sets itself apart from sibling methods like [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by operating on a per-voxel basis using global viewing directions rather than local neighbor checks, and from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by relying on 2D boundary tracing rather than coverage masks.

## How it works
**Inputs:** 3D volume data grid; set of viewing directions (typically six orthographic views along coordinate axes) [S1, unofficial].
**Outputs:** List of visible boundary voxels with attributes (coordinates, normal vectors) [S1, unofficial].

**Data Structures:**
*   `VolumeGrid`: 3D array of density/occupancy values.
*   `BoundaryList`: List of objects, each containing `position` (x, y, z) and `normal` (nx, ny, nz).

**Steps:**
1.  **Visibility Determination:** For each of the $N$ viewing directions $\vec{v}_i$, determine which boundary voxels are potentially visible. For the standard six orthographic views, this is performed efficiently using a 2D boundary tracing algorithm on slices perpendicular to each coordinate axis [S1, unofficial].
2.  **Extraction and Deduplication:** Collect all voxels identified as visible in any direction. Remove duplicate entries from the list [S1, unofficial].
3.  **Attribute Storage:** For each unique boundary voxel, compute and store necessary rendering attributes, specifically its 3D coordinates and surface normal vector [S1, unofficial].
4.  **Back-face Culling:** For a specific viewing direction $\vec{v}$, reduce the voxel list size. If the dot product of the surface normal $\vec{n}$ and the viewing direction $\vec{v}$ is positive, the voxel is a back face and is not rendered:
    $$ \vec{n} \cdot \vec{v} > 0 \implies \text{discard voxel} $$
    [S1, unofficial].
5.  **Projection:** Project each remaining voxel onto the image plane. To avoid holes caused by adjacent voxels mapping to non-adjacent pixels, project each voxel onto several pixels depending on the viewing direction [S1, unofficial].
6.  **Rendering:** Pass the set of surface points to the rendering engine. Shade the projected boundary surface points according to lighting conditions [S1, unofficial]. If cutting planes are used, voxels intersected by the plane are rendered by projecting their original density values onto the image plane [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_visible_boundary_voxels(volume_grid, viewing_directions):
    """
    Extracts visible boundary voxels using Sobierajski's method.
    
    Args:
        volume_grid: 3D array of density values.
        viewing_directions: List of 3D vectors representing camera/view directions.
        
    Returns:
        visible_voxels: List of dicts with 'pos' and 'normal'.
    """
    all_boundary_voxels = []
    
    # Step 1 & 2: Determine visibility and extract
    # For each viewing direction, perform 2D boundary tracing on slices
    for view_dir in viewing_directions:
        # Note: The specific 2D boundary tracing algorithm on slices 
        # perpendicular to coordinate axes is implied by the source [S1, unofficial].
        # Implementation detail: Iterate through slices, find boundary edges,
        # map back to 3D voxels.
        voxels_in_view = trace_boundary_slices(volume_grid, view_dir)
        all_boundary_voxels.extend(voxels_in_view)
        
    # Remove duplicates
    unique_voxels = remove_duplicates(all_boundary_voxels)
    
    # Step 3: Compute attributes
    visible_voxels = []
    for voxel in unique_voxels:
        pos = voxel.position
        normal = compute_surface_normal(volume_grid, pos)
        visible_voxels.append({'pos': pos, 'normal': normal})
        
    return visible_voxels

def render_sobierajski(visible_voxels, view_dir, image_plane):
    """
    Renders the extracted voxels with back-face culling and projection.
    """
    for voxel in visible_voxels:
        # Step 4: Back-face culling
        if dot(voxel['normal'], view_dir) > 0:
            continue # Discard back face
            
        # Step 5: Project onto several pixels to avoid holes
        projected_pixels = project_onto_pixels(voxel['pos'], view_dir, image_plane)
        
        # Step 6: Shade and render
        for pixel in projected_pixels:
            shade = apply_lighting(voxel['normal'], view_dir)
            image_plane.set_pixel(pixel.x, pixel.y, shade)
```

## Subtopics

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It extracts a subset of boundary voxels uniformly, converts them to geometrical primitives (e.g., crosslines perpendicular to the surface normal), and projects these primitives onto the image plane [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of viewing directions and the cost of 2D boundary tracing. The source notes that rendering speed is directly related to the length of the voxel list [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of visible boundary voxels stored in the list.
*   **Strengths:** Fast display method; supports flexible manipulation of extracted voxels (e.g., cutting planes) [S1, unofficial].
*   **Limitations:** Holes can appear if adjacent voxels map to non-adjacent pixels; requires multiple viewing directions for complete visibility coverage [S1, unofficial].
*   **When to use:** When fast previewing or direct rendering of boundary surfaces is required, and the volume data is suitable for orthographic projection [S1, unofficial].

## Background
Proposed by Sobierajski [116] as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is cited in the context of medical imaging systems where cutting operations and shaded isosurfaces are important [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
