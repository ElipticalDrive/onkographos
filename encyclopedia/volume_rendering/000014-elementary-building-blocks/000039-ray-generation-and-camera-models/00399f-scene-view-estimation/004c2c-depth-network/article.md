# Depth Network

Also called: Depth Network d

## Overview
The Depth Network is a component in sparse-view novel view synthesis systems that refines initial depth estimates to construct accurate 3D point clouds for rendering [S1]. It solves the problem of inaccurate depth estimation in sparse input scenarios, where traditional Multi-View Stereo (MVS) fails due to limited overlap and large baselines, and single-image estimation suffers from scaling/shifting ambiguity [S1]. It distinguishes itself from sibling methods like [Viewpoint Estimation via Sphere Classification](15364-viewpoint-estimation-via-sphere-classification.md) and [Batch Active View Selection](15787-batch-active-view-selection.md) by explicitly regressing geometric depth maps from image features rather than estimating camera poses or selecting views [S1].

## How it works
The Depth Network takes an input image and an initial depth map (derived from MVS or sensor data) and outputs a refined depth map [S1]. It employs a hybrid design that cascades a lightweight MVS module with a U-Net refinement network [S1].

**Inputs and Outputs:**
*   **Input Image ($I_i$):** The $i$-th input view [S1].
*   **Initial Depth ($D_{init}$):** Either MVS-estimated depths or sensor depths ($D^{sen}$) [S1].
*   **Output ($D_i$):** Refined depth map for building the point cloud [S1].

**Data Structures:**
*   **Binary Mask ($M_i$):** A mask indicating valid sensor depth pixels [S1].
*   **U-Net:** The core refinement architecture that takes the image $I_i$ and initial depth as inputs [S1].

**Steps:**
1.  **Initial Estimation:** Use PatchmatchNet [77] to generate initial depth estimations from the sparse input views [S1]. If sensor depth $D^{sen}$ is available, it is used directly as the initial input instead of MVS output [S1].
2.  **Feature Encoding:** The image $I_i$ is processed by a spatial feature encoder (built on BigGAN architecture [3]) to generate per-pixel feature vectors $F_i$ [S1]. Each vector is 61-dimensional and concatenated with RGB channels, resulting in 64 dimensions total [S1].
3.  **Refinement:** The U-Net takes the image $I_i$ and the initial depth map ($D_{init}$) as inputs [S1]. It refines the depth using multi-view stereo cues and image cues [S1].
4.  **Output:** The U-Net outputs the refined depth $D_i$, which is used to build the point cloud [S1].

**Formulas:**
During training, if sensor depth is available, a loss function $L_s$ is used to encourage the U-Net output to match the sensor depth:
$$ L_s = \| M_i D_i - M_i D^{sen} \| $$
where $M_i$ is the binary mask of valid sensor depths, $D_i$ is the network output, and $D^{sen}$ is the sensor depth [S1].

**Parameters:**
*   **MVS Module:** PatchmatchNet [77] is used for its speed and lightweight nature [S1].
*   **Feature Encoder:** Based on BigGAN [3], producing 64-dimensional features per pixel [S1].
*   **Loss Weight:** Not given in the sources. [S1]

**Edge Cases and Pitfalls:**
*   **Sparse Inputs:** Applying MVS solely on sparse inputs is challenging due to limited overlap and huge baselines, leading to inaccurate and low-confidence estimations [S1].
*   **Ambiguity:** Single-image depth estimation suffers from scaling/shifting ambiguity, losing valuable multi-view cues and causing inconsistent estimations across views [S1].
*   **Sensor Noise:** Sensor depths are often noisy and incomplete; the U-Net acts as a completion and refinement module in this setting [S1].

## Subtopics

### Direct Prediction Module
A multi-branch ResNet backbone that predicts intrinsic properties (albedo, normals, depth) and a global lighting feature from a single input image [S2]. It distinguishes itself by predicting a global lighting feature vector alongside geometric properties, enabling downstream volumetric lighting prediction [S2]. Choose this when inverse rendering of indoor scenes with spatially-varying lighting is required [S2].

**Definition:** Predicts albedo $\tilde{A}$, surface normals $\tilde{N}$, depth $\tilde{D}$, and a global lighting feature $\tilde{f}_L$ from a single linearized RGB image $I$ [S2].

**Formula:**
$$ \tilde{A}, \tilde{N}, \tilde{D}, \tilde{f}_L = h_{DP} (I; \Theta_{DP}) $$
where $h_{DP}$ is the ResNet backbone and $\Theta_{DP}$ are the network parameters [S2].

**Parameters:**
*   **Input:** Single linearized RGB image $I \in \mathbb{R}^{3 \times H \times W}$ [S2].
*   **Output:** Albedo $\tilde{A} \in \mathbb{R}^{3 \times H \times W}$, Normals $\tilde{N} \in \mathbb{R}^{3 \times H \times W}$, Depth $\tilde{D} \in \mathbb{R}^{H \times W}$, Global Feature $\tilde{f}_L \in \mathbb{R}^C$ [S2].

### PackNet Depth Estimation
Uses an off-the-shelf monocular depth estimator (PackNet) to obtain 2.5D geometry of the scene for lighting volume prediction and object insertion [S3].

**Steps:**
1.  **Depth Usage for Lighting:** Use predicted depth to unproject scene pixel values into 3D for lighting volume prediction [S3].
2.  **Depth Usage for Object Insertion:** Use depth to decide the 3D placement of the inserted object [S3].
3.  **Depth Rescaling:** Use projected sparse LiDAR points as ground-truth depth to rescale the estimated depth by minimizing the L2 error between them, addressing scale error [S3].
4.  **Shadow Rendering:** Use pre-computed depth to compute the 3D location of each scene pixel for ray-mesh queries [S3].

**Pitfalls:**
*   Scale error in depth prediction leads to error in the size of inserted objects [S3].
*   Errors in estimated depth may result in incorrect shadows, though empirically negligible when inserting on flat surfaces [S3].

### Batched Depth Estimation
Estimates depth for a list of RGB images by batching model forward passes for speed on MPS/CUDA, falling back to per-frame estimation on failure [S4, unofficial].

**Steps:**
1.  Initialize output list `out` and load model if torch stack is available [S4, unofficial].
2.  While `len(out) < n`, take a chunk of `rgbs` of size `_BATCH` (or remaining frames) [S4, unofficial].
3.  Pass the chunk to the pipeline; if successful, decode each result using `_model_output_to_depth` and extend `out`; if failed, break the loop [S4, unofficial].
4.  Call `progress` callback with current count and total if provided [S4, unofficial].
5.  After the batch loop, if `len(out) < n`, estimate depth for remaining frames individually using `estimate_depth` [S4, unofficial].

**Parameters:**
*   **Batch size `_BATCH`:** Determined by environment variable `FOURD_DEPTH_BATCH`, default 4 [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1]
*   **Memory Complexity:** Not given in the sources. [S1]
*   **Measured Performance:** Not given in the sources. [S1]
*   **Strengths:** Handles sparse input views where traditional MVS fails; refines noisy sensor data; integrates multi-view cues with single-image appearance cues [S1].
*   **Limitations:** Relies on the quality of the initial MVS estimate or sensor data; single-image estimation alone is ambiguous [S1].
*   **When to use:** When synthesizing novel views from a sparse set of images and accurate geometry is required for point cloud construction [S1].
*   **When not to use:** When dense input views are available and standard MVS is sufficient, or when camera pose estimation is the primary goal rather than depth refinement [S1].

## Background
The Depth Network was introduced in the context of Forward Warping and Depth (FWD) for real-time novel view synthesis [S1]. It addresses the limitations of learning 3D geometry end-to-end by relying on rendering and refinement [S1]. The method uses PatchmatchNet for initial MVS and a U-Net for refinement, leveraging the BigGAN architecture for feature extraction [S1]. The Direct Prediction Module variant was introduced in the context of learning indoor inverse rendering with 3D spatially-varying lighting [S2].

## Sources
[S1] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
[S2] Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting, https://doi.org/10.1109/iccv48922.2021.01231, fetched 2026-10-26
[S3] Neural Light Field Estimation for Street Scenes with Differentiable Virtual Object Insertion, https://doi.org/10.1007/978-3-031-20086-1_22, fetched 2026-10-30
[S4] devarshh/volumetric-video-4dgs, https://github.com/devarshh/volumetric-video-4dgs, fetched 2026-10-26 (unofficial)
[S5] Neural-radiance-fields-based holography [Invited], https://doi.org/10.1364/ao.523562, fetched 2026-10-26
