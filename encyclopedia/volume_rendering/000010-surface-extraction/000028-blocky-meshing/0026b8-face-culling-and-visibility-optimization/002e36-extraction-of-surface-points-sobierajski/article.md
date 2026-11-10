# Extraction of Surface Points (Sobierajski)
Also called: Sobierajski's method, Visible boundary voxel extraction

## Overview
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It sets itself apart from sibling methods like [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by operating in object space to generate a pre-filtered list of surface points rather than generating faces per voxel and culling them during rasterization. It differs from [Voxel Block Shape & Culling Configuration](2176-voxel-block-shape-culling-configuration.md) by focusing on orthographic visibility rather than block-based shape optimization.

## How it works
**Inputs:** Volume data set; set of viewing directions (typically six orthographic views along coordinate axes) [S1, unofficial].
**Outputs:** A list of unique boundary voxels with attributes (coordinates, normal vectors) [S1, unofficial].

**Data Structures:**
- `BoundaryVoxelList`: A dynamic list of structures.
- `BoundaryVoxel`:
  - `position`: 3D integer coordinates $(x, y, z)$ of the voxel.
  - `normal`: 3D float vector $(n_x, n_y, n_z)$ representing the surface normal.
  - `density`: Original density value (used for cutting planes) [S1, unofficial].

**Steps:**
1. **Visibility Determination:** Determine visibility using a set of viewing directions. For medical imaging and general visualization, six orthographic views (along $\pm X, \pm Y, \pm Z$) are sufficient for an approximate set of potentially visible boundary voxels [S1, unofficial].
2. **2D Boundary Tracing:** Perform visibility calculations using a 2D boundary tracing algorithm on slices perpendicular to each coordinate axis. This identifies which voxels are on the boundary relative to the viewing direction [S1, unofficial].
3. **Extraction and Deduplication:** Extract the boundary voxels. Remove duplicate elements from the list to ensure each voxel is processed once [S1, unofficial].
4. **Attribute Assignment:** Store necessary attributes for rendering, specifically coordinates and normal vectors, for each voxel in the list [S1, unofficial].
5. **Backface Culling (Voxel Culling):** Reduce the voxel list size by culling back faces. For a specific viewing direction, if the dot product of the surface normal $\mathbf{n}$ and the viewing direction $\mathbf{v}$ is positive, the voxel is a back face and is not rendered:
   $$ \mathbf{n} \cdot \mathbf{v} > 0 \implies \text{Cull} $$
   [S1, unofficial]
6. **Projection:** Pass the set of surface points to the rendering engine. Project each voxel onto several pixels depending on the viewing direction to avoid holes caused by adjacent voxels mapping to non-adjacent pixels [S1, unofficial].
7. **Shading and Cutting Planes:** Shade projected boundary surface points according to lighting conditions. Voxels intersected by cutting planes are rendered by projecting their original density values onto the image plane [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_surface_points(volume, viewing_directions):
    """
    Extract visible boundary voxels using Sobierajski's method.
    volume: 3D array of density values.
    viewing_directions: List of 3D vectors (e.g., unit vectors along axes).
    Returns: List of BoundaryVoxel objects.
    """
    boundary_list = []
    seen_voxels = set()

    # Step 1 & 2: Determine visibility via 2D boundary tracing on slices
    # Note: The specific 2D boundary tracing algorithm is not detailed in sources [S1, unofficial].
    # We assume a function `get_visible_boundary_voxels` performs this per direction.
    for direction in viewing_directions:
        visible_voxels = get_visible_boundary_voxels(volume, direction)
        
        for voxel in visible_voxels:
            # Step 3: Deduplication
            if voxel.position in seen_voxels:
                continue
            
            # Step 4: Attribute Assignment
            normal = calculate_normal(volume, voxel.position)
            voxel.normal = normal
            voxel.density = volume[voxel.position]
            
            # Step 5: Backface Culling
            # If dot product of normal and viewing direction is positive, it's a back face
            if dot(voxel.normal, direction) > 0:
                continue
                
            boundary_list.append(voxel)
            seen_voxels.add(voxel.position)

    return boundary_list

def calculate_normal(volume, pos):
    """
    Calculate surface normal at a voxel position.
    [G] Standard gradient-based normal calculation for voxel grids.
    """
    # Central difference approximation
    dx = volume[pos.x + 1, pos.y, pos.z] - volume[pos.x - 1, pos.y, pos.z]
    dy = volume[pos.x, pos.y + 1, pos.z] - volume[pos.x, pos.y - 1, pos.z]
    dz = volume[pos.x, pos.y, pos.z + 1] - volume[pos.x, pos.y, pos.z - 1]
    return normalize(vec3(dx, dy, dz))

def dot(v1, v2):
    return v1.x * v2.x + v1.y * v2.y + v1.z * v2.z

def normalize(v):
    length = sqrt(v.x**2 + v.y**2 + v.z**2)
    if length == 0: return vec3(0, 0, 0)
    return vec3(v.x / length, v.y / length, v.z / length)
```

**Edge Cases and Pitfalls:**
- **Holes:** Adjacent voxels may map to non-adjacent pixels, creating holes. Mitigation: Project each voxel onto several pixels [S1, unofficial].
- **Approximation:** Six orthographic views provide an *approximate* set of visible voxels. Increasing the number of directions improves accuracy [S1, unofficial].

## Subtopics

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It extracts a subset of boundary voxels, converts them to geometrical primitives (like crosslines perpendicular to the surface normal), and projects them onto the image plane [S1, unofficial].

### Trimmed Voxel Lists
A surface rendering algorithm that uses 3D points as geometric primitives by projecting exactly one point per visible surface cell to ensure image coverage without subdivision [S2, unofficial]. It identifies visible surface cells, selects exactly one 3D point per cell, and projects each point onto the image plane, allowing it to cover up to three pixels to guarantee coverage [S2, unofficial]. It differs from Dividing Cubes by avoiding recursive subdivision [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the 2D boundary tracing algorithm and the number of viewing directions. Generally faster than full raycasting for surface-only visualization as it reduces the number of primitives [S1, unofficial].
- **Memory Complexity:** Proportional to the number of visible boundary voxels. The list stores coordinates and normals for each voxel [S1, unofficial].
- **Strengths:** Fast display method; supports flexible manipulation of extracted voxels (e.g., cutting planes); efficient for previewing [S1, unofficial].
- **Limitations:** Approximate visibility with few viewing directions; holes may appear if not projected onto multiple pixels; only approximates surfaces contained within the original data [S1, unofficial][S2, unofficial].
- **When to use:** Medical imaging systems requiring fast previewing and cutting operations; scenarios where rendering speed is prioritized over perfect geometric accuracy [S1, unofficial].

## Background
Proposed by Sobierajski [116] as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is cited in the context of medical volume rendering techniques [S1, unofficial]. The method is discussed alongside other surface extraction techniques like Dividing Cubes and Trimmed Voxel Lists in volume visualization literature [S2, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
