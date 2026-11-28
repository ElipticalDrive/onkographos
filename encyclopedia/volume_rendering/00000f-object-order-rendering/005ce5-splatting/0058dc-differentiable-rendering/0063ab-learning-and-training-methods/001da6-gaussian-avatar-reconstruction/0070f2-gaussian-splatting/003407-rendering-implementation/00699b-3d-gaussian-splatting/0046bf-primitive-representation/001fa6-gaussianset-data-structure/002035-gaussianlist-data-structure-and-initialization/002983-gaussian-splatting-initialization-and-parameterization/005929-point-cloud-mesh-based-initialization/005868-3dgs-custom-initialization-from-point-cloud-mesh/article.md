# 3DGS Custom Initialization from Point Cloud/Mesh

Also called: 3DGS Model Learning Initialization from Point Cloud/Mesh, Custom Initialisation for 3DGS from Point Cloud/Mesh, Custom Initialization from Point Cloud/Mesh, Geometry-guided 3DGS Initialization, Point Cloud/Mesh Initialization for 3DGS

## Overview
This method initializes 3D Gaussian Splatting (3DGS) learning using a pre-existing dense 3D point cloud or mesh as the geometric prior, replacing the standard Structure-from-Motion (SfM) sparse point cloud initialization [S1, unofficial]. It solves the problem of creating high-quality 3DGS models from volumetric data (e.g., MPEG plenoptic point clouds or meshes) when the original multi-view images used for capture are unavailable [S1, unofficial]. Unlike standard 3DGS which relies on sparse SfM points and aggressive adaptive density control (leading to potential misalignment and 'floater' artifacts), this method uses dense, surface-aligned initial points, resulting in cleaner models, fewer splats, and significantly faster convergence (e.g., 5,000 vs. 15,000 iterations) [S1, unofficial].

## How it works
**Inputs:**
1. **Geometry:** A 3D point cloud or mesh [S1, unofficial]. For point clouds, the vertices $\{p_i\}$ are used [S1, unofficial]. For meshes, either the vertices or the center point of each mesh face (or a subset thereof) is used [S1, unofficial].
2. **Color Data:** View-dependent colors associated with the geometry (e.g., from a plenoptic point cloud) [S1, unofficial]. If the input is a mesh, colors may be sampled or rendered [S1, unofficial].
3. **Camera Parameters:** Extrinsic and intrinsic parameters for a set of viewpoints [S1, unofficial]. These are used to render multi-view images from the input geometry and must be transformed to the target coordinate system (e.g., COLMAP format) [S1, unofficial].

**Outputs:**
A 3DGS model (set of Gaussians) initialized with centers on the input geometry, ready for optimization [S1, unofficial].

**Data Structures:**
The method populates the standard 3DGS Gaussian attributes:
* `_xyz`: Initialized to the $(x, y, z)$ coordinates of the selected input points [G].
* `_features_dc`: Initialized to the RGB color of the corresponding point [G].
* `_features_rest`: Zero-initialized [G].
* `_opacity`: Initialized to a low value (raw parameter $\approx -2.197$, corresponding to opacity $\approx 0.1$) [G].
* `_scaling`: Derived from local density or depth heuristics (e.g., median distance to $k$-nearest neighbors or isometric radius based on focal length and depth) [G].
* `_rotation`: Initialized to identity quaternion $(1, 0, 0, 0)$ for all Gaussians [G].

**Steps:**
1. **Render Multi-View Images:** Generate multi-view images from the input 3D point cloud or mesh using a renderer (e.g., PccAppRenderer) [S1, unofficial].
2. **Align Camera Parameters:** Obtain camera parameters (extrinsic and intrinsic) for the viewpoints used to generate the images [S1, unofficial]. Transform these parameters to the target coordinate system (e.g., COLMAP) [S1, unofficial].
3. **Select Initial Points:**
   * If input is a **point cloud**: Use the vertices $(x, y, z)$ as the initial Gaussian centers [S1, unofficial].
   * If input is a **mesh**: Use the vertices or the center point of each mesh face (or a chosen subset of faces) as the initial Gaussian centers [S1, unofficial].
4. **Initialize Gaussian Attributes:**
   * Set `_xyz` to the selected points [G].
   * Set `_features_dc` to the RGB color of the points [G].
   * Set `_features_rest` to zero [G].
   * Set `_opacity` to a low initial value (e.g., raw parameter $-2.197$) [G].
   * Set `_rotation` to identity $(1, 0, 0, 0)$ [G].
   * Compute `_scaling` using heuristics such as median distance to $k$-nearest neighbors or isometric radius [G].
5. **Configure Optimization Constraints:**
   * **Deactivate Adaptive Density Control:** If the input point cloud/mesh is already dense and clean, deactivate cloning/splitting of splats to preserve the initial point density and reduce learning time [S1, unofficial].
   * **Freeze Position Learning Rate:** Set the position learning rate for the splat centers to 0 [S1, unofficial]. Retain a learning rate $> 0$ for other attributes (opacity, scale, rotation, color) to allow the splats to adapt their appearance and shape without moving from their geometric centers [S1, unofficial].
6. **Optimize:** Run the 3DGS optimization loop for the specified number of iterations (e.g., 5,000) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_3dgs_from_geometry(centers, colors, camera_params, renderer, k=10):
    """
    Initialize 3DGS model from a point cloud or mesh.
    
    Args:
        centers: Numpy array of shape (N, 3) containing vertex or face center coordinates.
        colors: Numpy array of shape (N, 3) containing RGB values.
        camera_params: List of camera extrinsics and intrinsics in target coordinate system.
        renderer: Renderer instance capable of rendering geometry to images.
        k: Number of nearest neighbors for scale heuristic.
        
    Returns:
        Dictionary containing initialized Gaussian attributes and optimization settings.
    """
    # Step 1: Render multi-view images for optimization targets
    target_images = []
    for cam in camera_params:
        img = renderer.render(centers, cam)
        target_images.append(img)

    # Step 3: Select initial points (vertices or face centers)
    # centers and colors are assumed to be numpy arrays of shape (N, 3)
    num_gaussians = len(centers)

    # Step 4: Initialize Gaussian Attributes
    # _xyz: Centers from geometry
    xyz = centers

    # _features_dc: RGB colors
    features_dc = colors.reshape(-1, 3)

    # _features_rest: Zero-initialized
    features_rest = np.zeros((num_gaussians, 15, 3))

    # _opacity: Low initial value
    # Raw parameter -2.197 corresponds to sigmoid(-2.197) ≈ 0.1
    opacity = np.full((num_gaussians, 1), -2.197)

    # _rotation: Identity quaternion (w, x, y, z)
    rotation = np.tile([1.0, 0.0, 0.0, 0.0], (num_gaussians, 1))

    # _scaling: Heuristic based on local density
    # Compute median distance to k-nearest neighbors
    dists = np.linalg.norm(centers[:, np.newaxis, :] - centers[np.newaxis, :, :], axis=2)
    np.fill_diagonal(dists, np.inf)
    sorted_dists = np.sort(dists, axis=1)
    scales = np.median(sorted_dists[:, :k], axis=1)
    # Convert distance to scale parameter (log scale)
    scaling = np.log(scales)

    # Step 5: Configure Optimization
    # Create model object
    model = {
        'xyz': xyz,
        'features_dc': features_dc,
        'features_rest': features_rest,
        'opacity': opacity,
        'rotation': rotation,
        'scaling': scaling
    }

    # Set learning rates
    model['position_lr'] = 0.0  # Freeze positions
    model['other_lr'] = 1e-3    # Allow adaptation of other attributes

    # Deactivate adaptive density control if geometry is dense/clean
    # Source does not specify exact density/cleanliness check
    model['adaptive_density_control'] = False

    return model
```

**Edge Cases and Pitfalls:**
* **Noisy/Sparse Input:** If the input 3D model is noisy, sparse, or has holes, deactivating adaptive density control should be avoided [S1, unofficial]. In such cases, the default SfM initialization may be preferred, or adaptive control should remain active to allow the model to fill gaps and correct errors [S1, unofficial].
* **Coordinate Systems:** Camera parameters must be correctly transformed to the target coordinate system (e.g., COLMAP) to ensure alignment between rendered images and the geometry [S1, unofficial].
* **View-Dependent Colors:** The method benefits significantly from input data containing view-dependent colors (e.g., plenoptic point clouds), as 3DGS is effective for non-Lambertian surfaces [S1, unofficial].

## Subtopics

### Feature-Aware Point Cloud Initialization
This variant initializes 3D Gaussian primitives from a feature-aware point cloud to provide a starting point close to the final solution, avoiding convergence issues and scale ambiguity [S2]. The position $\mu$ is initialized directly from the point's coordinate [S2]. The shape $\Sigma$ is initialized as a small isotropic sphere [S2]. The color is represented by Spherical Harmonics (SH) coefficients, with the DC term initialized from the point's RGB value [S2]. The opacity $\alpha$ is initialized to a high value [S2].

## Operating characteristics
* **Convergence Speed:** Significantly faster than SfM-based initialization [S1, unofficial]. In tests on the Thaidancer sequence, custom initialization converged in 5,000 iterations compared to 15,000 iterations for SfM initialization, with comparable visual quality [S1, unofficial].
* **Model Size:** Produces fewer splats than the original point cloud density [S1, unofficial]. For the Thaidancer sequence, the original point cloud was ~3M voxels (201MB PLY), while the resulting 3DGS model had ~250k splats (61MB PLY) [S1, unofficial].
* **Quality:** Produces cleaner surface representation with fewer 'floater' artifacts compared to SfM initialization, which often requires adaptive density control that can lead to misalignment [S1, unofficial].
* **Limitations:** Requires a pre-existing dense and clean point cloud or mesh [S1, unofficial]. If the input geometry is poor quality, the method's constraints (frozen positions, no densification) may lead to suboptimal results [S1, unofficial].
* **Hardware/Resolution:** Performance scales with the number of splats and iterations [S1, unofficial]. The reduction in iterations (5k vs 15k) directly reduces training time [S1, unofficial].

## Background
This method was proposed in the context of transcoding 3D Gaussian Splatting models from existing 3D plenoptic point clouds or meshes when original multi-view images are unavailable [S1, unofficial]. It addresses the need to leverage traditional volumetric media formats (like those in MPEG codec standardization) with the emerging popularity of 3DGS [S1, unofficial]. The approach was tested on the MPEG plenoptic point cloud dataset, specifically the "8i Voxelized Surface Light Field" (8iVSLF) [S1, unofficial].

## Sources
[S1] Transcoding a 3D Gaussian Splatting Model from a Plenoptic Point Cloud or Mesh without the Original Multi-view Images, https://arxiv.org/abs/2605.21051, fetched 2026-10-30 (unofficial)
[S2] A dual-prior driven Gaussian splatting framework for high-fidelity reconstruction of museum artifacts, https://doi.org/10.1038/s40494-026-02330-z, fetched 2026-10-30
