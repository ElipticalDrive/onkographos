# Extraction of Surface Points (Sobierajski)
Also called: Sobierajski's method, Visible boundary voxel extraction

## Overview
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It sets itself apart from sibling methods like [Face Culling](8735-face-culling.md) by operating on a pre-extracted list of surface points rather than generating faces per voxel, and from [Voxel Block Shape & Culling Configuration](2176-voxel-block-shape-culling-configuration.md) by using orthographic visibility checks to prune the voxel list before projection [S1, unofficial].

## How it works
**Inputs:** Volume data grid; set of viewing directions (typically six orthographic views: $\pm X, \pm Y, \pm Z$) [S1, unofficial].
**Outputs:** List of visible boundary voxels with attributes (coordinates, normal vectors) [S1, unofficial].

**Data Structures:**
*   **Boundary Voxel List:** A dynamic list storing tuples $(x, y, z, n_x, n_y, n_z)$ for each visible boundary voxel. Duplicates are removed during extraction [S1, unofficial].

**Steps:**
1.  **Visibility Determination:** For each of the $N$ viewing directions, determine which boundary voxels are potentially visible. Using the six orthographic views, this is performed efficiently via a 2D boundary tracing algorithm on slices perpendicular to each coordinate axis [S1, unofficial].
2.  **Extraction and Deduplication:** Extract the set of boundary voxels identified as visible. Remove duplicate entries from the list [S1, unofficial].
3.  **Attribute Assignment:** For each voxel in the list, compute and store necessary rendering attributes, specifically coordinates and the surface normal vector [S1, unofficial].
4.  **Backface Culling (Voxel Culling):** For a specific viewing direction, reduce the voxel list size by removing back faces. A voxel is culled if the dot product of its surface normal $\mathbf{n}$ and the viewing direction $\mathbf{v}$ is positive:
    $$ \mathbf{n} \cdot \mathbf{v} > 0 $$
    If this condition holds, the voxel belongs to a back face and is not rendered [S1, unofficial].
5.  **Projection:** Project each remaining voxel onto the image plane. To avoid holes caused by adjacent voxels mapping to non-adjacent pixels, each voxel is projected onto several pixels depending on the viewing direction [S1, unofficial].
6.  **Rendering:** Pass the set of surface points to the rendering engine. Shade the projected boundary surface points according to lighting conditions [S1, unofficial]. If cutting planes are used, voxels intersected by the plane are rendered by projecting their original density values onto the image plane [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_visible_boundary_voxels(volume, viewing_directions):
    """
    Extracts visible boundary voxels using Sobierajski's method.
    volume: 3D array of density values.
    viewing_directions: List of 3D vectors (e.g., [1,0,0], [-1,0,0], ...).
    Returns: List of visible boundary voxels with normals.
    """
    visible_voxels = []
    
    # Step 1 & 2: Visibility determination and extraction
    # Using 2D boundary tracing on slices perpendicular to axes for orthographic views
    for direction in viewing_directions:
        # Identify boundary voxels visible from this direction
        # This involves tracing boundaries on 2D slices perpendicular to the direction
        # and checking if the voxel is on the "front" side relative to the viewer
        voxels_from_dir = trace_2d_boundary_slices(volume, direction)
        visible_voxels.extend(voxels_from_dir)
        
    # Remove duplicates
    unique_voxels = remove_duplicates(visible_voxels)
    
    # Step 3: Attribute assignment
    for voxel in unique_voxels:
        voxel.normal = compute_normal(volume, voxel.position)
        
    return unique_voxels

def render_sobierajski(visible_voxels, viewing_direction, image_plane):
    """
    Renders the extracted boundary voxels.
    """
    # Step 4: Backface culling
    cullable_voxels = []
    for voxel in visible_voxels:
        # Cull if normal dot viewing direction is positive
        if dot(voxel.normal, viewing_direction) <= 0:
            cullable_voxels.append(voxel)
            
    # Step 5: Projection
    for voxel in cullable_voxels:
        # Project voxel onto multiple pixels to avoid holes
        projected_pixels = project_onto_pixels(voxel.position, voxel.normal, viewing_direction)
        for pixel in projected_pixels:
            image_plane.set_pixel(pixel.x, pixel.y, shade(voxel, viewing_direction))
            
    # Step 6: Cutting planes (if applicable)
    # Render voxels intersected by cutting plane by projecting density values
```

## Subtopics

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It extracts a subset of boundary voxels, converts them to geometrical primitives like crosslines perpendicular to the surface normal, and projects these primitives onto the image plane [S1, unofficial].

### Trimmed Voxel Lists
A surface rendering algorithm that uses 3D points as geometric primitives by projecting exactly one point per visible surface cell to ensure image coverage without subdivision [S2, unofficial]. It identifies visible surface cells, selects exactly one 3D point per cell, and projects each point onto the image plane, allowing it to cover up to three pixels to guarantee coverage [S2, unofficial]. It differs from Dividing Cubes by avoiding recursive subdivision [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the 2D boundary tracing on slices and the projection step. The number of voxels to be projected is reduced by backface culling [S1, unofficial].
*   **Memory Complexity:** Requires storage for the list of boundary voxels and their attributes. The list size is reduced by culling [S1, unofficial].
*   **Strengths:** Fast display method; efficient visibility calculation using 2D boundary tracing; supports cutting planes easily [S1, unofficial].
*   **Limitations:** Holes can appear if adjacent voxels are not projected onto adjacent pixels; requires multiple projections per voxel to avoid holes [S1, unofficial].
*   **When to use:** When fast previewing or direct rendering of boundary surfaces is needed, and orthographic views are sufficient for visibility determination [S1, unofficial].

## Background
Proposed by Sobierajski [116] as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is cited in the context of medical imaging systems where cutting operations are important [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
