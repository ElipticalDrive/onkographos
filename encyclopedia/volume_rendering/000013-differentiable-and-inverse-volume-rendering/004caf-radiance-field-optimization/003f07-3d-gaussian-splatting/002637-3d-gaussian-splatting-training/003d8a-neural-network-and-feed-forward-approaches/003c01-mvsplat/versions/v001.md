# MVSplat

Also called: Feed-forward 3DGS, Single-pass 3DGS

## Overview
MVSplat is a feed-forward deep neural network that directly regresses 3D Gaussian Splatting (3DGS) parameters from sparse multi-view images in a single pass, bypassing the per-scene gradient descent optimization loop required by standard 3DGS [S2, unofficial]. It builds upon PixelSplat by integrating multi-view feature extraction via transformers and cost volumes, which are subsequently processed by a U-Net decoder [S1]. This approach distinguishes itself from iterative inverse rendering methods by offering immediate inference without scene-specific training.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ sparse multi-view images and their corresponding camera parameters (intrinsics and extrinsics).
*   **Output:** A set of 3D Gaussian parameters for each pixel/ray, including mean position ($\mu$), covariance ($\Sigma$), opacity ($\alpha$), and spherical harmonics coefficients for color ($C$).

**Data Structures:**
The network processes features into a latent volume. The U-Net decoder outputs a tensor of shape $(H, W, N_{params})$, where $N_{params}$ corresponds to the dimensionality of the Gaussian parameter set (typically 14-15 values per Gaussian: 3 for mean, 6 for covariance/rotation, 1 for opacity, and remaining for SH coefficients).

**Method Steps:**
1.  **Multi-view Feature Extraction:** Extract features from each input image using a shared encoder (typically a CNN or Transformer backbone).
2.  **Epipolar Transformer / Cost Volume Construction:** Align features between views using epipolar constraints. PixelSplat uses an epipolar transformer to generalize features between two views [S1]. MVSplat extends this to multi-view by combining feature extraction through transformers and constructing cost volumes that encode geometric consistency across all input views [S1].
3.  **U-Net Decoding:** The aggregated cost volume and multi-view features are passed through a U-Net architecture. The U-Net upsamples the latent representation to the image resolution, predicting the Gaussian parameters for each pixel [S1].
4.  **Parameter Regression:** The output of the U-Net is interpreted as the parameters for 3D Gaussians. These Gaussians are then rasterized using standard differentiable 3DGS rasterization to produce the final image [S2, unofficial].

**Formulas:**
Let $I_i$ be the $i$-th input image and $P_i$ be its camera pose.
The network function $F$ is defined as:
$$ \{\theta_j\}_{j=1}^{M} = F(I_1, ..., I_N, P_1, ..., P_N) $$
where $\theta_j$ represents the parameters of the $j$-th Gaussian and $M$ is the number of Gaussians (typically $H \times W$).

**Parameters:**
*   **Number of Views:** Typically sparse (e.g., 2-4 views), though the architecture supports more.
*   **Network Depth:** Determined by the U-Net configuration (number of encoder/decoder blocks).

**Edge Cases and Pitfalls:**
*   **Scale Ambiguity:** Without known camera poses or metric depth, scale ambiguity may arise. NoPoSplat addresses this by using a canonical coordinate system and intrinsics token embeddings [S1]. MVSplat assumes known poses.
*   **Sparse View Limitations:** Performance may degrade with very sparse views or large baseline gaps if the cost volume construction fails to establish sufficient correspondence.

## Subtopics

### NoPoSplat
Eliminates the need for known camera poses by using a canonical coordinate system anchored to a single input view and intrinsics token embeddings to resolve scale ambiguity. Choose when camera poses are unavailable. [NoPoSplat](9527-noposplat.md)

### GS-LRM
Tokenizes 2-4 images and camera parameters, concatenates them, and passes them through a transformer block whose output tokens are decoded into Gaussian parameters. Choose for transformer-centric architectures. [GS-LRM](9528-gslrm.md)

### PixelSplat
The predecessor that incorporates an epipolar transformer for two views and probabilistic Gaussian prediction along camera-pixel rays. Choose as a baseline for two-view stereo. [PixelSplat](9529-pixelsplat.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per scene after training, as it is a single forward pass. Training time is significant but offline.
*   **Memory Complexity:** Depends on the U-Net and transformer size. Inference memory is dominated by the Gaussian storage ($O(H \times W)$).
*   **Performance:** Enables real-time rendering (900+ FPS mentioned for similar feed-forward methods like RadSplat [S2, unofficial]) due to the absence of per-scene optimization.
*   **Strengths:** Instant inference, no per-scene tuning, robust to initialization issues compared to SfM-based methods.
*   **Limitations:** Quality may be lower than iteratively optimized 3DGS, especially in complex geometries or with very sparse views. Generalization to unseen scene distributions depends on training data diversity.

## Background
MVSplat was introduced as an extension of PixelSplat, aiming to improve multi-view stereo by leveraging transformers and cost volumes for better feature alignment [S1]. It represents a shift from iterative inverse rendering to direct feed-forward prediction in the context of 3D Gaussian Splatting [S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
