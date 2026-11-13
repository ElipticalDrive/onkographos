# Extraction of Surface Points (Sobierajski)
Also called: Visible boundary voxel extraction

## Overview
Extraction of Surface Points is a fast display method for the direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It solves the problem of rendering speed in direct volume rendering by avoiding the processing of occluded geometry. Unlike its siblings in the Blocky meshing family which typically generate explicit mesh primitives (vertices and faces) for the entire surface, this method operates on a point-based representation of the boundary [S1, unofficial]. It sets itself apart by using a multi-view visibility test to prune the voxel list before rendering, rather than relying on geometric culling of generated faces [S1, unofficial].

## How it works
**Inputs:**
*   Volume data grid $V$ of dimensions $D_x \times D_y \times D_z$.
*   A set of viewing directions $\mathcal{D}$ (typically the six orthographic views: $\pm x, \pm y, \pm z$) [S1, unofficial].
*   Rendering parameters: lighting conditions, cutting plane definitions [S1, unofficial].

**Outputs:**
*   A list of visible boundary voxels, where each element contains coordinates $(x, y, z)$ and normal vector $\mathbf{n}$ [S1, unofficial].
*   Rendered image pixels [S1, unofficial].

**Data Structures:**
*   `VisibleVoxelList`: A dynamic list of structures. Each structure stores:
    *   `coords`: Integer tuple $(x, y, z)$ representing the voxel index.
    *   `normal`: 3D vector $(n_x, n_y, n_z)$ representing the surface normal at that voxel.
    *   `density`: Scalar value from the volume data (used for cutting planes) [S1, unofficial].

**Algorithm Steps:**

1.  **Initialize Visibility:** Create a boolean grid `IsVisible` of size $D_x \times D_y \times D_z$, initialized to `false`.
2.  **Determine Visibility via Orthographic Views:**
    For each viewing direction $\mathbf{d} \in \mathcal{D}$:
    *   Identify the axis aligned with $\mathbf{d}$. For example, if $\mathbf{d} = (1, 0, 0)$, the axis is $x$.
    *   Perform a 2D boundary tracing algorithm on slices perpendicular to the axis [S1, unofficial].
        *   *Note on 2D Boundary Tracing:* The source describes this as the method for efficient visibility calculation on orthographic views [S1, unofficial]. In implementation, this typically involves iterating through the slice and identifying boundary voxels (voxels with at least one empty neighbor in the slice plane or along the view direction) that are front-facing relative to $\mathbf{d}$.
    *   Mark all identified visible boundary voxels in `IsVisible` as `true`.
3.  **Construct Voxel List:**
    *   Initialize `VisibleVoxelList` as empty.
    *   Iterate through all voxels $(x, y, z)$ in $V$.
    *   If `IsVisible[x][y][z]` is `true`:
        *   Calculate the surface normal $\mathbf{n}$ for the voxel. For a blocky surface, $\mathbf{n}$ is typically one of the six face normals $\pm \hat{x}, \pm \hat{y}, \pm \hat{z}$, determined by the direction of the empty neighbor [S1, unofficial].
        *   Append a record `{coords: (x,y,z), normal: n, density: V[x][y][z]}` to `VisibleVoxelList`.
4.  **Backface Culling (Voxel Culling):**
    *   For the current specific viewing direction $\mathbf{d}_{curr}$:
    *   Create `FinalVoxelList` as empty.
    *   For each voxel $v$ in `VisibleVoxelList`:
        *   Let $\mathbf{n}$ be the normal of $v$.
        *   Compute dot product $dot = \mathbf{n} \cdot \mathbf{d}_{curr}$.
        *   If $dot > 0$, the voxel is a back face. Discard it [S1, unofficial].
        *   If $dot \le 0$, add $v$ to `FinalVoxelList`.
5.  **Projection and Rendering:**
    *   For each voxel $v$ in `FinalVoxelList`:
        *   Project the voxel onto the image plane. To avoid holes caused by adjacent voxels mapping to non-adjacent pixels, project the voxel onto several pixels depending on the viewing direction [S1, unofficial].
        *   *Implementation Note:* This implies a splatting or rasterization step where the voxel's footprint is expanded to cover the pixel area corresponding to its volume, ensuring no gaps appear in the rendered surface [S1, unofficial].
        *   If cutting planes are defined, check if the voxel is intersected by any cutting plane.
            *   If intersected, render the voxel by projecting its original density value onto the image plane [S1, unofficial].
            *   If not intersected, shade the projected boundary surface point according to lighting conditions [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_visible_boundary_voxels(volume, dimensions, viewing_directions):
    """
    volume: 3D array of density values
    dimensions: tuple (Dx, Dy, Dz)
    viewing_directions: list of 3D vectors (e.g., [(1,0,0), (-1,0,0), ...])
    """
    Dx, Dy, Dz = dimensions
    is_visible = [[False] * Dz for _ in range(Dy)] # Placeholder for 3D structure logic
    # Note: The source specifies a 3D grid of voxels. 
    # We need a 3D boolean array.
    is_visible_3d = [[[False] * Dz for _ in range(Dy)] for _ in range(Dx)]

    # Step 2: Determine visibility
    for d in viewing_directions:
        # Identify axis. Example: d=(1,0,0) -> axis=0 (x)
        axis = d.index(max(d, key=abs)) # Assumes unit vectors
        
        # Perform 2D boundary tracing on slices perpendicular to axis
        # This marks boundary voxels visible from direction d
        mark_visible_from_direction(volume, is_visible_3d, d, axis)

    # Step 3: Construct list
    visible_voxel_list = []
    for x in range(Dx):
        for y in range(Dy):
            for z in range(Dz):
                if is_visible_3d[x][y][z]:
                    normal = calculate_surface_normal(volume, x, y, z)
                    visible_voxel_list.append({
                        'coords': (x, y, z),
                        'normal': normal,
                        'density': volume[x][y][z]
                    })

    return visible_voxel_list

def mark_visible_from_direction(volume, is_visible_3d, d, axis):
    """
    Marks voxels visible from direction d using 2D boundary tracing.
    """
    # Implementation detail: Iterate through slices perpendicular to axis.
    # For each slice, find boundary voxels.
    # A voxel is a boundary voxel if it is occupied and has an empty neighbor.
    # It is visible from d if it is the first occupied voxel along ray d.
    # The source specifies "2D boundary tracing algorithm on slices perpendicular to each coordinate axis" [S1, unofficial].
    # This implies checking the 2D projection of the volume for that view.
    
    # Pseudocode for 2D boundary tracing on a slice:
    # For each slice index s along the axis:
    #   For each voxel (x,y,z) in slice s:
    #     If voxel is occupied:
    #       Check if it is a boundary voxel (neighbor in direction -d is empty or out of bounds)
    #       If boundary, mark is_visible_3d[x][y][z] = True

def calculate_surface_normal(volume, x, y, z):
    """
    Returns the normal of the first empty neighbor.
    """
    neighbors = [
        ((1,0,0), (1,0,0)), ((-1,0,0), (-1,0,0)),
        ((0,1,0), (0,1,0)), ((0,-1,0), (0,-1,0)),
        ((0,0,1), (0,0,1)), ((0,0,-1), (0,0,-1))
    ]
    for (dx, dy, dz), normal in neighbors:
        nx, ny, nz = x + dx, y + dy, z + dz
        if not is_occupied(volume, nx, ny, nz):
            return normal
    return (0, 0, 0) # Should not happen for boundary voxels

def is_occupied(volume, x, y, z):
    Dx, Dy, Dz = volume.shape
    if 0 <= x < Dx and 0 <= y < Dy and 0 <= z < Dz:
        return volume[x][y][z] > 0 # Assuming thresholded volume
    return False

def render_voxels(voxel_list, viewing_direction, image_plane, cutting_planes=None):
    """
    Renders the voxel list to the image plane.
    """
    final_list = []
    for v in voxel_list:
        dot = dot_product(v['normal'], viewing_direction)
        if dot <= 0: # Backface culling
            final_list.append(v)
    
    for v in final_list:
        # Project voxel to multiple pixels to avoid holes
        pixels = project_voxel_to_pixels(v['coords'], viewing_direction)
        
        for p in pixels:
            if cutting_planes and is_intersected(v['coords'], cutting_planes):
                # Render density value
                write_density_to_pixel(p, v['density'])
            else:
                # Shade according to lighting
                color = shade(v['normal'], v['coords'])
                write_color_to_pixel(p, color)
```

## Subtopics

### [Trimmed Voxel Lists](11309-trimmed-voxel-lists.md)
Trimmed Voxel Lists is a surface extraction algorithm that represents an iso-surface using 3D points as geometric primitives, projecting exactly one point per visible surface cell to ensure image coverage without recursive subdivision. It differs from Dividing Cubes by avoiding subdivision; instead, it allows a single projected point to span up to three pixels on the image plane to guarantee coverage.

### Saito's Fast Previewing Algorithm
Saito's Fast Previewing Algorithm is a previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It differs from the full extraction by sampling only a subset of the boundary voxels uniformly [S1, unofficial]. The extracted surface samples are converted to geometrical primitives like crosslines perpendicular to the surface normal and projected onto the image plane [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The visibility calculation depends on the number of viewing directions and the cost of the 2D boundary tracing. For $N$ voxels and $K$ viewing directions, the complexity is roughly $O(K \cdot N)$ for the marking phase, plus $O(N)$ for list construction and culling [S1, unofficial].
*   **Memory Complexity:** Requires storage for the volume data, a boolean visibility grid ($O(N)$), and the list of visible voxels ($O(N)$ in the worst case) [S1, unofficial].
*   **Strengths:** Fast rendering speed due to the reduction of the voxel list via visibility culling and backface culling [S1, unofficial]. Simple implementation using standard volume data structures [S1, unofficial].
*   **Limitations:** Potential for holes in the rendered image if the projection step does not adequately cover the pixel area (mitigated by projecting to several pixels) [S1, unofficial]. The approximation of visibility using only six orthographic views may miss some visible voxels in complex geometries, though increasing the number of directions improves accuracy [S1, unofficial].
*   **When to use:** Suitable for real-time or interactive visualization of volume data where speed is prioritized over geometric completeness, particularly in medical imaging systems [S1, unofficial].

## Background
The method was proposed by Sobierajski [S1, unofficial]. It is described in the context of medical imaging systems, where cutting operations and simultaneous rendering of shaded isosurfaces and cross-sectional slices are important [S1, unofficial]. The approach is part of a family of direct volume rendering techniques that aim to balance speed and visual quality by manipulating extracted voxels [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
