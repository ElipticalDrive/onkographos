# Multi-View Stereo Cost Volume Construction

Also called: Cost Volume Construction, Feature Volume Construction

## Overview
Multi-View Stereo (MVS) Cost Volume Construction aggregates features from multiple reference views into a 3D volume to perceive coarse scene structure and provide scene priors for subsequent sampling or rendering [S1]. Unlike simple occupancy voxelization which only records presence, this method encodes photometric consistency and feature similarity across views to estimate depth [S1]. It sets itself apart from [Orthographic Multi-View Voxelization](3892-orthographic-multi-view-voxelization.md) by computing per-voxel feature vectors rather than binary occupancy, and from [Single 2D Projection Voxelization](3482-single-2d-projection-voxelization.md) by leveraging multi-view geometric constraints rather than single-view projection [S1].

## How it works
**Inputs:**
*   $N$ reference views with known camera poses and images.
*   A target view (or query points) defining the region of interest.
*   Depth resolution parameter $L$ (number of depth layers). Typical value: $L=16$ [S1].

**Outputs:**
*   A cost volume $V_c$ containing fused feature vectors for each voxel.

**Data Structures:**
*   **Feature Volume $\{Vf_i\}$:** A 3D tensor for each reference view $i$, storing extracted features.
*   **Cost Volume $V_c$:** A 3D tensor of shape $(H, W, L)$ or $(R_x, R_y, R_z)$ depending on discretization, storing the final fused feature vector per voxel.

**Steps:**

1.  **Feature Extraction:** Extract 2D features from each reference view $i$ using a convolutional network (e.g., Res-UNet) [S1]. Let $f_i^{fe}$ be the feature map for view $i$.
2.  **Depth Warping:** For each voxel in the target view's depth range, warp the features from each reference view $i$ to the target view's coordinate system based on the hypothesized depth layer $l \in \{1, \dots, L\}$ [S1]. This produces warped feature vectors $\tilde{f}_i^{fe}$ for each view and depth layer.
3.  **Global Statistics Calculation:** Compute the mean $\bar{f}^{fe}$ and variance $f^{var}$ of the warped features across all $N$ reference views for each voxel [S1]:
    $$ \bar{f}^{fe} = \frac{1}{N} \sum_{i=1}^{N} \tilde{f}_i^{fe} $$
    $$ f^{var} = \frac{1}{N} \sum_{i=1}^{N} (\tilde{f}_i^{fe} - \bar{f}^{fe})^2 $$
    These statistics represent global information and inter-view differences [S1].
4.  **Weight Inference:** Use a Multi-Layer Perceptron (MLP) $f_e$ to infer mixing weights $\{w_i\}$ for each view. The MLP takes the concatenated input of the warped feature, the mean feature, and the variance feature [S1]:
    $$ \tilde{f}_i^{fe} = MLP_{f_e}(f_i^{fe} \oplus \bar{f}^{fe} \oplus f^{var}) $$
    Here, $\oplus$ denotes concatenation [S1].
5.  **Feature Fusion:** Compute normalized mixing weights $\tilde{w}_i$ (e.g., via Softmax) and fuse the features to obtain the unique feature $f^{fe}$ for each query point/voxel [S1]:
    $$ f^{fe} = \sum_{i=1}^{N} \tilde{w}_i \tilde{f}_i^{fe} $$
    This $f^{fe}$ constitutes the value stored in the cost volume $V_c$ at the corresponding voxel location [S1].

**Pitfalls:**
*   **Checkerboard Artifacts:** Using deconvolution (transposed convolution) layers in the UNet processing the cost volume can cause checkerboard artifacts. This is eliminated by replacing deconvolution with combinations of upsampling and convolution layers [S1].
*   **Precision:** Warping requires accurate depth-to-pixel mapping. Errors in camera poses or depth quantization can lead to misaligned features [S1].

## Subtopics

### Single-Reference Cost-Volume MVS
A lightweight MVS stage using a single reference view per target to recover dense geometry under mobile budgets, addressing sparse tracking clouds [S2, unofficial]. It differs from the standard multi-reference approach by selecting only one reference view to minimize memory and compute, making it suitable for on-device training [S2, unofficial]. Choose this when computational resources are limited or when the input sparse point cloud is too sparse for reliable multi-view initialization [S2, unofficial]. It uses a probabilistic depth range estimation based on 5% and 95% quantiles of sparse BA points, set to $[q_{0.5} \cdot 0.5, q_{0.95} \cdot 1.5]$ [S2, unofficial]. Matching cost uses the Census Transform for illumination robustness, and depth estimation uses plane-sweep with Semi-Global Matching (SGM) aggregation [S2, unofficial]. The reference view is selected to maximize the score:
$$ S_{ref} = \exp\left(-\frac{(b - b_{target})^2}{2\sigma_b^2}\right) \cdot \text{norm}\left(\frac{\alpha}{\alpha_{min}}, 1\right) $$
where $b$ is the candidate baseline, $b_{target}$ is the preferred baseline, and $\alpha$ is the viewing angle [S2, unofficial]. Depth maps are filtered at a confidence threshold of 0.4 [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by feature extraction ($O(N \cdot H \cdot W \cdot C)$) and warping/fusion ($O(H \cdot W \cdot L \cdot N)$). The MLP inference is negligible compared to convolutional operations [S1].
*   **Memory Complexity:** $O(H \cdot W \cdot L \cdot C)$ for the cost volume, where $C$ is the feature dimension. Multi-reference approaches scale linearly with $N$ during warping but only store one fused volume [S1]. Single-reference variants reduce memory by half (one reference instead of many) [S2, unofficial].
*   **Performance:** Trained on Nvidia GeForce RTX 3090 with batch size 500 for 250,000 steps [S1].
*   **Strengths:** Provides rich geometric priors, skips blank areas, and handles illumination variations (with Census Transform) [S1][S2, unofficial].
*   **Limitations:** Requires accurate camera poses and depth ranges. Checkerboard artifacts if deconvolution is used improperly [S1].

## Background
Originated from generalizable Neural Radiance Field (NeRF) methods that use multi-view stereo matching to infer depth [S1]. Specifically, NeRF-SDP introduced this cost volume construction to provide scene priors for sampling [S1]. Later adapted for mobile 3D Gaussian Splatting (PocketGS) to initialize geometry from sparse tracking clouds [S2, unofficial].

## Sources
[S1] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S2] PocketGS: High-Fidelity On-Device Training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2601.17354, fetched 2026-10-26 (unofficial)
