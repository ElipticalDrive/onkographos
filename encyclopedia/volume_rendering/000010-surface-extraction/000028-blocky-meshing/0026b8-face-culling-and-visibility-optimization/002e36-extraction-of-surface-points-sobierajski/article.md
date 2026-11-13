# Extraction of Surface Points (Sobierajski)
Also called: Visible boundary voxel extraction

## Overview
Extraction of Surface Points is a direct volume rendering method that accelerates surface visualization by extracting only the visible boundary voxels from the volume data [S1, unofficial]. It solves the performance bottleneck of rendering entire isosurfaces by projecting a sparse set of surface points onto the image plane, ensuring coverage without generating a full mesh. It differs from sibling methods like [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by operating in image space via projection rather than object space via geometry generation, and from [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by using viewing-direction-based visibility tests instead of neighbor-based internal face culling.

## How it works
**Inputs:**
*   Volume data grid $V$ with dimensions $D_x \times D_y \times D_z$.
*   A set of viewing directions $\mathcal{D} = \{d_1, d_2, ..., d_k\}$. Typically, the six orthographic directions $\{\pm x, \pm y, \pm z\}$ are used [S1, unofficial].
*   Threshold value $T$ for boundary detection (implicit in "boundary voxels").

**Outputs:**
*   A list of surface points $\mathcal{P}$, where each point $p \in \mathcal{P}$ contains coordinates $(x, y, z)$ and a normal vector $\mathbf{n}$.

**Data Structures:**
*   **Boundary Voxel List:** A dynamic list storing tuples $(x, y, z, \mathbf{n})$. Duplicates are removed before passing to the renderer.
*   **Visibility Masks:** 2D arrays (slices) used during the boundary tracing phase to track which voxels are visible from specific directions.

**Steps:**

1.  **Visibility Determination:**
    For each viewing direction $d_i \in \mathcal{D}$, determine which boundary voxels are potentially visible.
    *   If using the six orthographic views, perform a 2D boundary tracing algorithm on slices perpendicular to each coordinate axis [S1, unofficial].
    *   A voxel is considered a boundary voxel if it is occupied and adjacent to an empty voxel (or out of bounds) in the direction of the viewing ray.
    *   Collect all such voxels into a candidate set.

2.  **Deduplication:**
    Remove duplicate entries from the candidate set. A voxel may be visible from multiple viewing directions; it should appear only once in the final list $\mathcal{P}$ [S1, unofficial].

3.  **Attribute Assignment:**
    For each unique voxel in $\mathcal{P}$, compute and store:
    *   Coordinates $(x, y, z)$.
    *   Surface normal vector $\mathbf{n}$. The normal is typically derived from the gradient of the volume data or the direction to the adjacent empty voxel [S1, unofficial].

4.  **Backface Culling (Voxel Culling):**
    For a specific rendering view with viewing direction $\mathbf{v}$, reduce the list size by removing back-facing voxels.
    *   For each voxel $p \in \mathcal{P}$ with normal $\mathbf{n}_p$:
        $$ \text{if } \mathbf{n}_p \cdot \mathbf{v} > 0 \text{ then remove } p \text{ from the active list} $$
    *   Voxels where the dot product is positive are facing away from the camera and are not rendered [S1, unofficial].

5.  **Projection and Coverage:**
    Project the remaining surface points onto the image plane.
    *   To avoid holes caused by adjacent voxels mapping to non-adjacent pixels, project each voxel onto multiple pixels depending on the viewing direction [S1, unofficial].
    *   This "splatting" or multi-pixel projection ensures image coverage without recursive subdivision [S1, unofficial].

6.  **Shading and Cutting Planes:**
    *   Shade the projected boundary surface points according to lighting conditions (e.g., Phong shading) [S1, unofficial].
    *   If cutting planes are used, render voxels intersected by the cutting plane by projecting their original density values onto the image plane, overlaying the shaded surface [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_surface_points(volume, viewing_directions):
    """
    Extracts visible boundary voxels for direct rendering.
    volume: 3D array of density values.
    viewing_directions: List of 3D vectors (e.g., [1,0,0], [-1,0,0], ...).
    Returns: List of dicts with 'pos' (x,y,z) and 'normal' (nx,ny,nz).
    """
    boundary_set = set()
    
    # Step 1 & 2: Visibility Determination and Deduplication
    # Using 6 orthographic views as per source
    axes = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
    
    for direction in axes:
        # Perform 2D boundary tracing on slices perpendicular to direction
        # This identifies voxels that are on the boundary relative to this view
        visible_voxels = trace_boundary_2d(volume, direction)
        for voxel in visible_voxels:
            # Compute normal (simplified: direction to empty neighbor or gradient)
            normal = compute_normal(volume, voxel)
            boundary_set.add((voxel, normal))
            
    # Convert set to list
    surface_points = list(boundary_set)
    
    return surface_points

def render_surface_points(surface_points, camera_view_direction):
    """
    Renders the extracted points with backface culling and projection.
    """
    active_points = []
    
    # Step 4: Backface Culling
    for point in surface_points:
        dot_product = dot(point['normal'], camera_view_direction)
        if dot_product <= 0:
            active_points.append(point)
            
    # Step 5: Projection
    for point in active_points:
        # Project to multiple pixels to avoid holes
        projected_pixels = project_to_pixels(point['pos'], camera_view_direction)
        for pixel in projected_pixels:
            # Step 6: Shading
            color = shade(point['normal'], camera_view_direction)
            draw_pixel(pixel, color)
```

**Parameters:**
*   **Viewing Directions:** Typically 6 orthographic views are sufficient for an approximate set of visible voxels [S1, unofficial]. Increasing the number of directions improves approximation but increases extraction cost.
*   **Projection Radius:** The number of pixels each voxel is projected onto depends on the viewing angle and resolution to prevent holes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Holes:** If each voxel is projected to only one pixel, holes may appear in the image because adjacent voxels map to non-adjacent pixels [S1, unofficial]. Multi-pixel projection is required.
*   **Backface Culling Condition:** The condition $\mathbf{n} \cdot \mathbf{v} > 0$ identifies back faces. If the normal is not consistently oriented, this culling may fail [S1, unofficial].

## Subtopics

### [Trimmed Voxel Lists](11309-trimmed-voxel-lists.md)
Trimmed Voxel Lists is a surface extraction algorithm that represents an iso-surface using 3D points as geometric primitives, projecting exactly one point per visible surface cell to ensure image coverage without recursive subdivision. It differs from Dividing Cubes by avoiding subdivision; instead, it allows a single projected point to span up to three pixels on the image plane to guarantee coverage.

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It extracts a subset of boundary voxels, converts them to geometrical primitives like crosslines perpendicular to the surface normal, and projects these primitives onto the image plane [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Extraction is $O(N)$ where $N$ is the number of voxels, dominated by the 2D boundary tracing on slices. Rendering is $O(M)$ where $M$ is the number of visible boundary voxels, which is typically much smaller than $N$.
*   **Memory Complexity:** $O(M)$ to store the list of surface points and their attributes.
*   **Performance:** Rendering speed is directly related to the length of the voxel list. Backface culling significantly reduces the number of voxels to be projected [S1, unofficial].
*   **Strengths:** Fast previewing; simple implementation; supports cutting planes easily by overlaying density projections [S1, unofficial].
*   **Limitations:** Approximate visibility (depends on viewing directions); potential for holes if projection is not handled carefully; less geometrically accurate than mesh-based methods [S1, unofficial].
*   **When to use:** Fast previewing in medical imaging systems where flexible manipulation of extracted voxels is needed [S1, unofficial].
*   **When not to use:** When high-fidelity, hole-free surface geometry is required without post-processing.

## Background
The method was proposed by Sobierajski [116] as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is cited in the context of medical imaging systems, where cutting operations (rendering shaded isosurfaces and arbitrary cross-sectional slices simultaneously) are important [S1, unofficial]. The approach is classified as a hybrid between image-order and object-order algorithms, similar to shear-warp factorization, but specifically focused on boundary voxel extraction [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
