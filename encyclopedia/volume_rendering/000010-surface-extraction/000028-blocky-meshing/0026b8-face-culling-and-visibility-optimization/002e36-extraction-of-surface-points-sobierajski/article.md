# Extraction of Surface Points (Sobierajski)
Also called: Sobierajski's method, Visible boundary voxel extraction

## Overview
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It differs from standard marching cubes or raycasting by operating in object space: it pre-computes a list of visible surface points (voxels) based on viewing directions, then projects them to the image plane. This avoids per-pixel ray traversal but requires handling projection artifacts like holes.

## How it works
**Inputs:** 3D volume data grid $V$ with dimensions $D_x \times D_y \times D_z$; set of viewing directions $\mathcal{D}$ (typically 6 orthographic axes); camera parameters.
**Outputs:** List of projected surface points with attributes (screen coordinates, normal, density).

**Data Structures:**
- `VoxelList`: A dynamic array of structures. Each element contains:
  - `pos`: 3D integer coordinates $(x, y, z)$.
  - `normal`: 3D float vector $(n_x, n_y, n_z)$.
  - `density`: float value from $V$.
- `VisibilityMask`: 3D boolean array or hash set indicating if a voxel is already processed to remove duplicates.

**Steps:**
1. **Visibility Determination:** For each voxel $v$ in $V$, determine if it is a boundary voxel. A voxel is a boundary voxel if it is occupied and has at least one empty neighbor.
2. **Viewing Direction Check:** For each boundary voxel, check visibility against each direction $d \in \mathcal{D}$. A voxel is visible from direction $d$ if no other occupied voxel lies between it and the camera along $d$. For orthographic views, this simplifies to checking if the voxel is the first occupied voxel along the ray parallel to $d$ passing through its position.
3. **2D Boundary Tracing:** To efficiently find visible boundary voxels for orthographic views, perform 2D boundary tracing on slices perpendicular to each coordinate axis. For example, for the $+X$ view, iterate through $YZ$ planes. In each plane, trace the boundary of occupied regions. Only voxels on the outer boundary of the projection are candidates.
4. **Duplicate Removal:** Store visible boundary voxels in `VoxelList`. Use `VisibilityMask` to ensure each voxel is added only once, even if visible from multiple directions.
5. **Backface Culling:** For each voxel in `VoxelList`, calculate the dot product between its surface normal $\mathbf{n}$ and the viewing direction $\mathbf{d}$. If $\mathbf{n} \cdot \mathbf{d} > 0$, the voxel is a back face and is removed from the list [S1, unofficial].
6. **Projection:** Project each remaining voxel onto the image plane. To avoid holes caused by adjacent voxels mapping to non-adjacent pixels, project each voxel onto multiple pixels depending on the viewing direction and voxel size [S1, unofficial].
7. **Rendering:** Pass the projected points to the rendering engine. Shade points according to lighting. For cutting planes, render voxels intersected by the plane by projecting their original density values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_surface_points(volume, viewing_directions):
    voxel_list = []
    visited = set()
    
    # Step 1 & 2: Determine visible boundary voxels
    for direction in viewing_directions:
        # Step 3: 2D boundary tracing for orthographic views
        # Simplified: iterate through slices perpendicular to direction
        # For each slice, find boundary voxels
        boundary_voxels = get_boundary_voxels_for_direction(volume, direction)
        
        for voxel in boundary_voxels:
            if voxel.pos not in visited:
                # Step 5: Backface culling
                normal = compute_normal(volume, voxel.pos)
                if dot(normal, direction) <= 0:
                    voxel_list.append(voxel)
                    visited.add(voxel.pos)
    
    # Step 6: Projection
    projected_points = []
    for voxel in voxel_list:
        # Project voxel to multiple pixels to avoid holes
        pixels = project_onto_pixels(voxel, viewing_directions)
        for pixel in pixels:
            projected_points.append(pixel)
            
    return projected_points
```

**Parameters:**
- `viewing_directions`: Typically 6 orthographic directions $(\pm x, \pm y, \pm z)$. More directions improve approximation [S1, unofficial].
- `voxel_projection_size`: Number of pixels per voxel to avoid holes. Not given in the sources.

**Edge Cases:**
- Holes in the image if voxel projection is insufficient [S1, unofficial].
- Duplicate voxels if visibility check is not robust [S1, unofficial].

## Subtopics

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It extracts a subset of boundary voxels, converts them to geometrical primitives like crosslines perpendicular to the surface normal, and projects them onto the image plane [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of boundary voxels and viewing directions. 2D boundary tracing is efficient for orthographic views.
- **Memory Complexity:** Proportional to the number of visible boundary voxels.
- **Strengths:** Fast for previewing; simple implementation; supports cutting planes easily [S1, unofficial].
- **Limitations:** Potential for holes in the rendered image; less accurate than raycasting; requires tuning of projection size [S1, unofficial].
- **When to use:** Fast previewing in medical imaging systems where cutting operations are important [S1, unofficial].

## Background
Proposed by Sobierajski [116] as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is based on the principle of extracting only visible boundary voxels from volume data [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
