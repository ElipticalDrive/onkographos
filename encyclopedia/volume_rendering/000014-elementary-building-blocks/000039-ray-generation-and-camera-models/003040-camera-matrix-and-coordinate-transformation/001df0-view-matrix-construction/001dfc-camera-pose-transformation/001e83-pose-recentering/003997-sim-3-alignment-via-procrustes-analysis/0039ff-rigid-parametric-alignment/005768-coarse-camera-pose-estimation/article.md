# Coarse Camera Pose Estimation

Also called: Coarse Pose Estimation, Initial Pose Estimation via Scene Regressor

## Overview
Coarse Camera Pose Estimation initializes the extrinsic parameters for a target image by leveraging a pre-trained scene regressor to generate 3D geometry, followed by a Perspective-n-Point (PnP) solver. It solves the problem of registering unposed images to a global coordinate frame when no prior pose information is available. Unlike its siblings, which rely on geometric feature matching (SfM), iterative point cloud alignment (ICP), or deep learning-based image matching, this method uses a neural scene representation to directly predict 3D points from image pairs, enabling registration even in textureless or repetitive regions where traditional feature matching fails [S1, unofficial].

## How it works
**Inputs:**
- Registered reference image $I_{ref}$ (or $I_i$).
- Target image $I_k$ to be registered.
- Camera intrinsic matrix $K$.
- Inlier threshold $s_{inlier}$ (typically 5,000) [S1, unofficial].
- Reprojection error threshold (typically 6 pixels) [S1, unofficial].

**Outputs:**
- Coarse camera pose $T_{coarse}$ (extrinsics) for $I_k$.
- Boolean flag indicating if $I_k$ is accepted into the training buffer.

**Data Structures:**
- Scene Regressor $f_{SCR}$: A neural network (e.g., Transformer-based) that takes two images and outputs a set of 3D points $\{X_k\}$ in the global coordinate frame.
- Pixel Coordinates $\{u_k\}$: 2D coordinates for each pixel in $I_k$.
- 3D Points $\{X_k\}$: Predicted 3D coordinates corresponding to $\{u_k\}$.
- Inlier Set: Subset of correspondences satisfying the reprojection error threshold.

**Steps:**
1. **Scene Regression:** Pass $I_{ref}$ and $I_k$ to the scene regressor $f_{SCR}$ to obtain the set of 3D points $\{X_k\}$ in the global coordinate frame [S1, unofficial].
   $$ \{X_k\} = f_{SCR}(I_{ref}, I_k) $$
2. **Correspondence Generation:** Construct 2D-3D correspondences $\{(u_k, X_k)\}$ using the pixel coordinates $\{u_k\}$ from $I_k$ and the predicted 3D points $\{X_k\}$ [S1, unofficial].
3. **PnP Solver:** Use a PnP solver (specifically DSAC, modified for non-square pixels) to estimate the camera pose $T_{coarse}$ and compute the number of inliers $S_k$ [S1, unofficial].
   $$ T_{coarse}, S_k = \text{PnP}(K, \{(u_k, X_k)\}) $$
   The solver minimizes the reprojection error. The specific loss function used during the underlying optimization (if iterative) involves the ray direction $\nu_{i,k}$ and scaling factor $d_{i,k}$:
   $$ \rho(\|d_{i,k} \cdot \nu_{i,k} - (X_i - C_k)\|_2) $$
   where $C_k$ is the camera center [S1, unofficial].
4. **Validation:** Compare the number of inliers $S_k$ against the threshold $s_{inlier}$ [S1, unofficial].
5. **Buffer Update:** If $S_k > s_{inlier}$, add $I_k$ to the training buffer and register it. Otherwise, discard it [S1, unofficial].

**Parameters:**
- $s_{inlier}$: Inlier threshold. Default: 5,000 [S1, unofficial].
- Reprojection Error Threshold: Maximum allowed pixel distance for an inlier. Default: 6 pixels [S1, unofficial].
- Focal Lengths: DSAC is modified to support different focal lengths for x and y axes [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Gauge Ambiguity:** The camera pose of the seed image is fixed during optimization to resolve gauge ambiguity [S1, unofficial].
- **Scale Ambiguity:** The scaling factor between the seed image and its most similar adjacent image is fixed to resolve scale ambiguity [S1, unofficial].
- **Pose Inaccuracy:** Poses of newly registered images may be inaccurate if the scene regressor has not seen similar images. A MLP pose refiner was tested but found ineffective for transformer-based regressors [S1, unofficial].
- **Reference Selection:** The reference image is selected from registered images that connect to the most unregistered images to ensure robust initialization [S1, unofficial].

## Subtopics

### [Structure-from-Motion (SfM)](16341-structure-from-motion-sfm.md)
Structure-from-Motion (SfM) is a traditional image-based reconstruction pipeline that estimates camera parameters (extrinsics and intrinsics) to build sparse and dense point clouds, followed by mesh generation and texture extraction.

## Operating characteristics
- **Time Complexity:** Dominated by the inference time of the scene regressor $f_{SCR}$ and the PnP solver. The PnP solver (DSAC) is typically fast, but the neural network inference scales with image resolution and network depth.
- **Memory Complexity:** Requires storing the 3D points $\{X_k\}$ for the target image and the scene regressor weights.
- **Strengths:** Does not rely on sparse features, making it robust in textureless or repetitive scenes where SfM fails. It provides a dense set of correspondences, potentially leading to more robust PnP solutions.
- **Limitations:** Accuracy depends heavily on the quality of the scene regressor. If the regressor has not seen similar images, the predicted 3D points may be erroneous, leading to low inlier counts. It requires a pre-trained scene regressor, adding an initialization overhead.
- **When to use:** When traditional feature-based methods (SfM) fail due to lack of distinctive features, or when a dense geometric prior is beneficial.
- **When not to use:** When computational resources are extremely limited (neural inference is slower than feature extraction) or when high-precision initial poses are available from other sources.

## Background
Coarse Camera Pose Estimation via scene regression is a technique used in recent neural rendering pipelines, such as ZeroGS, to initialize camera poses from unposed images. It builds on the idea that a neural network can learn to predict 3D structure from image pairs, bypassing the need for explicit feature matching. The method was introduced in the context of training 3D Gaussian Splatting from unposed images, addressing the challenge of incremental registration [S1, unofficial].

## Sources
[S1] ZeroGS: Training 3D Gaussian Splatting from Unposed Images, https://doi.org/10.48550/arxiv.2411.15779, fetched 2026-10-26 (unofficial)
