# VolSplat

Also called: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, Voxel-aligned Feed-forward 3DGS

## Overview
VolSplat is a multi-view feed-forward framework for 3D Gaussian Splatting (3DGS) that replaces the standard pixel-aligned Gaussian prediction with a voxel-aligned prediction strategy [S1]. It solves the problems of view-biased density distributions and alignment errors (such as floaters) inherent in pixel-aligned methods by aggregating multi-view features into a shared 3D voxel grid before regressing Gaussian parameters [S1]. Unlike its parent method, which uses a transformer to decode tokens directly into Gaussian parameters, VolSplat explicitly constructs a 3D voxel feature grid via depth unprojection and refines it with a sparse 3D decoder to enable adaptive density control based on volumetric scene complexity [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ input images $I = \{I_i\}_{i=1}^N$ where $I_i \in \mathbb{R}^{H \times W \times 3}$ and their corresponding camera poses $P = \{P_i\}_{i=1}^N$ [S1].
*   **Output:** A set of 3D Gaussians $\mathcal{G} = \{(\mu_j, \Sigma_j, \alpha_j, c_j)\}_{j=1}^M$, where $M$ is the number of occupied voxels, and novel views rendered from these Gaussians [S1].

**Data Structures:**
*   **Voxel Feature Grid:** A sparse 3D grid storing features for occupied voxels. Each voxel holds a feature vector $f \in \mathbb{R}^C$.
*   **Gaussian Parameters:** For each occupied voxel, the model predicts:
    *   Position $\mu \in \mathbb{R}^3$
    *   Covariance $\Sigma \in \mathbb{R}^{3 \times 3}$ (or its 6 independent components)
    *   Opacity $\alpha \in [0, 1]$
    *   Color $c \in \mathbb{R}^3$ [S1]

**Steps:**

1.  **2D Feature Extraction:** Extract 2D features for each input image using a Transformer-based network (pretrained image encoder $\Phi_{image}$) [S1][S2, unofficial].
    $$ F_i^{raw} = \Phi_{image}(I_i) $$
2.  **Cost Volume Construction:** Construct per-view cost volumes using plane sweeping [S1][S2, unofficial]. This involves matching features across views at different depth hypotheses.
3.  **Depth Prediction:** Estimate a depth map $D_i$ for each view using a Depth Prediction Module [S1][S2, unofficial].
4.  **Unprojection to Voxel Grid:** Unproject the 2D features into 3D space using the estimated depth maps $D_i$ and camera poses $P_i$ to form an initial voxel feature grid [S1][S2, unofficial]. Features are aggregated into the shared 3D voxel container, decoupling feature fusion from camera view frustums [S1].
5.  **3D Feature Refinement:** Refine the 3D voxel features using a sparse 3D decoder (implemented as a 3D U-Net) [S1][S2, unofficial].
    *   The sparse 3D decoder takes the initial voxel features as input.
    *   It predicts residual features $\Delta f$ for each occupied voxel [S1].
    *   The refined voxel features $f_{refined}$ are obtained by adding the residuals to the original features:
        $$ f_{refined} = f_{initial} + \Delta f $$
        (See Figure 3 in [S1])
6.  **Gaussian Parameter Prediction:** Predict the parameters of a 3D Gaussian for each occupied voxel based on the refined voxel features $f_{refined}$ [S1][S2, unofficial]. A prediction head $\Psi_{pred}$ maps $f_{refined}$ to $(\mu, \Sigma, \alpha, c)$ [S1].
7.  **Rendering:** Render novel views from the predicted 3D Gaussians using standard differentiable Gaussian splatting [S1][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volsplat_forward(images, camera_poses):
    """
    images: List of N images, shape (H, W, 3)
    camera_poses: List of N camera pose matrices
    Returns: List of Gaussian parameters for occupied voxels
    """
    N = len(images)
    
    # Step 1: Extract 2D features
    # Phi_image is a pretrained Transformer-based encoder
    raw_features = [Phi_image(img) for img in images]
    
    # Step 2: Construct per-view cost volumes with plane sweeping
    # This step aligns features across views at discrete depth levels
    cost_volumes = [construct_cost_volume(raw_features[i], camera_poses[i], raw_features, camera_poses) 
                    for i in range(N)]
    
    # Step 3: Estimate depth maps
    depth_maps = [DepthPredictionModule(cost_volumes[i]) for i in range(N)]
    
    # Step 4: Unproject features to 3D voxel grid
    # Unproject 2D features using depth maps and camera poses
    # Aggregates features into a shared 3D voxel space
    voxel_features_initial = unproject_to_voxel_grid(raw_features, depth_maps, camera_poses)
    
    # Step 5: Refine voxel features using Sparse 3D Decoder (3D U-Net)
    # The decoder predicts residual features
    residual_features = Sparse3DDecoder(voxel_features_initial)
    
    # Add residuals to initial features
    voxel_features_refined = voxel_features_initial + residual_features
    
    # Step 6: Predict Gaussian parameters for each occupied voxel
    # Psi_pred is the prediction head
    gaussians = [Psi_pred(voxel_features_refined[voxel_id]) for voxel_id in occupied_voxels]
    
    return gaussians
```

**Parameters:**
*   $N$: Number of input views.
*   $H, W$: Image height and width.
*   $C$: Feature dimension.
*   $\Phi_{image}$: Pretrained image encoder (e.g., Transformer-based) [S1].
*   $h$: Function for cross-view feature matching and fusion (used in pixel-aligned baselines, but VolSplat replaces this with volumetric aggregation) [S1].
*   $U$: Feature upsampler (used in pixel-aligned baselines to upsample features to full resolution $F_{full} = U(F)$) [S1]. VolSplat does not use this upsampling step for the final prediction [S1].

**Edge Cases and Pitfalls:**
*   **Occlusions and Low Texture:** Pixel-aligned methods suffer from alignment errors in these regions, leading to floaters. VolSplat mitigates this by fusing information from multiple views into a shared 3D container, ensuring robust multi-view consistency [S1].
*   **Scene Complexity:** Pixel-aligned methods enforce a uniform density distribution ($|S| = H \times W \times N$), causing redundant primitives in simple regions and failure to capture fine-grained structures. VolSplat enables adaptive density control based on 3D scene complexity, as the distribution of Gaussians is determined by the volumetric structure itself [S1].
*   **Discretization Error:** Sampling at discrete pixel locations is sensitive to camera calibration and discretization error. Voxel alignment reduces this by operating in a unified 3D space [S1].

## Subtopics

### [VS-Splat](8542-vs-splat.md)
VS-Splat is a feed-forward framework for 3D object reconstruction from sparse-view 2D images that employs a coarse-to-fine (C2F) strategy to predict 3D Gaussian primitives. It distinguishes itself from sibling methods like LaRa and GenDen by performing voxel-level selection *before* generating fine primitives, rather than selecting existing primitives for refinement. This ensures that the coarse representation determines the spatial location for new fine primitives, enabling adaptive density control without auxiliary supervision.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Evaluated on RealEstate10K, ScanNet, and ACID benchmarks, achieving state-of-the-art performance [S1].
*   **Strengths:**
    *   Robust multi-view consistency due to volumetric aggregation [S1].
    *   Cleaner occlusion handling [S1].
    *   Adaptive density control based on scene complexity [S1].
    *   Eliminates the need for per-query 2D prediction patterns [S1].
*   **Limitations:**
    *   Requires accurate depth estimation for unprojection [S1].
    *   Relies on a sparse 3D decoder (3D U-Net), which may have higher computational overhead compared to simple pixel-wise prediction heads [S1].
*   **When to use:** When multi-view alignment is challenging due to occlusions or low texture, and adaptive density control is required for complex scenes [S1].
*   **When not to use:** When computational resources are extremely limited and a simpler pixel-aligned method suffices, or when the scene is very simple and uniform density is acceptable [S1].

## Background
VolSplat was introduced in the paper "VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction" [S1]. It addresses the limitations of existing feed-forward 3DGS methods that rely on pixel-aligned Gaussian prediction paradigms [S1]. The method was released with code and checkpoints in September 2025 [S2, unofficial]. It is evaluated on large-scale benchmarks such as RealEstate10K, ScanNet, and ACID [S1].

## Sources
[S1] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
[S2] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
