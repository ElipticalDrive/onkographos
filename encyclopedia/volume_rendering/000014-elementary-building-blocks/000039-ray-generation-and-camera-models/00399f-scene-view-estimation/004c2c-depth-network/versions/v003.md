# Depth Network

Also called: Depth Network d

## Overview
The Depth Network is a component in sparse-view novel view synthesis systems that refines initial depth estimates to construct accurate 3D point clouds for rendering [S1]. It solves the problem of inaccurate depth estimation in sparse input scenarios, where traditional Multi-View Stereo (MVS) fails due to limited overlap and large baselines, and single-image estimation suffers from scaling/shifting ambiguity [S1]. It distinguishes itself from sibling methods like [Viewpoint Estimation via Sphere Classification](15364-viewpoint-estimation-via-sphere-classification.md) and [Batch Active View Selection](15787-batch-active-view-selection.md) by explicitly regressing geometric depth maps from image features rather than estimating camera poses or selecting views [S1].

## How it works
The Depth Network takes an input image and an initial depth map (derived from MVS or sensor data) and outputs a refined depth map [S1]. It employs a hybrid design that cascades a lightweight MVS module with a U-Net refinement network [S1].

**Inputs and Outputs:**
*   **Input Image ($I_i$):** The $i$-th input view.
*   **Initial Depth ($D_{init}$):** Either MVS-estimated depths or sensor depths ($D^{sen}$) [S1].
*   **Output ($D_i$):** Refined depth map for building the point cloud [S1].

**Data Structures:**
*   **Binary Mask ($M_i$):** A mask indicating valid sensor depth pixels. If sensor depth is not used, this is not applicable [S1].
*   **U-Net:** The core refinement architecture. It takes the image $I_i$ and initial depth as inputs [S1].

**Steps:**
1.  **Initial Estimation:** Use PatchmatchNet [77] to generate initial depth estimations from the sparse input views [S1]. If sensor depth $D^{sen}$ is available, it is used directly as the initial input instead of MVS output [S1].
2.  **Feature Encoding:** The image $I_i$ is processed by a spatial feature encoder (built on BigGAN architecture [3]) to generate per-pixel feature vectors $F_i$. Each vector is 61-dimensional and concatenated with RGB channels, resulting in 64 dimensions total [S1].
3.  **Refinement:** The U-Net takes the image $I_i$ and the initial depth map ($D_{init}$) as inputs. It refines the depth using multi-view stereo cues and image cues [S1].
4.  **Output:** The U-Net outputs the refined depth $D_i$, which is used to build the point cloud [S1].

**Formulas:**
During training, if sensor depth is available, a loss function $L_s$ is used to encourage the U-Net output to match the sensor depth:
$$ L_s = \| M_i D_i - M_i D^{sen} \| $$
where $M_i$ is the binary mask of valid sensor depths, $D_i$ is the network output, and $D^{sen}$ is the sensor depth [S1].

**Parameters:**
*   **MVS Module:** PatchmatchNet [77] is used for its speed and lightweight nature [S1].
*   **Feature Encoder:** Based on BigGAN [3], producing 64-dimensional features per pixel [S1].
*   **Loss Weight:** Not given in the sources.

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

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Handles sparse input views where traditional MVS fails; refines noisy sensor data; integrates multi-view cues with single-image appearance cues [S1].
*   **Limitations:** Relies on the quality of the initial MVS estimate or sensor data; single-image estimation alone is ambiguous [S1].
*   **When to use:** When synthesizing novel views from a sparse set of images and accurate geometry is required for point cloud construction [S1].
*   **When not to use:** When dense input views are available and standard MVS is sufficient, or when camera pose estimation is the primary goal rather than depth refinement [S1].

## Background
The Depth Network was introduced in the context of Forward Warping and Depth (FWD) for real-time novel view synthesis [S1]. It addresses the limitations of learning 3D geometry end-to-end by relying on rendering and refinement [S1]. The method uses PatchmatchNet for initial MVS and a U-Net for refinement, leveraging the BigGAN architecture for feature extraction [S1]. The Direct Prediction Module variant was introduced in the context of learning indoor inverse rendering with 3D spatially-varying lighting [S2].

## Sources
[S1] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
[S2] Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting, https://doi.org/10.1109/iccv48922.2021.01231, fetched 2026-10-26
