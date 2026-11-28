# MVSplat

Also called: Amortized 3D Reconstruction, Feed-forward 3DGS, Feedforward 3D Reconstruction, Single-pass 3DGS

## Overview
MVSplat is a feed-forward deep neural network that directly regresses 3D Gaussian Splatting (3DGS) parameters from sparse multi-view images in a single pass, bypassing the per-scene gradient descent optimization loop required by standard 3DGS [S4]. It builds upon PixelSplat by combining multi-view feature extraction through transformers and cost volumes, which are then processed by a U-Net [S1]. Unlike iterative methods that optimize primitives for a specific scene, MVSplat provides a universal reconstruction model trained on all CO3Dv2 categories, offering useful 3D priors for subsequent 3D optimizations [S5].

## How it works
**Inputs and Outputs:**
*   **Input:** $K$ sparse-view images $I^i \in \mathbb{R}^{H \times W \times 3}$ and their corresponding camera projection matrices $P^i = K_i [R_i | t_i]$ [S4].
*   **Output:** A set of $H \times W \times K$ 3D Gaussian primitives, each defined by position $\mu_j$, opacity $\alpha_j$, covariance $\Sigma_j$, and color $c_j$ (represented as spherical harmonics) [S4].

**Data Structures:**
The core intermediate structure is a per-view cost volume that stores cross-view feature similarities for all potential depth candidates, providing geometry cues for depth estimation [S4].

**Algorithm Steps:**
1.  Extract multi-view image features using a Transformer backbone [S4].
2.  Construct per-view cost volumes using plane sweeping to store cross-view feature similarities for potential depth candidates [S4].
3.  Concatenate Transformer features and cost volumes as input to a 2D U-Net with cross-view attention [S4].
4.  Refine the cost volume and predict per-view depth maps using the 2D U-Net [S4].
5.  Unproject the per-view depth maps to 3D space using camera intrinsics and poses [S4].
6.  Combine unprojected depths using a deterministic union operation to form the 3D Gaussian centers [S4].
7.  Predict opacity, covariance, and color Gaussian parameters jointly with the depth maps in parallel [S4].
8.  Render novel views from the predicted 3D Gaussians using the differentiable splatting rasterization operation [S4].

**Formulas:**
Cost volume construction: $C_i(u, d_k) = \text{Agg}_{m,i} \langle F_i(u), W_{m\to i}(F_m; d_k) \rangle$, where $C_i$ is the cost volume, $u$ is the pixel coordinate, $d_k$ is a depth hypothesis, $F_i$ and $F_m$ are features from view $i$ and reference view $m$, and $W$ is a warping function [S3].
Depth prediction: $\hat{d}_i = \sum_k \text{softmax}(C_i)_k d_k$, where $\hat{d}_i$ is the predicted depth and $d_k$ are depth hypotheses [S3].
Gaussian center unprojection: $\mu_i(u) = \pi^{-1}(u, \hat{d}_i(u); K_i, T_i)$, where $\mu_i$ is the Gaussian mean, $\pi^{-1}$ is the unprojection function, $K_i$ is the camera intrinsic matrix, and $T_i$ is the camera pose [S3].

**Parameters:**
*   **Training Setup:** 20 source images are used as inputs during training, with MSE losses calculated on 5 novel views [S5].
*   **Finetuning Setup:** 80 views are used as inputs to the feed-forward model for initialization in the finetuning regime [S5].
*   **Loss Function:** The model is trained end-to-end purely with photometric loss between rendered and ground truth images [S4].

**Edge Cases and Pitfalls:**
*   **Single-View Ambiguity:** Single-view reconstruction is ill-posed and ambiguous for general scenes; MVSplat avoids this by requiring multi-view inputs [S4].
*   **Feature-to-Depth Mapping:** Predicting probabilistic depth distributions solely from image features is ambiguous and unreliable, leading to noisy geometry artifacts; MVSplat mitigates this by using a cost volume for feature matching [S4].

## Subtopics

### [Sparse Input Reconstruction via Generalizability](11788-sparse-input-reconstruction-via-generalizability.md)
Addresses reconstructing 3D scenes from a limited number of input views (sparse views) or even a single image, bypassing the iterative per-scene optimization required by standard 3D Gaussian Splatting. It achieves this by employing learned priors within a feed-forward deep neural network that directly regresses 3D Gaussian parameters from input images in a single pass. This approach favors inference speed and cross-scene generalization over the high geometric fidelity typically achieved by regularization-based methods that rely on per-scene gradient descent.

### [VolSplat Architecture and Training Pipeline](13066-volsplat-architecture-and-training-pipeline.md)
VolSplat is a feed-forward deep neural network that directly regresses 3D Gaussian Splatting (3DGS) parameters from sparse multi-view images in a single pass, bypassing the per-scene gradient descent optimization required by standard 3DGS [Parent](9213-mvsplat.md). It distinguishes itself by lifting 2D features into a sparse 3D voxel grid via plane-sweep depth prediction and predicting one Gaussian primitive per occupied voxel, ensuring scene-adaptive primitive allocation while maintaining stable sparse 3D decoding.

## Operating characteristics
*   **Time Complexity:** The method performs inference in a single forward pass, achieving 22 fps on RealEstate10K and ACID benchmarks [S4].
*   **Memory Complexity:** Memory usage is determined by the neural network's parameter count and intermediate feature maps, requiring 1.18 GiB GPU memory on RealEstate10K [S3].
*   **Measured Performance:** Model-stage latency is 37.63 ms with a throughput of 26.57 scenes/s, yielding 871.21 rFPS for decoder-only rendering [S3]. The model uses 10× fewer parameters and infers more than 2× faster than pixelSplat, while providing higher appearance and geometry quality [S4].
*   **Strengths:** Bypasses per-scene gradient descent; provides useful 3D priors for subsequent 3D optimizations; works on a variety of object types [S5].
*   **Limitations:** As a feed-forward model, it may not achieve the same level of geometric fidelity as iterative methods that overfit to a specific scene's data, particularly in areas with limited viewpoint coverage [S4].

## Background
MVSplat was introduced to combine multi-view feature extraction through transformers and cost volumes, processed by a U-Net, building upon the foundations of PixelSplat [S1]. It is part of a broader trend towards amortized 3D reconstruction, where a neural network is trained to generalize across scenes rather than optimizing for a single instance [S4]. The model was trained on all CO3Dv2 categories to create a universal reconstruction model capable of handling various object types [S5]. Unlike typical MVS methods that require ground truth depth supervision, MVSplat is fully differentiable and trained from RGB images alone using only photometric loss [S4].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] Gaussian Splatting for real-time scene reconstruction: Methods, systems, and trade-offs, https://doi.org/10.1016/j.patcog.2026.115120, fetched 2026-10-26
[S4] MVSplat: Efficient 3D Gaussian Splatting from Sparse Multi-view Images, https://doi.org/10.1007/978-3-031-72664-4_21, fetched 2026-10-26
[S5] Lightplane: Highly-Scalable Components for Neural 3D Fields, https://doi.org/10.1109/3dv66043.2025.00058, fetched 2026-10-29
