# Neural Volumes

Also called: Deformable Volumes, Neural Volumes: Learning Dynamic Renderable Volumes from Images, Volumetric Reconstruction with Ray Potentials

## Overview
Neural Volumes (NV) is a method for synthesizing novel views of dynamic scenes by optimizing a deep 3D convolutional network to predict a discretized RGB$\alpha$ voxel grid and a 3D warp field [S1, S2]. It solves the problem of rendering semi-transparent, dynamic objects (such as smoke or hair) from sparse multi-view video input without requiring explicit geometry reconstruction or depth sensors [S2]. Unlike implicit radiance fields that map coordinates to color via MLPs, NV uses an explicit voxel grid, trading memory for rendering speed and interpretability [S1, S2]. It sets itself apart from static voxel methods like DeepVoxels by introducing a learnable warp field to model non-rigid motion and from MLP-based methods like NeRF by avoiding the prohibitive computational cost of evaluating deep networks at every ray-marching step [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $K$ synchronized, calibrated RGB images from different viewpoints (typically $K=3$ orthogonal views) for each time instant [S2].
*   **Outputs:** A rendered RGB$\alpha$ image for a novel camera pose [S1, S2].

**Data Structures:**
1.  **Latent Code ($z$):** A vector in $\mathbb{R}^{256}$ representing the state of the scene at a given time instant [S2]. In the Variational Autoencoder (VAE) formulation, the encoder outputs parameters $\mu$ and $\sigma$ for a diagonal Gaussian distribution, from which $z$ is sampled via the reparameterization trick [S2].
2.  **Conditioning Variable ($c$):** An $L$-dimensional vector concatenated with $z$ to allow conditional decoding (e.g., view-dependent appearance) [S2].
3.  **Voxel Grid ($V$):** An explicit 3D grid of resolution $D \times D \times D$ (typically $128^3$) storing RGBA values [S1, S2]. The volume function is defined as $\mathbf{V}: \mathbb{R}^3 \rightarrow \mathbb{R}^4$, where $\mathbf{V}(\mathbf{x}) = (\mathbf{V}_{\text{rgb}}(\mathbf{x}), \mathbf{V}_{\alpha}(\mathbf{x}))$ [S2]. $\mathbf{V}_{\text{rgb}} \in \mathbb{R}^3$ is color and $\mathbf{V}_{\alpha} \in \mathbb{R}$ is differential opacity in $[0, \infty]$ [S2].
4.  **Warp Grid ($W$):** A 3D grid of resolution $32^3$ storing inverse warp vectors [S1]. The warp field allows sampling color/opacity from a deformed location within the template grid [S2].

**Algorithm Steps:**

1.  **Encoding:**
    *   Input images (downsampled, e.g., $334 \times 512$) are passed through dedicated CNN branches for each of the $K$ cameras [S2].
    *   Features are concatenated and encoded to produce the latent code $z$ (and $\mu, \sigma$ if using VAE) [S2].
    *   Conditioning variable $c$ is concatenated with $z$ [S2].

2.  **Decoding (Voxel Generation):**
    *   The decoder network takes $[z; c]$ as input and outputs the Voxel Grid $V$ and Warp Grid $W$ [S1, S2].
    *   *Decoder Architectures:*
        *   **Convolutional Decoder:** Uses 3D transposed convolutions to upsample a $1 \times 1 \times 1$ feature cube to the full resolution grid (e.g., $128^3$) [S4, unofficial].
        *   **Linear Basis Decoder:** Uses fully connected layers to directly generate a flattened vector representing the grid [S4, unofficial].
    *   Activation functions (e.g., softplus) ensure non-negative RGBA values [S4, unofficial].

3.  **Warping (Deformable Volumes):**
    *   Instead of sampling directly from $V$, the system samples from a warped version of the volume to model motion and increase effective resolution [S2].
    *   For a point $\mathbf{x}$, the inverse warp $\mathbf{W}^{-1}(\mathbf{x})$ is computed. The sources describe two approaches:
        *   *Simple Warp:* Sample the warp vector at $\mathbf{x}$ to get the source location [S2].
        *   *Affine Warp Spatial Mixing:* Use $N$ affine transformations $A_i^{-1}$ with weights $w_i$ sampled in warp space:
            $$ \mathbf{W}^{-1}(\mathbf{x}) = \sum_{i=1}^{N} w_i(\mathbf{A}_i^{-1}(\mathbf{x})) \times \mathbf{A}_i^{-1}(\mathbf{x}) $$
            where $\mathbf{A}_i^{-1}$ includes rotation $R_i$, scale $s_i$, and translation $t_i$ [S4, unofficial].
    *   The effective volume value at $\mathbf{x}$ is $\mathbf{V}_{\text{eff}}(\mathbf{x}) = \mathbf{V}(\mathbf{W}^{-1}(\mathbf{x}))$ [S4, unofficial].

4.  **Differentiable Ray Marching:**
    *   For each camera ray, color and opacity are accumulated front-to-back using alpha blending [S4, unofficial]:
        $$ I_{\text{rgb}}(p) = \sum_{j} \mathbf{V}_{\text{rgb}}(r_j) \cdot \mathbf{V}_{\alpha}(r_j) \cdot \prod_{k<j} (1 - \mathbf{V}_{\alpha}(r_k)) $$
        $$ I_{\alpha}(p) = 1 - \prod_{j} (1 - \mathbf{V}_{\alpha}(r_j)) $$
    *   Here, $r_j$ are the sample points along the ray [S4, unofficial].

5.  **Optimization:**
    *   The network is trained end-to-end by minimizing the reconstruction loss against input images [S2].
    *   **Loss Function:**
        $$ \mathcal{L}(\theta) = \| I_{\text{rgb}} - I^*_{\text{rgb}} \|_2^2 + \beta \cdot D_{KL}(z \| \mathcal{N}(0,1)) + \lambda_{tv} J_{tv}(\mathbf{V}_{\alpha}) + \lambda_B J_B(I_{\alpha}) $$
        *   $D_{KL}$: KL-divergence regularization for the latent space [S2].
        *   $J_{tv}$: Total variation regularization on log-opacity to encourage sharp boundaries [S4, unofficial].
        *   $J_B$: Beta distribution prior on exit opacity to encourage binary transparency (fully transparent or opaque) [S4, unofficial].

**Parameters:**
*   Voxel Grid Resolution: $128^3$ [S1].
*   Warp Grid Resolution: $32^3$ [S1].
*   Latent Dimension: 256 [S2].
*   Input Views: $K=3$ (frontal, left, right) [S2].
*   Training Time: At least 12 hours per scene [S1].

**Edge Cases and Pitfalls:**
*   **Background:** NV assumes the object lies entirely within a bounded volume in front of a distinct background, which must be captured separately without the object [S1].
*   **Resolution Limits:** The explicit $128^3$ grid prevents scaling to represent fine details at high resolutions, leading to blurry results compared to higher-resolution methods [S1, S3].
*   **MLP Alternative:** Using an MLP decoder instead of a voxel grid is computationally prohibitive for real-time ray marching due to the need to evaluate the network at every sample point [S2][S4, unofficial].

## Subtopics

### [Voxel Grid Decoders](2219-voxel-grid-decoders.md)
Models the volume as a discrete 3D grid, trading memory for speed by avoiding on-the-fly MLP evaluation.

### [RenderNet Architecture](6691-rendernet-architecture.md)
Separates 3D feature extraction and 2D image decoding, using view-space resampling.

### [GRU Volume Fusion](7543-gru-volume-fusion.md)
Incrementally fuses local radiance fields into a global sparse field using recurrent updates.

## Operating characteristics
*   **Time Complexity:** Training is slow, requiring at least 12 hours per scene [S1]. Inference is real-time once the volume is decoded, as it relies on fixed-function ray marching [S2].
*   **Memory Complexity:** High memory usage due to the explicit $128^3$ voxel grid and $32^3$ warp grid [S1].
*   **Performance:** Captures reasonably detailed volumetric geometry and appearance for dynamic scenes [S1]. However, it produces blurry results compared to methods with higher effective resolution or adaptive sampling [S3].
*   **Strengths:** Handles semi-transparent materials (smoke, hair) and dynamic non-rigid motion; interpretable volume; real-time inference [S2].
*   **Limitations:** Limited by fixed voxel resolution; requires bounded volume with separate background capture; slow training [S1, S2].
*   **When to use:** Novel view synthesis of dynamic, semi-transparent objects with sparse multi-view input.
*   **When not to use:** Scenes requiring high-frequency detail beyond $128^3$ resolution; scenes with complex background interactions; static scenes where simpler methods suffice.

## Background
Neural Volumes was introduced by Gkioxari et al. in 2019 [S2]. It builds on the concept of "ray potentials" used in reconstruction but shifts the focus to rendering by modeling voxel transparency rather than occupancy probability [S2]. It addresses the limitations of static voxel methods (like DeepVoxels) by introducing a warp field for motion modeling [S2]. The method is a baseline for subsequent dynamic neural rendering techniques [S3].

## Sources
[S1] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S2] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
[S3] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S4] Neural Volumes: 从图像中学习动态可渲染体, https://www.alphaxiv.org/zh/abs/1906.07751, fetched 2026-10-30 (unofficial)
