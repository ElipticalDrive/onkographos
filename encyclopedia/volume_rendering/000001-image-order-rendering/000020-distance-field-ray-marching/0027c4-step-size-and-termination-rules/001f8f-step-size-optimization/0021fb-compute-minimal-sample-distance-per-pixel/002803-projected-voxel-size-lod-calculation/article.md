# Projected Voxel Size / LOD Calculation

Also called: ComputeVoxelSize

## Overview
This method computes a scalar metric representing the projected size of a voxel relative to the camera, primarily used for Level of Detail (LOD) decisions and step size adaptation in distance-field ray marching [S1, unofficial]. It evaluates the angular spread of the voxel as seen from the camera center to provide a normalized measure of voxel appearance in viewing space [S1, unofficial].

## How it works
**Inputs:**
- Camera position $\mathbf{C}$ (world space) [G].
- Voxel geometry defined by its center $\mathbf{V}_c$ and half-extents $\mathbf{h} = (h_x, h_y, h_z)$ in world space [G].
- The voxel is treated as an axis-aligned box in its local space, transformed to world space [G].

**Outputs:**
- A scalar projected size metric $S$.

**Data Structures:**
- Vectors $\mathbf{u}_{LL}$ and $\mathbf{u}_{UR}$: Normalized direction vectors from the camera to the voxel corners [S1, unofficial].

**Steps:**
1. Identify the lower-left and upper-right corners of the voxel's half-size box in world space [S1, unofficial]. Let $\mathbf{P}_{LL}$ be the position of the lower-left corner and $\mathbf{P}_{UR}$ be the position of the upper-right corner of the voxel's bounding box [S1, unofficial].
2. Compute vectors from the camera position $\mathbf{C}$ to these corners: $\mathbf{v}_{LL} = \mathbf{P}_{LL} - \mathbf{C}$ and $\mathbf{v}_{UR} = \mathbf{P}_{UR} - \mathbf{C}$ [S1, unofficial].
3. Normalize these vectors to obtain unit direction vectors: $\mathbf{u}_{LL} = \mathbf{v}_{LL} / \|\mathbf{v}_{LL}\|$ and $\mathbf{u}_{UR} = \mathbf{v}_{UR} / \|\mathbf{v}_{UR}\|$ [S1, unofficial].
4. Compute the dot product of the two normalized vectors [S1, unofficial].
5. Calculate the projected size metric $S$ using the formula: $S = 1 - \mathbf{u}_{LL} \cdot \mathbf{u}_{UR}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_projected_voxel_size(camera_pos, voxel_center, voxel_half_extents):
    # 1. Determine lower-left and upper-right corners in world space
    corner_ll = voxel_center - voxel_half_extents
    corner_ur = voxel_center + voxel_half_extents
    
    # 2. Vectors from camera to corners
    vec_ll = corner_ll - camera_pos
    vec_ur = corner_ur - camera_pos
    
    # 3. Normalize vectors
    len_ll = length(vec_ll)
    len_ur = length(vec_ur)
    
    # Guard against degenerate zero-length vectors
    if len_ll < 1e-9 or len_ur < 1e-9:
        return 0.0
            
    u_ll = vec_ll / len_ll
    u_ur = vec_ur / len_ur
    
    # 4. Dot product
    dot_product = dot(u_ll, u_ur)
    
    # 5. Projected size metric
    S = 1.0 - dot_product
    
    return S
```

**Parameters:**
- No explicit tuning parameters are defined in the core method [S1, unofficial].

**Edge Cases and Pitfalls:**
- The metric returns $0$ when the camera is at the voxel center (or coincident with a corner such that vectors align perfectly), because the dot product becomes $1$ [S1, unofficial].
- The method lacks explicit clamping or fallback for degenerate near-plane cases in the source implementation [S1, unofficial].
- Precision issues may arise if the camera is extremely close to the voxel, requiring a minimum length guard before normalization [G].

## Subtopics

### Depth-Dependent Spatial Resolution (Voxel Sizing)
This variant determines the minimum reproducible voxel size in a light field display based on the distance from the screen and beam angular divergence, dictating volume sampling rates [S2]. It is chosen when rendering for light field displays where physical resolution limits vary with depth [S2]. The metric $s(z)$ is calculated as $s(z) = s_0 + 2|z| \tan(\Phi/2)$, where $s(z)$ is the minimum voxel size at distance $z$ from the screen, $s_0$ is the pixel size on the screen surface, and $\Phi$ is the finite angular size of light beams leaving the screen [S2]. This approach limits the field-of-depth and dictates sampling step sizes to match the display's physical resolution [S2].

### MIP Scale Computation
This variant computes a per-Gaussian scale factor based on camera distance and focal length, typically used for anti-aliasing or level-of-detail adjustments in Gaussian splatting [S3, unofficial]. It calculates the Euclidean distance from the camera center to each Gaussian, converts the horizontal field of view and image width to a focal length, and returns the ratio of distance to focal length as the scale factor [S3, unofficial]. The implementation computes `dist_pp = (pc.get_xyz - viewpoint_camera.camera_center).norm(dim=1, keepdim=True)`, derives `focal = fov2focal(viewpoint_camera.FoVx, viewpoint_camera.image_width)`, and returns `dist_pp / focal` [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel, involving basic vector arithmetic and a dot product [G].
- **Memory Complexity:** $O(1)$ auxiliary space [G].
- **Strengths:** Provides a simple, normalized metric for angular size suitable for LOD decisions [S1, unofficial].
- **Limitations:** Does not account for screen-space pixel coverage directly, only angular spread [S1, unofficial]. Lacks robustness for degenerate cases (camera inside voxel) without additional guards [S1, unofficial].
- **When to use:** When LOD or step size adaptation is driven by the angular appearance of voxels in distance-field ray marching [S1, unofficial].
- **When not to use:** When rendering for light field displays requiring physical resolution limits based on screen distance [S2], or when screen-space coverage is the primary concern.

## Background
Not given in the sources.

## Sources
[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S2] GPU Accelerated Direct Volume Rendering on an Interactive Light Field Display, https://doi.org/10.1111/j.1467-8659.2008.01120.x, fetched 2026-10-29
[S3] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
