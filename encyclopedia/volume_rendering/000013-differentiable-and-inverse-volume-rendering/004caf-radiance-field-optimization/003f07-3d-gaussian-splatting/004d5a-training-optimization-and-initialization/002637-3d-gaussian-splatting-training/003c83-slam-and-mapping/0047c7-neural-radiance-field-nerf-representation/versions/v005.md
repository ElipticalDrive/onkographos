# Neural Radiance Field (NeRF) Representation

Also called: INGP, Instant Neural Graphics Primitives, Photon Field Networks, Photon Fields

## Overview
Neural Radiance Field (NeRF) representation is a method for implicitly modeling 3D scenes using neural networks to enable novel view synthesis [S1][S2, unofficial]. It solves the problem of reconstructing a continuous volumetric appearance field from a sparse set of 2D images by optimizing a network to predict color and density at any 3D point and viewing direction [S1]. Unlike explicit representations such as [3D Gaussian Splatting](12052-3d-gaussian-splatting.md) which store millions of discrete blobs, NeRF stores the scene implicitly within a neural network, resulting in lower memory usage but significantly slower rendering and training times [S2, unofficial].

## How it works
The term "NeRF Representation" in the provided context encompasses two distinct methodologies found in the sources: the original implicit Neural Radiance Field (often associated with Instant Neural Graphics Primitives or INGP) and the explicit 3D Gaussian Splatting method [S1][S2, unofficial]. Additionally, [Photon Field Networks](12484-scene-nerf.md) represents a variant for volumetric global illumination. The following details cover the general NeRF/INGP implicit approach and the explicit Gaussian Splatting approach as described in the sources.

### Inputs and Outputs
*   **Inputs:** A set of $N$ images $I_1, \dots, I_N$ with known camera poses $\mathbf{T}_1, \dots, \mathbf{T}_N$ (typically obtained via Structure from Motion like COLMAP) [S2, unofficial]. For Photon Fields, inputs also include light source positions and phase function coefficients [S1].
*   **Outputs:** A neural network or explicit data structure that can render a novel view image $I_{rendered}$ from arbitrary camera poses and predict scene properties (density, color, radiance) [S1][S2, unofficial].

### Data Structures

#### Implicit NeRF / INGP (Photon Field Networks)
*   **Network Architecture:** A 5-layer Multi-Layer Perceptron (MLP) with 64 neurons per layer, ReLU activations for hidden layers, and no output activation [S1].
*   **Encoding:** Uses Hashgrid encoding for positions (3D) and directions (2D) [S1].
    *   Grid levels: 16.
    *   Features per level: 8.
    *   Scaling factor between levels: $2\times$.
    *   Hash table size: $2^{19}$ [S1].
    *   Phase coefficient $g$ is left unencoded [S1].
*   **Photon Field Specifics:** The network maps position $x \in [0,1]^3$, view direction $\vec{\omega}$, and phase coefficient $g \in [-1,1]$ to RGB radiance [S1].

#### Explicit 3D Gaussian Splatting
*   **Gaussian Blob:** Each Gaussian $i$ is defined by:
    *   Position: $\mu_i \in \mathbb{R}^3$ (center in 3D space) [S2, unofficial].
    *   Covariance: $\Sigma_i \in \mathbb{R}^{3 \times 3}$, constructed as $R S S^T R^T$ where $R$ is rotation and $S$ is scale, ensuring positive semi-definiteness [S2, unofficial].
    *   Opacity: $\alpha_i \in [0, 1]$ [S2, unofficial].
    *   Color: Stored as spherical harmonics coefficients to allow view-dependent color [S2, unofficial].
*   **Storage:** Each Gaussian stores approximately 59 learnable numbers [S2, unofficial].

### Numbered Steps

#### Implicit NeRF / INGP Training
1.  **Photon Tracing (for Photon Fields):** Generate training data by tracing photons from light sources. For each photon, randomly select a phase coefficient $g$ from a discrete set $G = \{-0.75, 0.0, 0.75\}$ [S1]. Use delta tracking to compute sample distances and interaction locations. Store photons past their first interaction in a photon map [S1].
2.  **KD-Tree Construction:** Construct a KD-tree on the GPU from the generated multi-phase photon maps to enable fast spatial queries [S1].
3.  **Sampling:** Generate training batches of size $2^{16}$. Each batch contains uniformly sampled grid coordinates $x \in [0,1]$, random uniform directions $\vec{\omega}$, and phase coefficients $g$ randomly selected from $G$ [S1].
4.  **Ground Truth Computation:** For each sample, compute the target radiance using a phase-specific k-nearest-neighbor (KNN) range query on the photon map. Use $K=1024$ neighbors and a maximum query radius $r$ [S1]. The radiance is estimated as:
    $$ L_i(x, \vec{\omega}) \approx \sum_{n=1}^K p_g(x, \vec{\omega}, \vec{\omega}_n) \frac{\Phi_n}{4\pi r^3} $$
    where $\Phi_n$ is the photon intensity and $p_g$ is the Henyey-Greenstein phase function [S1].
5.  **Target Encoding:** Encode the target radiance logarithmically to normalize it to $[0,1]$. Compute $L_i'(x, \vec{\omega})$ using precision parameter $\psi$ (typically 4-6). If $L_i < 10^{-\psi}$, clamp to 1 (which maps to 0 in the inverse transformation) [S1]:
    $$ L_i'(x, \vec{\omega}) = \begin{cases} -\frac{\log_{10}(L_i(x, \vec{\omega}))}{\psi} & \text{if } L_i(x, \vec{\omega}) > \frac{1}{10^{\psi}} \\ 1 & \text{otherwise} \end{cases} $$
6.  **Network Forward Pass:** Pass the encoded inputs (position, direction, phase) through the hashgrid encoder and the 5-layer MLP to predict the log-encoded radiance $L_i'$ [S1].
7.  **Loss Calculation:** Compare the predicted radiance with the ground truth target. (Note: Standard NeRF uses L1/SSIM loss on rendered images [S2, unofficial]; Photon Fields use a single loss term on the radiance prediction [S1]).
8.  **Optimization:** Backpropagate the error to update the network weights via gradient descent [S1][S2, unofficial].

#### Explicit 3D Gaussian Splatting Training
1.  **Initialization:** Start with photos and camera poses from Structure from Motion (e.g., COLMAP) to generate a sparse cloud of 3D points. Each point becomes the starting position of one Gaussian [S2, unofficial].
2.  **Projection:** Project every 3D Gaussian onto the camera image. A projected 3D Gaussian results in a 2D Gaussian (soft ellipse) on the screen. The 2D covariance is computed using the camera transformation and a local linear approximation of the projection [S2, unofficial].
3.  **Tiling:** Divide the screen into small tiles of 16x16 pixels. Assign each Gaussian to the tiles it overlaps [S2, unofficial].
4.  **Sorting:** Within each tile, sort the Gaussians by depth from front to back using a very fast GPU sort [S2, unofficial].
5.  **Rasterization/Blending:** For each pixel, blend the colors of the sorted Gaussians front-to-back using alpha compositing [S2, unofficial].
6.  **Loss Calculation:** Compare the rendered image with the real training photo using an L1 loss combined with a structural similarity (SSIM) term [S2, unofficial].
7.  **Optimization:** Backpropagate the error to update each Gaussian's position, shape (scale/rotation), opacity, and color directly via gradient descent [S2, unofficial].
8.  **Adaptive Density Control:**
    *   **Cloning:** In regions under-reconstructed with missing details, clone small Gaussians [S2, unofficial].
    *   **Splitting:** Where a single Gaussian is too large and covers too much detail, split it into two smaller ones [S2, unofficial].
    *   **Pruning:** Prune away Gaussians that become almost transparent [S2, unofficial].

### Formulas
*   **Photon Field Mapping:**
    $$ P_{\theta}(x, g, \vec{\omega}) = (R, G, B) $$
    where $x \in [0,1]^3$ is position, $g \in [-1,1]$ is the scattering coefficient, and $\vec{\omega}$ is the view direction [S1].
*   **Indirect Radiance Approximation:**
    $$ P_{\theta}(x, g, \vec{\omega}) \approx L_i(x, \vec{\omega}) = \int_{\Omega_{4\pi}} p_g(x, \vec{\omega}, \vec{\omega}_i) L(x, \vec{\omega}_i) d\omega_i $$
    where $L$ is total radiance and $p_g$ is the phase function [S1].
*   **Inverse Transformation:**
    $$ L_i(x, \vec{\omega}) = 10^{-L_i'(x, \vec{\omega})} \cdot \Psi $$
    where $\Psi$ is related to the precision parameter $\psi$ [S1].
*   **Gaussian Covariance Construction:**
    $$ \Sigma = R S S^T R^T $$
    ensuring the shape remains valid (positive semi-definite) [S2, unofficial].

### Parameters
*   **$\psi$:** Maximum desired floating point precision in digits beyond zero. Values of 4 to 6 are sufficient [S1].
*   **$G$:** Set of discrete phase coefficients. $G = \{-0.75, 0.0, 0.75\}$ is the best compromise for Photon Fields [S1].
*   **Batch Size:** $2^{16}$ for Photon Fields [S1].
*   **KNN Neighbors ($K$):** 1024 for Photon Fields [S1].
*   **Tile Size:** 16x16 pixels for Gaussian Splatting [S2, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** Radiance values below $10^{-\psi}$ are clamped, potentially losing very dim details [S1].
*   **Aliasing:** Gaussian Splatting suffers from aliasing when zooming in or out [S2, unofficial].
*   **Degeneracy:** Gaussian Splatting can produce floating artifacts and needle-like Gaussians in regions seen by few cameras [S2, unofficial].
*   **Surface Extraction:** Neither method directly provides a clean surface or mesh; Gaussian Splatting explicitly does not [S2, unofficial].

## Subtopics

### [Scene NeRF](12484-scene-nerf.md)
Scene NeRF is a method for isolating and rendering the static background of a scene by training a Neural Radiance Field (NeRF) exclusively on pixels identified as background via human segmentation masks. It differs from standard NeRF by explicitly excluding dynamic foreground elements (e.g., humans) from the training data and employing an emptiness regularizer to prevent artifacts in empty space. Choose this when dynamic foreground removal is required. [Scene NeRF](12484-scene-nerf.md)

### [NeRF-OSR](14477-nerf-osr.md)
NeRF-OSR is a neural radiance field approach for outdoor scene relighting that learns a neural representation of scene geometry, diffuse albedo, and illumination-dependent shadows from a set of images capturing the same site from different viewpoints and at different times, enabling simultaneous editing of both the scene’s lighting and viewpoint. Unlike standard NeRF which learns a Lambertian model under fixed lighting, NeRF-OSR decomposes the scene into spatial occupancy, illumination, shadowing, and diffuse albedo reflectance, trained in a self-supervised manner from outdoor data captured from various viewpoints and at different illuminations. It sets itself apart from siblings like [Scene NeRF](12484-scene-nerf.md) by providing explicit intrinsic decomposition (albedo, geometry, lighting) rather than just view-dependent appearance, allowing for physical relighting and albedo editing. Choose this for outdoor relighting and intrinsic decomposition. [NeRF-OSR](14477-nerf-osr.md)

### SLAIM
SLAIM is a SLAM method that introduces a coarse-to-fine tracking pipeline and improves photometric bundle adjustment for NeRF-based mapping [S3]. It maintains NeRF’s original volume density formulation while introducing a KL regularization over the ray termination distribution [S3]. It addresses the challenge of high-frequency renderings in NeRF hindering image alignment [S3].

### SNI-SLAM
SNI-SLAM is a dense NeRF-based RGB-D SLAM system for accurate real-time 3D semantic mapping [S3]. It introduces a hierarchical semantic encoding and cross-attention mechanism to enable mutual reinforcement between appearance, geometry, and semantics modalities [S3]. It proposes a novel one-way decoder design to enhance inter-modal information flow without reverse interference [S3].

### KN-SLAM
KN-SLAM is a NeRF-based SLAM method that integrates local feature correspondences for coarse pose initialization and joint optimization [S3]. It uses global image features and local matches for explicit loop detection [S3]. It performs pose graph optimization and global refinement of the neural map to ensure consistency [S3].

## Operating characteristics
*   **Time Complexity:** NeRF training takes many hours, while Gaussian Splatting reduces this to tens of minutes [S2, unofficial]. NeRF rendering takes seconds per image, while Gaussian Splatting achieves >100 FPS [S2, unofficial].
*   **Memory Complexity:** NeRF uses a few megabytes of memory for the network [S2, unofficial]. Gaussian Splatting uses hundreds of megabytes for millions of Gaussians [S2, unofficial].
*   **Strengths:** NeRF provides high-quality, continuous novel view synthesis with low memory footprint [S2, unofficial]. Gaussian Splatting offers real-time rendering and faster training [S2, unofficial].
*   **Limitations:** NeRF is slow to render and train [S2, unofficial]. Gaussian Splatting suffers from aliasing, floating artifacts, needle-like Gaussians, and cannot directly extract clean meshes [S2, unofficial].
*   **When to use:** Use NeRF for high-quality offline rendering or memory-constrained applications. Use Gaussian Splatting for real-time applications, VR, and rapid reconstruction [S2, unofficial].

## Background
*   **Origin:** The concept of Neural Radiance Fields was introduced by Mildenhall et al. [S1][S2, unofficial]. Instant Neural Graphics Primitives (INGP) optimized this for speed [S1]. 3D Gaussian Splatting was introduced by Kerbl and colleagues at SIGGRAPH 2023 [S2, unofficial].
*   **History:** NeRF has been widely adopted for novel view synthesis. Gaussian Splatting emerged as a faster, explicit alternative [S2, unofficial]. Variants like Scene NeRF and NeRF-OSR extend the method to specific domains like dynamic background removal and outdoor relighting [S1]. SLAM applications like SLAIM, SNI-SLAM, and KN-SLAM integrate NeRF representations into simultaneous localization and mapping pipelines [S3].
*   **Usage:** Used in virtual reality games, film, robotics, mapping, digital humans, ecommerce, and cultural heritage [S2, unofficial].

## Sources
[S1] Photon Field Networks for Dynamic Real-Time Volumetric Global Illumination, https://doi.org/10.1109/tvcg.2023.3327107, fetched 2026-10-26
[S2] Gaussian Splatting Explained: Real-Time 3D from Photos (with a Real Demo), https://www.youtube.com/watch?v=HABVlXlRqOs, fetched 2026-10-27 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
