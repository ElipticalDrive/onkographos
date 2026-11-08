# VolSplat

Also called: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction

## Overview
VolSplat is a feed-forward method for 3D Gaussian Splatting (3DGS) that regresses Gaussian primitives directly from 3D features using a voxel-aligned prediction strategy [S1, unofficial]. It solves the limitations of pixel-aligned feed-forward methods, specifically the struggle to resolve multi-view alignment and the inability to adaptively control Gaussian density according to scene complexity [S1, unofficial]. By unprojecting 2D features into a 3D voxel grid, it achieves adaptive density control and robust multi-view consistency [S1, unofficial].

## How it works
**Inputs:** A set of $N$ multi-view images and their corresponding camera parameters.
**Outputs:** A set of 3D Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients) and rendered novel views.

**Data Structures:**
1.  **2D Feature Map:** Extracted per view, dimension $H \times W \times C_{2D}$.
2.  **Cost Volume:** Constructed via plane sweeping, dimension $H \times W \times D_{depth} \times C_{cost}$.
3.  **Depth Map:** Estimated per view, dimension $H \times W$.
4.  **Voxel Feature Grid:** 3D feature tensor, dimension $X \times Y \times Z \times C_{3D}$.
5.  **Gaussian Parameters:** Predicted per occupied voxel.

**Steps:**
1.  **2D Feature Extraction:** Extract 2D features for each input multi-view image using a Transformer-based network [S1, unofficial].
2.  **Cost Volume Construction:** Construct per-view cost volumes using plane sweeping [S1, unofficial].
3.  **Depth Estimation:** Estimate a depth map for each view using a Depth Prediction Module [S1, unofficial].
4.  **3D Unprojection:** Unproject the 2D features into 3D space using the estimated depth maps to form a voxel feature grid [S1, unofficial].
5.  **3D Feature Refinement:** Refine features in 3D space using a sparse 3D decoder [S1, unofficial].
6.  **Gaussian Prediction:** Predict the parameters of a 3D Gaussian for each occupied voxel in the refined grid [S1, unofficial].
7.  **Rendering:** Render novel views from the predicted 3D Gaussians using standard differentiable rasterization [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volsplat_forward(images, cameras):
    """
    images: List[Tensor], shape (N, C, H, W)
    cameras: List[Tensor], shape (N, 4, 4) or similar projection format
    Returns: rendered_images, gaussian_params
    """
    # Step 1: Extract 2D features
    # Using a Transformer-based network as described in [S1, unofficial]
    # Note: Specific architecture details (e.g., ViT backbone) are not provided in sources.
    # [G] Assuming a standard feature extractor outputting C_2D channels.
    features_2d = []
    for img in images:
        feat = transformer_encoder(img) 
        features_2d.append(feat)

    # Step 2: Construct per-view cost volumes with plane sweeping
    # [G] Plane sweeping typically involves back-projecting features to a set of 
    # discrete depth planes and aggregating similarity.
    cost_volumes = []
    for feat, cam in zip(features_2d, cameras):
        cv = plane_sweep_cost_volume(feat, cam)
        cost_volumes.append(cv)

    # Step 3: Estimate depth maps
    depth_maps = []
    for cv in cost_volumes:
        # Depth Prediction Module
        depth = depth_prediction_module(cv)
        depth_maps.append(depth)

    # Step 4: Unproject 2D features into 3D voxel grid
    # [G] Requires defining a 3D bounding box and voxel resolution (X, Y, Z).
    # Unprojection maps (x, y, z) in voxel space to (u, v) in image space using depth.
    voxel_grid_shape = (X, Y, Z)
    voxel_features = initialize_voxel_grid(voxel_grid_shape, C_3D)
    
    for feat, depth, cam in zip(features_2d, depth_maps, cameras):
        # Unproject feature from 2D image to 3D voxel based on depth
        voxel_features = unproject_features_to_voxels(
            voxel_features, feat, depth, cam, voxel_grid_shape
        )

    # Step 5: Refine features in 3D space
    # Using a sparse 3D decoder (e.g., Sparse Convolution)
    refined_voxel_features = sparse_3d_decoder(voxel_features)

    # Step 6: Predict Gaussian parameters for occupied voxels
    # [G] "Occupied" implies a threshold on feature magnitude or occupancy probability.
    gaussian_params = predict_gaussian_params(refined_voxel_features)

    # Step 7: Render novel views
    rendered_images = render_gaussians(gaussian_params, novel_view_cameras)
    
    return rendered_images, gaussian_params
```

**Parameters:**
*   **Voxel Resolution:** Not given in the sources.
*   **Depth Planes:** Not given in the sources.
*   **Transformer Backbone:** Not given in the sources.

**Edge Cases:**
*   **Occlusions:** The method relies on depth estimation; incorrect depth leads to misaligned features in the voxel grid [S1, unofficial].
*   **Empty Space:** The sparse 3D decoder handles empty voxels, but the definition of "occupied" is not explicitly detailed [S1, unofficial].

## Subtopics

### [VS-Splat](8542-vs-splat.md)
VS-Splat is a feed-forward framework for 3D object reconstruction from sparse-view 2D images that employs a coarse-to-fine (C2F) strategy to predict 3D Gaussian primitives. It distinguishes itself from sibling methods like LaRa and GenDen by performing voxel-level selection *before* generating fine primitives, rather than selecting existing primitives for refinement. This ensures that the coarse representation determines the spatial location for new fine primitives, enabling adaptive density control without auxiliary supervision.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Resolves multi-view alignment issues inherent in pixel-aligned methods; enables adaptive density control [S1, unofficial].
*   **Limitations:** Relies on accurate depth estimation; voxel resolution limits spatial precision [S1, unofficial].

## Background
VolSplat was proposed by ziplab and released on arXiv in September 2025 [S1, unofficial]. It addresses the limitations of pixel-aligned feed-forward 3DGS methods, which struggle with multi-view alignment and adaptive density control [S1, unofficial]. The method uses a voxel-aligned prediction strategy to regress Gaussians from 3D features [S1, unofficial].

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
