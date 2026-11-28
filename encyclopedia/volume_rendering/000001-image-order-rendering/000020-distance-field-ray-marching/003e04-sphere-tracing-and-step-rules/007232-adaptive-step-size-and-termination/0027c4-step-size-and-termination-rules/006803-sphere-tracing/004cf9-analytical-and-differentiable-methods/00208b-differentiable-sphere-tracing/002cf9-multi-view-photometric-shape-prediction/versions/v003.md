# Multi-View Photometric Shape Prediction

Also called: Latent Code Optimization via Photometric Warping, Photometric Consistency Optimization for Shape, Pix2Vex, Pix2Vex Network, p2v

## Overview
Predicts 3D shape from multiple color images by minimizing cross-view photometric consistency loss without requiring segmentation masks [S1]. It differs from encoder-based methods like PMO by using random initialization and implicit representations, making it robust to initialization noise and camera focal length changes [S1]. It also differs from Pix2Vex [S2, unofficial] by optimizing a latent code for a specific instance rather than training a network to predict vertex coordinates from a fixed topology.

## How it works
**Inputs:** A set of $N$ color images $\{I_i\}_{i=0}^{N-1}$ with known camera poses.
**Outputs:** An optimized latent code $\mathbf{z}$ representing the 3D shape.

**Data Structures:**
*   $\mathbf{z}$: Latent code vector.
*   $f(\mathbf{z})$: Implicit Signed Distance Function (SDF) decoder (e.g., DeepSDF) [S1].
*   $\mathcal{R}_d^i(\mathbf{z})$: Rendered depth image for view $i$ using differentiable sphere tracing.
*   $I_{j \rightarrow i}$: Warped image from view $j$ to view $i$.

**Steps:**
1.  Initialize $\mathbf{z}$ randomly [S1].
2.  For each view $i \in \{0, \dots, N-1\}$:
    a. Render depth image $D_i = \mathcal{R}_d^i(f(\mathbf{z}))$ [S1].
3.  For each view $i$ and each neighbor $j \in \mathcal{N}_i$:
    a. Warp image $I_j$ to view $i$ using $D_j$, camera poses, and $D_i$ to get $I_{j \rightarrow i}$ [S1].
    b. Compute photometric difference between $I_i$ and $I_{j \rightarrow i}$ [S1].
4.  Minimize the total photometric loss with respect to $\mathbf{z}$ [S1].

**Formula:**
$$ \arg \min_{\mathbf{z}} \sum_{i=0}^{N-1} \sum_{j \in \mathcal{N}_i} \| I_i - I_{j \rightarrow i}(\mathcal{R}_d^i(f(\mathbf{z}))) \| $$
where $\mathcal{R}_d^i$ is the rendered depth image at view $i$, $\mathcal{N}_i$ are the neighboring images of $I_i$, and $I_{j \rightarrow i}$ is the warped image from view $j$ to $i$ using the rendered depth [S1]. No mask is required [S1].

**Parameters:**
*   $\mathbf{z}$ initialization: Random vector [S1].
*   Neighbor set $\mathcal{N}_i$: Not specified in sources [S1].

**Edge Cases:**
*   Initialization: Random initialization is used, which is robust to noise [S1].
*   Masks: No segmentation masks are required [S1].

## Subtopics

### 3D Object Prediction with Texture and Light
A variant where the network predicts vertex positions, a texture map, and lighting information [S3, unofficial]. It uses a W-GAN with gradient penalty and perceptual loss for training stability [S3, unofficial]. The objective is:
$$ \theta^*, \phi^* = \arg \min_\theta \arg \max_\phi (\lambda_{adv} L_{adv} - \lambda_{gp} L_{gp}) + \lambda_{per} L_{percep} + L_1 $$
where $L_{adv}$ and $L_{gp}$ are adversarial and gradient penalty losses, and $L_{percep}$ is a perceptual loss using pre-trained VGG and discriminator features [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On ShapeNet, it achieves a median Chamfer Distance of 5.37 (partial dense) and 12.05 (sparse) compared to DeepSDF's 13.76 and 15.77 [S1]. For multi-view prediction, it achieves 0.919 Chamfer Distance with random initialization, outperforming PMO's 1.187 [S1].
*   **Strengths:** Robust to initialization noise and camera focal length changes [S1]. Does not require segmentation masks [S1].
*   **Limitations:** Requires known camera poses [S1]. Optimization may be slow due to iterative gradient descent [S1].

## Background
Originated from the DIST paper [S1] and Pix2Vex [S2, unofficial]. The method leverages differentiable sphere tracing to enable geometry-based reasoning from multi-view color images [S1]. It was developed to address the sensitivity of encoder-based methods like PMO to domain shifts and initialization [S1].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] Pix2Vex: Image-to-Geometry Reconstruction using a Smooth Differentiable Renderer, https://doi.org/10.48550/arxiv.1903.11149, fetched 2026-10-26 (unofficial)
[S3] Learning to Predict 3D Objects with an Interpolation-based Differentiable Renderer, https://doi.org/10.48550/arxiv.1908.01210, fetched 2026-10-26 (unofficial)
