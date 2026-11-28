# Gaussian Splatting Rendering with Camera Pose Override

Also called: render function with rel_w2c transform

## Overview
This method modifies the standard 3D Gaussian Splatting rasterization pipeline to allow the camera pose to be overridden by an external tensor during rendering, rather than using the stored viewpoint parameters. It solves the problem of rendering a pre-trained Gaussian model from arbitrary or dynamically computed camera positions (e.g., from a SLAM system or a differentiable reconstruction loop) without retraining the model. It sets apart from standard rendering by injecting a custom world-to-camera transformation (`rel_w2c`) and quaternion rotation into the projection and covariance computation steps, enabling differentiable rendering with respect to the camera pose [S1, unofficial].

## How it works
**Inputs:**
- `pc`: The Gaussian Point Cloud object containing 3D means (`xyz`), rotations (quaternions), scales, opacities, and spherical harmonics (SH) coefficients.
- `camera_pose`: A tensor representing the override camera pose. Format depends on implementation, typically a 4x4 matrix or a quaternion + translation vector. In the reference implementation, it is used to construct a relative world-to-camera matrix [S1, unofficial].
- `viewpoint_camera`: The original viewpoint object containing intrinsic parameters (FoVx, FoVy, projection matrix) and resolution.
- `pipe`: A pipeline configuration object with boolean flags `compute_cov3D_python` and `convert_SHs_python` [S1, unofficial].
- `override_color`: Optional tensor to replace SH-based color computation [S1, unofficial].

**Outputs:**
- `render_image`: The rendered RGB image.
- `viewspace_points`: Screen-space points with gradient tracking enabled.
- `visibility_filter`: A boolean mask indicating which Gaussians have non-zero radii (are visible).
- `radii`: The screen-space radii of the Gaussians.

**Data Structures:**
- `screenspace_points`: A tensor of shape `(N, 3)` (where N is the number of Gaussians), initialized to zeros, with `requires_grad=True` and `retain_grad()` called to allow backpropagation to 2D means [S1, unofficial].
- `gaussians_xyz_trans`: Transformed 3D means in the camera frame.
- `gaussians_rot_trans`: Transformed rotations (quaternions) in the camera frame.

**Steps:**

1.  **Initialize Screen-Space Points:**
    Create a zero tensor for screen-space points with gradient tracking enabled. This allows the rasterizer to update these points during forward pass, which are then used for backpropagation [S1, unofficial].
    $$ \text{screenspace\_points} = \text{zeros}(\text{pc.get\_xyz.shape}, \text{dtype}=\text{pc.get\_xyz.dtype}, \text{requires\_grad}=\text{True}) $$
    $$ \text{screenspace\_points.retain\_grad()} $$

2.  **Compute Field of View Tangents:**
    Calculate the tangents of half the field of view angles from the `viewpoint_camera` intrinsics [S1, unofficial].
    $$ \text{tanfovx} = \tan(\text{viewpoint\_camera.FoVx} \times 0.5) $$
    $$ \text{tanfovy} = \tan(\text{viewpoint\_camera.FoVy} \times 0.5) $$

3.  **Construct Relative World-to-Camera Matrix:**
    The method constructs a relative world-to-camera matrix (`rel_w2c`) from the `camera_pose` input. In the reference implementation, this involves extracting the rotation and translation from the pose tensor [S1, unofficial].
    $$ \text{rel\_w2c} = \text{get\_camera\_from\_tensor}(\text{camera\_pose}) $$
    *Note: In some variants, if no override is needed, `w2c` is set to identity and combined with the viewpoint's projection matrix. However, with override, `rel_w2c` is derived directly from the pose tensor [S1, unofficial].*

4.  **Transform Gaussian Means:**
    Transform the 3D Gaussian means from world space to the camera space using the relative world-to-camera matrix. Homogeneous coordinates are used for the transformation [S1, unofficial].
    $$ \text{xyz\_homo} = [\text{pc.get\_xyz}, \mathbf{1}]^T $$
    $$ \text{gaussians\_xyz\_trans} = (\text{rel\_w2c} @ \text{xyz\_homo}.T).T[:, :3] $$

5.  **Transform Gaussian Rotations:**
    Transform the Gaussian rotations (stored as quaternions) from world space to the camera space. This is done by applying the rotation component of the `camera_pose` to the stored Gaussian rotations using quaternion multiplication [S1, unofficial].
    $$ \text{gaussians\_rot\_trans} = \text{quadmultiply}(\text{camera\_pose}[:4], \text{gaussians\_rot}) $$
    Where `quadmultiply` performs the rotation of one quaternion by another.

6.  **Determine Covariance Representation:**
    Check the `pipe.compute_cov3D_python` flag.
    - If `True`: Compute the 3D covariance matrix (`cov3D_precomp`) in Python using `pc.get_covariance(scaling_modifier)` [S1, unofficial].
    - If `False`: Pass the scales and the transformed rotations (`gaussians_rot_trans`) separately to the rasterizer, which computes the covariance on the GPU [S1, unofficial].

7.  **Determine Color Representation:**
    Check the `override_color` and `pipe.convert_SHs_python` flags.
    - If `override_color` is `None` and `pipe.convert_SHs_python` is `True`: Convert Spherical Harmonics (SH) coefficients to RGB in Python. This involves evaluating the SH function at the direction vectors from the camera to the Gaussian centers (`dir_pp_normalized`) and clamping the result to $[0, 1]$ [S1, unofficial].
    - Otherwise: Pass the SH coefficients directly to the rasterizer or use the `override_color` tensor if provided [S1, unofficial].

8.  **Rasterize:**
    Call the `GaussianRasterizer` with the transformed means (`means3D`), screen-space points (`means2D`), colors/SHs, opacities, and scales/rotations/covariances. The rasterizer returns the rendered image and the radii of the Gaussians [S1, unofficial].

9.  **Return Results:**
    Return the rendered image, viewspace points, visibility filter (where `radii > 0`), and radii [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_with_camera_pose_override(pc, camera_pose, viewpoint_camera, pipe, override_color=None):
    # 1. Initialize screen-space points for gradient tracking
    screenspace_points = torch.zeros_like(pc.get_xyz, dtype=pc.get_xyz.dtype, requires_grad=True, device='cuda') + 0
    screenspace_points.retain_grad()

    # 2. Compute FOV tangents
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)

    # 3. Construct relative world-to-camera matrix from pose tensor
    # get_camera_from_tensor extracts the 4x4 transformation matrix from the camera_pose tensor
    rel_w2c = get_camera_from_tensor(camera_pose)

    # 4. Transform 3D means to camera space
    # xyz_homo is (N, 4) with 1s in the last column
    xyz_homo = torch.cat([pc.get_xyz, torch.ones(pc.get_xyz.shape[0], 1, device='cuda')], dim=1)
    gaussians_xyz_trans = (rel_w2c @ xyz_homo.T).T[:, :3]

    # 5. Transform rotations (quaternions) to camera space
    # camera_pose[:4] contains the quaternion [w, x, y, z] or [x, y, z, w] depending on convention
    # quadmultiply applies the rotation of camera_pose to the Gaussian's rotation
    gaussians_rot_trans = quadmultiply(camera_pose[:4], pc.get_rotations())

    # 6. Determine covariance representation
    if pipe.compute_cov3D_python:
        cov3D_precomp = pc.get_covariance(scaling_modifier=1.0) # scaling_modifier default
    else:
        cov3D_precomp = None # Pass scales and rotations separately

    # 7. Determine color representation
    if override_color is None and pipe.convert_SHs_python:
        # Compute directions from camera to Gaussian centers in camera space
        # Note: dir_pp_normalized should be normalized direction vectors
        dir_pp = gaussians_xyz_trans - rel_w2c.inverse()[3, :3] # Camera position in world space transformed? 
        # Actually, in camera space, camera is at origin. So direction is just gaussians_xyz_trans normalized
        dir_pp_normalized = torch.nn.functional.normalize(gaussians_xyz_trans, dim=1)
        
        # Evaluate SHs
        shs_view = pc.get_SHs() # Shape (N, K, 3)
        sh_degree = pc.active_sh_degree
        colors_precomp = eval_sh(sh_degree, shs_view, dir_pp_normalized)
        colors_precomp = torch.clamp(colors_precomp, 0.0, 1.0)
    else:
        colors_precomp = override_color if override_color is not None else pc.get_SHs()

    # 8. Rasterize
    # GaussianRasterizer expects specific arguments based on pipe settings
    rasterizer = GaussianRasterizer(
        raster_settings={
            'tanfovx': tanfovx,
            'tanfovy': tanfovy,
            'image_height': viewpoint_camera.image_height,
            'image_width': viewpoint_camera.image_width,
            'sh_degree': pc.active_sh_degree,
            'near': viewpoint_camera.near,
            'far': viewpoint_camera.far
        }
    )
    
    rendered_image, radii = rasterizer(
        means3D=gaussians_xyz_trans,
        means2D=screenspace_points,
        shs=colors_precomp if not pipe.convert_SHs_python else None,
        colors_precomp=colors_precomp if pipe.convert_SHs_python else None,
        opacities=pc.get_opacities(),
        scales=pc.get_scales() if not pipe.compute_cov3D_python else None,
        rotations=gaussians_rot_trans if not pipe.compute_cov3D_python else None,
        cov3D_precomp=cov3D_precomp if pipe.compute_cov3D_python else None
    )

    # 9. Return results
    visibility_filter = radii > 0
    return rendered_image, screenspace_points, visibility_filter, radii
```

**Parameters:**
- `camera_pose`: The override pose. Must be compatible with `get_camera_from_tensor` [S1, unofficial].
- `pipe.compute_cov3D_python`: Boolean. If true, covariance is computed in Python; otherwise, on GPU [S1, unofficial].
- `pipe.convert_SHs_python`: Boolean. If true, SH to RGB conversion is done in Python; otherwise, on GPU [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Gradient Flow:** `screenspace_points` must have `requires_grad=True` and `retain_grad()` called to ensure gradients can flow back to the 2D means during backpropagation [S1, unofficial].
- **Rotation Convention:** `quadmultiply` must handle the specific quaternion convention (wxyz vs xyzw) used by the Gaussian storage and the `camera_pose` tensor [S1, unofficial].
- **Direction Vectors:** When computing SH colors in Python, the direction vectors must be normalized and computed in the correct coordinate system (camera space) [S1, unofficial].

## Subtopics

### 3D Mask Representation (3D-GS)
This variant integrates a mask confidence score $m_{gi}$ as a new attribute for each 3D Gaussian $g_i$, initialized to zero. The Gaussian is wholly assigned to either the segmentation target or the background. The mask rendering process uses the differentiable rasterization algorithm:
$$ M(\mathbf{r}) = \sum_{i=1}^{|G_r|} \omega_{gi} m_{gi} $$
where $\omega_{gi}$ is the weight of the Gaussian and $m_{gi}$ is its mask confidence score [S2]. This approach is more memory-efficient than using voxel grids (10 million Gaussians vs 32.77 million entries for a $320^3$ grid) and has lower computation as no additional querying is needed [S2].

## Operating characteristics
- **Time Complexity:** Dominated by the rasterization step, which is $O(N)$ for $N$ Gaussians, but depends on the number of visible Gaussians per tile. The overhead of transforming means and rotations is $O(N)$.
- **Memory Complexity:** $O(N)$ for storing Gaussian attributes. The mask representation adds $O(N)$ memory for the mask confidence score [S2].
- **Strengths:** Allows rendering from arbitrary camera poses without retraining. The mask representation is memory-efficient and computationally lightweight compared to grid-based methods [S2].
- **Limitations:** Requires careful handling of coordinate transformations and quaternion rotations. The Python-based SH and covariance computations (if enabled) may be slower than GPU-based equivalents [S1, unofficial].

## Background
This method is based on the implementation in NVlabs/InstantSplat, which provides a rendering function that overrides the camera pose [S1, S1, unofficial]. The concept of integrating mask attributes into 3D Gaussians for segmentation is described in "Segment Anything in 3D with Radiance Fields" [S2].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] Segment Anything in 3D with Radiance Fields, https://doi.org/10.1007/s11263-025-02421-7, fetched 2026-10-26
