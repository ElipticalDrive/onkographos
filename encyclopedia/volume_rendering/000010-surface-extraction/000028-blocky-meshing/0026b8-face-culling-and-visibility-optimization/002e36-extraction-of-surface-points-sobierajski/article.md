# Extraction of Surface Points (Sobierajski)
Also called: Visible boundary voxel extraction

## Overview
Extraction of Surface Points is a fast display method for the direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It solves the problem of high computational cost in volume rendering by avoiding the processing of internal or occluded geometry. It sets itself apart from sibling methods like [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by operating on individual voxel points rather than blocky meshes, and from [Face Culling](8735-face-culling.md) by using a pre-extraction visibility filter based on orthographic views rather than per-face geometric checks during rasterization.

## How it works
**Inputs:** Volume data (density/opacity field); viewing directions (typically six orthographic views).
**Outputs:** A list of surface points (coordinates, normal vectors) ready for rendering.

**Data Structures:**
*   `VoxelList`: A dynamic array or linked list storing extracted boundary voxels.
*   `VoxelElement`: Struct containing:
    *   `coords`: 3D integer coordinates $(x, y, z)$ of the voxel.
    *   `normal`: 3D float vector $(n_x, n_y, n_z)$ representing the surface normal.
    *   `density`: Scalar value (optional, for cutting plane rendering).

**Steps:**
1.  **Visibility Determination:** Determine potential visibility using a set of viewing directions. Six orthographic views (along $\pm X, \pm Y, \pm Z$) are typically sufficient for an approximate set of all potentially visible boundary voxels [S1, unofficial]. Increasing the number of directions improves approximation accuracy.
2.  **2D Boundary Tracing:** Perform visibility calculations efficiently using a 2D boundary tracing algorithm on slices perpendicular to each coordinate axis [S1, unofficial]. This identifies which voxels on the boundary of solid regions are exposed to the viewing direction.
3.  **Extraction and Deduplication:** Extract the identified boundary voxels. Remove duplicate elements from the list to ensure each physical boundary voxel is represented once [S1, unofficial].
4.  **Attribute Assignment:** Store necessary attributes for each voxel in the list, specifically coordinates and normal vectors [S1, unofficial].
5.  **Backface Culling (Voxel Culling):** Reduce the voxel list size by filtering based on the viewing direction. For a specific viewing direction vector $\vec{v}$ and a voxel's surface normal $\vec{n}$:
    *   If $\vec{n} \cdot \vec{v} > 0$, the voxel is a back face and is not rendered [S1, unofficial].
    *   Otherwise, it is retained.
6.  **Projection:** Pass the set of surface points to the rendering engine. Project each voxel onto several pixels depending on the viewing direction to avoid holes in the produced image, as adjacent voxels may map to non-adjacent pixels [S1, unofficial].
7.  **Shading and Cutting Planes:** Shade projected boundary surface points according to lighting conditions. If cutting planes are used, render voxels intersected by the cutting plane by projecting their original density values onto the image plane [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_surface_points(volume_data, viewing_directions):
    """
    Extract visible boundary voxels using Sobierajski's method.
    
    Args:
        volume_data: 3D array of density values.
        viewing_directions: List of 3D vectors representing view directions.
        
    Returns:
        surface_points: List of dicts with 'coords', 'normal', 'density'.
    """
    surface_points = []
    seen_coords = set()
    
    # Step 1 & 2: Determine visibility via 2D boundary tracing on slices
    # Note: The specific 2D boundary tracing algorithm is not detailed in sources [S1, unofficial]
    # We assume a function that returns boundary voxels visible from a given direction
    for view_dir in viewing_directions:
        visible_voxels = perform_2d_boundary_tracing(volume_data, view_dir)
        
        for voxel in visible_voxels:
            coords = voxel.coords
            normal = voxel.normal
            
            # Step 3: Deduplication
            if coords in seen_coords:
                continue
            seen_coords.add(coords)
            
            # Step 5: Backface Culling
            # If dot product of surface normal and viewing direction is positive, it is a back face
            if dot(normal, view_dir) > 0:
                continue
                
            # Step 4: Store attributes
            point = {
                'coords': coords,
                'normal': normal,
                'density': volume_data[coords]
            }
            surface_points.append(point)
            
    # Step 6 & 7: Projection and Shading are handled by the rendering engine
    # The list is passed to the renderer which projects each voxel to multiple pixels
    return surface_points

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]
```

**Parameters:**
*   `viewing_directions`: Typically 6 orthographic views. More directions increase accuracy [S1, unofficial].
*   `projection_radius`: Not explicitly defined, but "several pixels" are used per voxel to prevent holes [S1, unofficial].

**Edge Cases:**
*   **Holes:** Adjacent voxels may map to non-adjacent pixels, creating holes. Mitigated by projecting each voxel onto multiple pixels [S1, unofficial].
*   **Duplicate Voxels:** A voxel may be visible from multiple viewing directions. Deduplication is required [S1, unofficial].

## Subtopics

### [Trimmed Voxel Lists](11309-trimmed-voxel-lists.md)
Trimmed Voxel Lists is a surface extraction algorithm that represents an iso-surface using 3D points as geometric primitives, projecting exactly one point per visible surface cell to ensure image coverage without recursive subdivision. It differs from Dividing Cubes by avoiding subdivision; instead, it allows a single projected point to span up to three pixels on the image plane to guarantee coverage.

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It extracts a subset of boundary voxels, converts them to geometrical primitives like crosslines perpendicular to the surface normal, and projects them onto the image plane [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the 2D boundary tracing algorithm and the number of viewing directions. Typically faster than full volume raycasting as it reduces the number of primitives.
*   **Memory Complexity:** Proportional to the number of visible boundary voxels.
*   **Strengths:** Fast previewing; simple implementation; supports cutting planes easily [S1, unofficial].
*   **Limitations:** Approximate visibility (may miss some visible voxels or include some occluded ones if fewer than 6 views are used); potential for holes if projection is not handled carefully [S1, unofficial].
*   **When to use:** Fast previews, medical imaging systems where cutting operations are important [S1, unofficial].

## Background
Proposed by Sobierajski [116] as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is cited in surveys of medical volume rendering techniques [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
