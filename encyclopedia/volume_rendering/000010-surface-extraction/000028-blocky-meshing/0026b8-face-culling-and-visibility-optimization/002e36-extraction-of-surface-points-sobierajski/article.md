# Extraction of Surface Points (Sobierajski)
Also called: Visible boundary voxel extraction

## Overview
Extraction of Surface Points is a fast display method for the direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It solves the problem of high geometric complexity in voxel-based rendering by pre-filtering the voxel list based on viewing direction, thereby reducing the number of primitives passed to the rendering engine. Unlike its siblings which often rely on block-level culling or mesh generation, this method operates directly on individual boundary voxels, projecting them as points or small primitives to approximate the surface [S1, unofficial].

## How it works
**Inputs:**
- Volume data grid $V$ with dimensions $D_x \times D_y \times D_z$.
- A set of viewing directions $\mathcal{D}$, typically the six orthographic directions $\{\pm \hat{x}, \pm \hat{y}, \pm \hat{z}\}$ [S1, unofficial].
- Rendering parameters: lighting conditions, cutting plane definitions.

**Outputs:**
- A list of unique boundary voxels, each storing coordinates $(x, y, z)$ and normal vector $\mathbf{n}$ [S1, unofficial].
- Rendered image.

**Data Structures:**
- `BoundaryVoxel`: A struct containing:
  - `coords`: 3D integer coordinates $(x, y, z)$.
  - `normal`: 3D float vector $(n_x, n_y, n_z)$.
- `VoxelList`: A dynamic array of `BoundaryVoxel` objects.

**Steps:**

1. **Visibility Determination via Orthographic Views:**
   Iterate through each of the six orthographic viewing directions. For each direction, determine which boundary voxels are potentially visible.
   - A voxel is a boundary voxel if it is occupied and has at least one adjacent empty (or transparent/non-existent) neighbor.
   - Visibility is calculated efficiently using a 2D boundary tracing algorithm on slices perpendicular to the coordinate axis corresponding to the viewing direction [S1, unofficial].
   - *Note:* The source states that six orthographic views are sufficient for an approximate set of visible voxels, but better approximation can be achieved by increasing the number of directions [S1, unofficial].

2. **List Generation and Deduplication:**
   Collect all boundary voxels identified as visible from any of the viewing directions. Remove duplicate entries from the list. Each voxel in the final list stores its coordinates and normal vector [S1, unofficial].

3. **Backface Culling (Voxel Culling):**
   For a specific viewing direction $\mathbf{v}$, reduce the voxel list size by removing back-facing voxels.
   - Calculate the dot product between the voxel's surface normal $\mathbf{n}$ and the viewing direction $\mathbf{v}$.
   - If $\mathbf{n} \cdot \mathbf{v} > 0$, the voxel is a back face and is not rendered [S1, unofficial].
   - *Note:* The condition $\mathbf{n} \cdot \mathbf{v} > 0$ implies the normal points towards the viewer, which is standard for backface culling if $\mathbf{v}$ is defined as the vector from the viewer to the scene, or if the normal is defined outward and the view vector is inward. The source explicitly states: "If the dot product of the surface normal and the viewing direction is positive the given voxel belongs to a back face, therefore it is not rendered" [S1, unofficial].

4. **Projection and Hole Avoidance:**
   Pass the set of surface points to the rendering engine.
   - Since adjacent voxels may map to non-adjacent pixels, holes can appear in the image.
   - To avoid this, project each voxel onto several pixels depending on the viewing direction [S1, unofficial].
   - *Note:* The exact projection geometry (e.g., splatting radius, point size) is not specified in the source, only that multiple pixels are covered per voxel [S1, unofficial].

5. **Rendering and Shading:**
   - Shade the projected boundary surface points according to lighting conditions [S1, unofficial].
   - If cutting planes are used, render voxels intersected by the cutting plane by projecting their original density values onto the image plane [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_surface_points(volume, viewing_directions):
    """
    Extracts visible boundary voxels from volume data.
    
    Args:
        volume: 3D array of density values.
        viewing_directions: List of 3D vectors representing viewing directions.
        
    Returns:
        list of BoundaryVoxel objects.
    """
    boundary_voxels = set()
    
    # Step 1 & 2: Determine visibility and generate list
    for direction in viewing_directions:
        # Efficient 2D boundary tracing on slices perpendicular to direction
        # This identifies boundary voxels visible from this direction
        visible_voxels = trace_boundary_slices(volume, direction)
        
        for voxel in visible_voxels:
            # Calculate normal vector for the voxel face facing the direction
            # The source implies normals are stored; here we assume a function to compute it
            normal = compute_normal(voxel, volume)
            boundary_voxels.add((voxel.coords, normal))
            
    # Convert set to list to remove duplicates
    unique_voxels = list(boundary_voxels)
    
    return unique_voxels

def render_surface_points(voxel_list, viewing_direction, lighting, cutting_plane=None):
    """
    Renders the extracted surface points.
    
    Args:
        voxel_list: List of (coords, normal) tuples.
        viewing_direction: 3D vector of the current view.
        lighting: Lighting parameters.
        cutting_plane: Optional plane definition.
    """
    image = create_empty_image()
    
    for coords, normal in voxel_list:
        # Step 3: Backface Culling
        # If dot product is positive, it is a back face and not rendered
        if dot(normal, viewing_direction) > 0:
            continue
            
        # Step 4: Projection
        # Project voxel onto several pixels to avoid holes
        projected_pixels = project_onto_pixels(coords, viewing_direction)
        
        for pixel in projected_pixels:
            # Step 5: Shading
            color = shade(pixel, normal, lighting)
            
            # Check for cutting plane intersection
            if cutting_plane is not None and is_intersected(coords, cutting_plane):
                # Render original density values for intersected voxels
                density = get_density(coords)
                color = blend_with_density(color, density)
                
            image.set_pixel(pixel, color)
            
    return image
```

## Subtopics

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It differs from the main method by not extracting all visible boundary voxels, but rather a sparse subset. The extracted surface samples are converted to geometrical primitives like crosslines perpendicular to the surface normal and projected onto the image plane [S1, unofficial].

### Trimmed Voxel Lists
A surface rendering algorithm that uses 3D points as geometric primitives by projecting exactly one point per visible surface cell to ensure image coverage without subdivision [S2, unofficial]. It differs from Dividing Cubes by avoiding recursive subdivision; it achieves pixel coverage by projecting a single point per cell that can span multiple pixels (up to three) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The visibility calculation is efficient due to the use of 2D boundary tracing on slices [S1, unofficial]. The complexity is proportional to the number of boundary voxels and the number of viewing directions.
- **Memory Complexity:** Stores a list of boundary voxels with coordinates and normals. The size is significantly smaller than the full volume but larger than a block-based mesh if many small structures are present.
- **Strengths:** Fast previewing; simple implementation; supports cutting planes natively by projecting density values [S1, unofficial].
- **Limitations:** Potential for holes in the rendered image if voxel projection is not handled carefully (mitigated by projecting to multiple pixels) [S1, unofficial]. Approximate surface representation; accuracy depends on the number of viewing directions and voxel size [S1, unofficial].
- **When to use:** Fast previewing, medical imaging systems where cutting operations are important [S1, unofficial].
- **When not to use:** When high-fidelity surface reconstruction is required without subdivision, or when the scene has complex internal structures that are not well-approximated by boundary voxels.

## Background
The method was proposed by Sobierajski [116] as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is cited in the context of medical imaging systems, where flexible manipulation of extracted voxels, such as simultaneous rendering of shaded isosurfaces and arbitrary cross-sectional slices, is important [S1, unofficial]. The approach is part of a broader family of hybrid acceleration methods in volume visualization, distinct from pure raycasting or splatting, by focusing on object-order extraction of boundary primitives [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
