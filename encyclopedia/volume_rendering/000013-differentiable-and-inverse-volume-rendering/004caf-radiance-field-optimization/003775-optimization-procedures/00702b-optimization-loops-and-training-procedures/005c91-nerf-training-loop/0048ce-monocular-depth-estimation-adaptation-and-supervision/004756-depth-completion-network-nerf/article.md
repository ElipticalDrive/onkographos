# Depth Completion Network NeRF

Also called: DCN-NeRF, Depth Completion (MFFNet), Guided depth completion, MFFNet

## Overview
Depth Completion Network NeRF integrates a guided depth completion network into the NeRF training loop to recover dense depth maps from sparse LiDAR measurements using image guidance for higher accuracy [S2]. Unlike sibling methods that rely on monocular depth estimation (MDE) priors which lack absolute scale or suffer from accuracy compromises, this method uses LiDAR-based completion to provide dense, metrically consistent depth supervision [S2]. It sets itself apart by explicitly addressing the sparsity of raw LiDAR data through guided completion, making it suitable for unbounded real-world scenes where raw LiDAR is insufficient [S2]. Builds on [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** Input RGB images $I$, sparse raw LiDAR depth measurements $D_l$, and camera poses [S2].
*   **Outputs:** A dense depth map $D_{guide}$ used for supervision, and the final NeRF radiance field parameters [S2].

**Data Structures:**
*   **Sparse LiDAR Depth ($D_l$):** A sparse 2D map or point cloud representing raw LiDAR returns [S2].
*   **RGB Image ($I$):** The corresponding color image for the LiDAR frame [S2].
*   **Completion Network ($\delta$):** A neural network (specifically MFFNet) that takes $I$ and $D_l$ as input [S2].

**Steps:**
1.  **Sparse Point Cloud Extraction:** Extract a sparse point cloud from the scene using COLMAP to establish initial geometry and camera poses [S1].
2.  **Depth Completion:** Apply the guided depth completion network $\delta$ to the input image $I$ and raw LiDAR depth $D_l$ to produce a dense, guided depth map $D_{guide}$ [S2]:
    $$ D_{guide} = \delta(I, D_l) $$
    where $\delta$ represents the completion network (e.g., MFFNet) [S2].
3.  **NeRF Rendering:** Render the NeRF model $F_\Theta$ to obtain the predicted depth $\bar{D}$ and color for each ray, using standard volume rendering accumulation [S1].
4.  **Loss Calculation:** Compute the total loss. This includes the standard volumetric loss (color reconstruction) and a depth loss based on the predicted depth and its uncertainty, minimizing the divergence between the ray's distribution and the noisy depth distribution [S1].
5.  **Optimization:** Update NeRF parameters to minimize the combined loss [S1].

**Parameters:**
*   **Completion Method:** MFFNet is selected as the representative guided depth completion method [S2].
*   **Training Data:** RGB-D data from ScanNet and Matterport3D datasets [S1].
*   **Noise:** Gaussian noise is added to the depth during training to improve robustness [S1].

**Edge Cases and Pitfalls:**
*   **Sparsity:** Raw LiDAR is inherently sparse. The completion network is necessary to bridge this gap [S2].
*   **Uncertainty:** If the completion network outputs uncertainty, it should be used to weight the depth loss, penalizing confident predictions more heavily [S1].

## Subtopics

### LiDAR Depth Completion with NLSPN
NLSPN propagates depth information from LiDAR points to surrounding pixels to provide credible depth supervisions from defect LiDAR depths [S3, unofficial]. It requires accumulating neighbor LiDAR frames to get much denser depths for input [S3, unofficial]. However, NLSPN performs well with 64-channel LiDAR data (e.g., KITTI) but fails with nuScenes' 32-channel LiDAR data because it is too sparse for the depth completion network [S3, unofficial]. Additionally, accumulated LiDAR data contain outliers due to moving objects, ill poses, and occlusions, leading to wrong depth supervisions [S3, unofficial].

## Operating characteristics
*   **Complexity:** Adds the inference cost of the depth completion network (MFFNet) to the NeRF training loop.
*   **Performance:** Achieves higher accuracy than unguided methods by leveraging image guidance [S2].
*   **Strengths:** Effective for unbounded real scenes where raw LiDAR is sparse [S2].
*   **Limitations:** Relies on the quality of the LiDAR hardware and the completion network; sensitive to outliers in accumulated LiDAR data if not handled [S3, unofficial].
*   **When to use:** When sparse LiDAR data is available and high-accuracy dense depth is required for NeRF training in outdoor or large-scale scenes [S2].
*   **When not to use:** When only monocular images are available (use MDE-based methods instead) or when LiDAR is too sparse for the completion network (e.g., 32-channel LiDAR without accumulation) [S3, unofficial].

## Background
The method originates from the need to handle unbounded real scenes where the original NeRF parametrization was limited to bounded or forward-facing scenes [S2]. It builds on the observation that raw LiDAR provides sparse measurements, while monocular methods compromise on accuracy [S2]. The approach was developed to integrate guided depth completion (specifically MFFNet) into NeRF training, using datasets like ScanNet and Matterport3D [S1, S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
[S3] S-NeRF: Neural Radiance Fields for Street Views, https://doi.org/10.48550/arxiv.2303.00749, fetched 2026-10-29 (unofficial)
