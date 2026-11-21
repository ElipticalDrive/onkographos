# Projected Voxel Size / LOD Calculation

Also called: ComputeVoxelSize

## Overview
Projected Voxel Size / LOD Calculation computes a scalar metric representing the projected size of a voxel relative to the camera, primarily used for Level-of-Detail (LOD) decisions and step size adaptation in volume rendering [S1, unofficial]. It differs from standard distance-field ray marching step rules by explicitly evaluating the angular span of a voxel's extent from the camera center to determine sampling density, rather than relying solely on Euclidean distance or fixed grid spacing.

## How it works
**Inputs and Outputs:**
- Input: Camera position ($\mathbf{C}$), voxel center ($\mathbf{V}$), voxel half-size vector ($\mathbf{h}$), and camera orientation basis vectors (forward $\mathbf{f}$, right $\mathbf{r}$, up $\mathbf{u}$).
- Output: A scalar projected size metric $S$.

**Data Structures:**
- Voxel geometry is defined by its center $\mathbf{V}$ and half-extents $\mathbf{h} = (h_x, h_y, h_z)$.
- Camera geometry is defined by origin $\mathbf{C}$ and normalized direction vectors.

**Steps:**
1.  Define the eight corners of the voxel bounding box. For a voxel centered at $\mathbf{V}$ with half-size $\mathbf{h}$, the corners are $\mathbf{V} \pm h_x \mathbf{e}_x \pm h_y \mathbf{e}_y \pm h_z \mathbf{e}_z$, where $\mathbf{e}_i$ are the local axis vectors.
2.  Identify the two corners that define the maximal angular separation from the camera. The source notes specify computing vectors to the "lower-left" (LL) and "upper-right" (UR) corners. In a canonical view where the voxel is axis-aligned and the camera looks along the Z-axis, these correspond to the corners with minimal and maximal screen-space coordinates.
    -   $\mathbf{P}_{LL} = \mathbf{V} - h_x \mathbf{e}_x - h_y \mathbf{e}_y + h_z \mathbf{e}_z$ (assuming standard coordinate conventions where Z is depth).
    -   $\mathbf{P}_{UR} = \mathbf{V} + h_x \mathbf{e}_x + h_y \mathbf{e}_y + h_z \mathbf{e}_z$.
    -   *Note:* The specific selection of LL and UR depends on the camera's view matrix. Generally, these are the corners that maximize the dot product difference or angular span.
3.  Compute the vectors from the camera position $\mathbf{C}$ to these corners:
    -   $\mathbf{v}_{LL} = \mathbf{P}_{LL} - \mathbf{C}$
    -   $\mathbf{v}_{UR} = \mathbf{P}_{UR} - \mathbf{C}$
4.  Normalize these vectors:
    -   $\mathbf{u}_{LL} = \frac{\mathbf{v}_{LL}}{\|\mathbf{v}_{LL}\|}$
    -   $\mathbf{u}_{UR} = \frac{\mathbf{v}_{UR}}{\|\mathbf{v}_{UR}\|}$
5.  Calculate the projected size metric $S$ using the dot product:
    $$ S = 1 - \mathbf{u}_{LL} \cdot \mathbf{u}_{UR} $$

**Formula:**
$$ S = 1 - \mathbf{u}_{LL} \cdot \mathbf{u}_{UR} $$
Where:
-   $S$ is the projected size metric.
-   $\mathbf{u}_{LL}$ is the normalized vector from camera to the lower-left corner of the voxel.
-   $\mathbf{u}_{UR}$ is the normalized vector from camera to the upper-right corner of the voxel.

**Parameters:**
-   Not given in the sources.

**Edge Cases and Pitfalls:**
-   **Camera at Voxel Center:** If the camera is located exactly at the voxel center, $\mathbf{v}_{LL}$ and $\mathbf{v}_{UR}$ are opposite vectors (or identical if corners are misidentified), leading to $\mathbf{u}_{LL} \cdot \mathbf{u}_{UR} = -1$ or $1$. The source notes state the metric returns 0 when the dot product is 1, implying the camera is looking directly at a corner or the vectors are identical. If $\mathbf{u}_{LL} = \mathbf{u}_{UR}$, $S=0$. This is a degenerate case.
-   **Near-Plane Clipping:** The metric lacks explicit clamping or fallback for degenerate near-plane cases where the voxel intersects the camera plane. Division by zero in normalization must be guarded [G].
-   **Corner Selection:** The "lower-left" and "upper-right" labels are view-dependent. In a general implementation, one must project all 8 corners to screen space or compute the angular span correctly to ensure the maximal angle is used for conservative LOD estimation. The source notes simplify this to LL and UR, assuming a specific canonical alignment [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_projected_voxel_size(camera_pos, voxel_center, voxel_half_size, camera_forward, camera_right, camera_up):
    """
    Computes the projected size metric for a voxel.
    
    Args:
        camera_pos: (x, y, z) camera position.
        voxel_center: (x, y, z) voxel center.
        voxel_half_size: (x, y, z) half-extents of the voxel.
        camera_forward: (x, y, z) normalized forward vector.
        camera_right: (x, y, z) normalized right vector.
        camera_up: (x, y, z) normalized up vector.
        
    Returns:
        S: Scalar projected size metric.
    """
    # Define local axes of the voxel (assuming axis-aligned for simplicity as per source context)
    # If voxel is rotated, these would be the voxel's local axes.
    # The source implies a canonical view where LL and UR are well-defined.
    # We assume the voxel axes align with the camera axes for the LL/UR definition,
    # or that LL/UR refer to the corners that maximize angular separation.
    
    # Calculate the 8 corners of the voxel
    corners = []
    for dx in [-1, 1]:
        for dy in [-1, 1]:
            for dz in [-1, 1]:
                corner = (
                    voxel_center[0] + dx * voxel_half_size[0],
                    voxel_center[1] + dy * voxel_half_size[1],
                    voxel_center[2] + dz * voxel_half_size[2]
                )
                corners.append(corner)
    
    # Project corners to screen space to find LL and UR
    # This is a simplification; the source notes directly use LL and UR vectors.
    # In a general case, we project all corners and find the min/max screen coords.
    # For this implementation, we assume the source's LL/UR refers to the corners
    # that are extremal in the view direction.
    
    # To be robust, we calculate vectors to all corners and find the pair with the
    # minimum dot product (max angle).
    
    min_dot = 2.0
    best_u1 = None
    best_u2 = None
    
    for i in range(len(corners)):
        v1 = (
            corners[i][0] - camera_pos[0],
            corners[i][1] - camera_pos[1],
            corners[i][2] - camera_pos[2]
        )
        len_v1 = math.sqrt(v1[0]**2 + v1[1]**2 + v1[2]**2)
        if len_v1 == 0:
            continue # Degenerate case
        u1 = (v1[0]/len_v1, v1[1]/len_v1, v1[2]/len_v1)
        
        for j in range(i + 1, len(corners)):
            v2 = (
                corners[j][0] - camera_pos[0],
                corners[j][1] - camera_pos[1],
                corners[j][2] - camera_pos[2]
            )
            len_v2 = math.sqrt(v2[0]**2 + v2[1]**2 + v2[2]**2)
            if len_v2 == 0:
                continue # Degenerate case
            u2 = (v2[0]/len_v2, v2[1]/len_v2, v2[2]/len_v2)
            
            dot = u1[0]*u2[0] + u1[1]*u2[1] + u1[2]*u2[2]
            if dot < min_dot:
                min_dot = dot
                best_u1 = u1
                best_u2 = u2
                
    if best_u1 is None or best_u2 is None:
        return 0.0 # Fallback for degenerate case
        
    S = 1.0 - min_dot
    return S
```

## Subtopics

### Depth-Dependent Spatial Resolution (Voxel Sizing)
Determines the minimum reproducible voxel size in a light field display based on distance from the screen and beam angular divergence, dictating volume sampling rates [S2]. It differs from standard projected voxel size calculations by incorporating the physical properties of the light field display, specifically the angular size of light beams, to define a distance-dependent resolution limit. It is chosen when rendering for light field displays where spatial resolution degrades with depth. The formula is $s(z) = s_0 + 2|z| \tan(\Phi/2)$, where $s(z)$ is the minimum voxel size at distance $z$ from the screen, $s_0$ is the pixel size on the screen surface, and $\Phi$ is the finite angular size of light beams leaving the screen [S2].

### Max Sample Rate Calculation
Calculates the maximum sampling rate across all cameras for each voxel, based on voxel length and sample interval derived from depth [S3, unofficial]. It differs by aggregating sampling requirements from multiple camera views to determine a global maximum rate for a voxel. It is chosen in multi-view reconstruction or training scenarios where voxel density must be consistent across all viewpoints. The method initializes a max sampling rate tensor, calculates the z-distance from voxel centers to camera positions, filters visible voxels, computes sample intervals based on pixel size and depth, and updates the maximum rate [S3, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(1)$ per voxel for the basic projected size calculation [S1, unofficial]. The max sample rate calculation is $O(N)$ where $N$ is the number of voxels, with additional cost proportional to the number of cameras [S3, unofficial].
-   **Memory Complexity:** $O(1)$ for the basic metric. The max sample rate calculation requires $O(N)$ storage for the rate tensor [S3, unofficial].
-   **Strengths:** Provides a geometrically intuitive measure of voxel importance relative to the viewer. Adapts sampling density to viewing angle.
-   **Limitations:** The basic metric lacks robustness for near-plane cases and requires careful corner selection [S1, unofficial]. The max sample rate calculation is dependent on the visibility of voxels across multiple views [S3, unofficial].
-   **When to use:** Use for LOD management in volume rendering where angular coverage is a key factor. Use max sample rate for multi-view consistent voxelization.

## Background
The concept of projected voxel size for LOD is discussed in the context of voxel visualization and ray marching [S1, unofficial]. The depth-dependent spatial resolution is derived from research on GPU-accelerated direct volume rendering on interactive light field displays [S2]. The max sample rate calculation is part of a sparse voxel rasterization pipeline for neural rendering or TSDF fusion [S3, unofficial].

## Sources
[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S2] GPU Accelerated Direct Volume Rendering on an Interactive Light Field Display, https://doi.org/10.1111/j.1467-8659.2008.01120.x, fetched 2026-10-29
[S3] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
