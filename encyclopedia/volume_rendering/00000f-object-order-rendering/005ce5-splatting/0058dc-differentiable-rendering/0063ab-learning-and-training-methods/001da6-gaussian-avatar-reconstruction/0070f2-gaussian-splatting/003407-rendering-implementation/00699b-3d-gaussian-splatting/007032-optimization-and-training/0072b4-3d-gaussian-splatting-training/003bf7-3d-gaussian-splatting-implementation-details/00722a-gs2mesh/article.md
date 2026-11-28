# GS2Mesh

Also called: Surface Reconstruction from Gaussian Splatting via Novel Stereo Views

## Overview
GS2Mesh is a post-processing method that extracts a smooth, triangulated 3D mesh from a trained 3D Gaussian Splatting (3DGS) representation by leveraging a pre-trained stereo-matching model [S1]. Unlike methods that enforce geometric constraints during the Gaussian optimization phase (e.g., [SuGaR](12925-desplat.md), [2DGS](12929-360-gaussian-splatting.md), [GOF](15421-scene-representation-and-geometry-constraints.md)), which often yield noisy surfaces because Gaussian centers are optimized for photometric consistency rather than geometric accuracy, GS2Mesh bypasses the noisy Gaussian locations entirely [S1]. It generates artificial stereo-aligned image pairs from the 3DGS model, extracts depth via a shape-from-stereo algorithm, and fuses these depths into a mesh using a Truncated Signed Distance Function (TSDF) [S1].

## How it works
**Inputs:**
1. A trained 3D Gaussian Splatting model (defined by the set of Gaussians $\mathcal{G}$, including means $\mu_i$, covariances $\Sigma_i$, opacities $\alpha_i$, and spherical harmonics coefficients for color) [S1].
2. The camera poses and intrinsics used during the 3DGS training [S1].
3. A pre-trained stereo-matching model (shape-from-stereo algorithm) capable of outputting dense depth maps from stereo image pairs [S1].
4. A TSDF fusion implementation [S1].

**Outputs:**
A triangulated mesh representing the scene surface [S1].

**Data Structures:**
*   **3DGS Model:** Standard 3DGS representation. No modification to the Gaussian parameters is required [S1].
*   **Stereo Pairs:** Two rendered images $I_L$ and $I_R$ with known baseline and epipolar geometry [S1].
*   **Depth Maps:** Dense RGB-D structures $D_L, D_R$ corresponding to the stereo pairs [S1].
*   **TSDF Volume:** A 3D voxel grid storing signed distance values and weights for fusion [S1].

**Steps:**
1.  **Scene Representation:** Apply the trained 3DGS model to the input images/video to obtain the optimized Gaussian distribution [S1].
2.  **Stereo Pair Rendering:** Select pairs of original training views that are sufficiently separated in baseline to provide good stereo disparity. Render images from these poses using the 3DGS model to create stereo-aligned pairs $(I_L, I_R)$ [S1].
3.  **Stereo Depth Estimation:** Feed each stereo-aligned pair $(I_L, I_R)$ into the pre-trained stereo-matching model. The model solves the correspondence problem to reconstruct an RGB-D structure (depth profile) for each view [S1].
4.  **TSDF Fusion:** Integrate all reconstructed depth profiles from all selected views into a single 3D volume using the Truncated Signed Distance Function (TSDF) algorithm [S1].
5.  **Mesh Extraction:** Extract the zero-level set from the fused TSDF volume to generate a triangulated mesh of the scene [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gs2mesh(trained_3dgs_model, training_cameras, stereo_model, tsdf_fuser):
    """
    Extracts a mesh from a 3DGS model using stereo matching and TSDF fusion.
    
    Args:
        trained_3dgs_model: The optimized 3D Gaussian Splatting representation.
        training_cameras: List of camera poses and intrinsics from training.
        stereo_model: Pre-trained stereo-matching model (e.g., RAFT-Stereo, CREStereo).
        tsdf_fuser: TSDF integration object.
        
    Returns:
        mesh: A triangulated mesh object.
    """
    # Step 1: Select stereo pairs from training cameras
    # Heuristic: Select pairs with sufficient baseline and overlap.
    # Not given in sources: specific selection algorithm.
    stereo_pairs = select_stereo_pairs(training_cameras)
    
    # Step 2: Render stereo-aligned image pairs
    rendered_pairs = []
    for cam_left, cam_right in stereo_pairs:
        # Render images using 3DGS rasterization
        img_left = render_3dgs(trained_3dgs_model, cam_left)
        img_right = render_3dgs(trained_3dgs_model, cam_right)
        rendered_pairs.append((img_left, img_right, cam_left, cam_right))
        
    # Step 3: Estimate depth from stereo pairs
    depth_maps = []
    for img_l, img_r, cam_l, cam_r in rendered_pairs:
        # Use pre-trained stereo model to get depth
        # The stereo model expects rectified or epipolar-aligned images.
        # If cameras are not rectified, rectification is required.
        # Not given in sources: specific rectification steps.
        depth_map = stereo_model.compute_depth(img_l, img_r)
        depth_maps.append((depth_map, cam_l, cam_r))
        
    # Step 4: Fuse depth maps into TSDF volume
    for depth_map, cam_l, cam_r in depth_maps:
        # Project depth map into 3D space relative to camera frame
        # Not given in sources: specific projection matrix construction.
        tsdf_fuser.integrate(depth_map, cam_l)
        
    # Step 5: Extract mesh from TSDF volume
    mesh = tsdf_fuser.extract_mesh()
    
    return mesh
```

**Parameters:**
*   **Stereo-matching model:** A pre-trained model used as a geometric prior. The specific architecture is not defined in the source, but it must be capable of dense depth estimation from stereo pairs [S1].
*   **TSDF Truncation Threshold:** Not given in the sources.
*   **Stereo Pair Selection:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Noisy Gaussian Centers:** Direct reconstruction from Gaussian centers is inaccurate because they are optimized for photometric matching, not geometric consistency [S1]. GS2Mesh avoids this by using rendered images rather than point clouds [S1].
*   **Stereo Ambiguity:** Regions with low texture or repetitive patterns may yield inaccurate depth from the stereo model [S1].
*   **Occlusions:** Stereo matching may fail in occluded regions; TSDF fusion helps mitigate this by integrating multiple views [S1].

## Operating characteristics
*   **Time Complexity:** Reconstruction time is significantly faster than neural surface reconstruction methods. It requires less than five minutes of additional computation time after 3DGS capture for in-the-wild scenes [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Tested on Tanks and Temples (TnT) and DTU benchmarks, achieving state-of-the-art results [S1].
*   **Strengths:** Produces smoother, more accurate surfaces with intricate details compared to methods that add geometric constraints during Gaussian optimization (e.g., SuGaR, 2DGS, GOF) [S1]. It leverages real-world geometric priors from the stereo model [S1].
*   **Limitations:** Requires a pre-trained stereo-matching model. Performance depends on the quality of the stereo model and the rendered images [S1].
*   **When to use:** When a high-quality, smooth mesh is needed from a 3DGS representation, and computational efficiency is important.
*   **When not to use:** When real-time mesh extraction is required during training, or when a pre-trained stereo model is not available.

## Background
GS2Mesh was proposed by Y. Wolf et al. to bridge the gap between the noisy 3DGS representation and the smooth 3D mesh representation [S1]. The method injects real-world knowledge into the depth extraction process by using a pre-trained stereo-matching model, which acts as a geometric regularizer trained on real-world data [S1]. The approach was tested on in-the-wild scenes captured with a smartphone, as well as on the Tanks and Temples and DTU benchmarks [S1]. The method allows for straightforward binding of the mesh to the original 3DGS model for mesh-based manipulation [S1].

## Sources
[S1] GS2Mesh: Surface Reconstruction from Gaussian Splatting via Novel Stereo Views, https://doi.org/10.1007/978-3-031-73024-5_13, fetched 2026-10-29
