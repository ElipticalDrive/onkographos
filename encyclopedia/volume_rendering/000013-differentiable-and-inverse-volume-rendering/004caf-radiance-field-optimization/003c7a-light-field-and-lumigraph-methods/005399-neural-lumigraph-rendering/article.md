# Neural Lumigraph Rendering

Also called: Deep Surface Light Field, Neural Radiance Fields with SDF export, Neural Surface Light Field, SIREN-based Neural Rendering

## Overview
Neural Lumigraph Rendering (NLR) is a neural rendering framework that jointly optimizes an implicit Signed Distance Function (SDF) for shape and a radiance field for appearance, supervised exclusively by posed 2D images [S1, S4]. It distinguishes itself from standard volumetric rendering methods like NeRF by exporting a triangular mesh with view-dependent textures, enabling real-time rasterization via OpenGL while maintaining state-of-the-art image quality [S1, S4]. Unlike [Local Light Field Fusion](1124-local-light-field-fusion.md) which relies on explicit light field parameterization, NLR learns an implicit neural representation that is subsequently discretized for rendering.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $m$ multi-view 2D images $I \in \mathbb{R}^{m \times w \times h \times 3}$ and corresponding binary object masks $M \in \mathbb{R}^{m \times w \times h}$ (where 1 marks foreground) [S1, S4]. Camera poses are defined by view matrix $V \in \mathbb{R}^{4 \times 4}$ and projection matrix $P \in \mathbb{R}^{4 \times 4}$ [S1, S4].
*   **Outputs:** Optimized parameters $\theta$ (SDF) and $\phi$ (appearance); optionally, a triangular mesh and projective textures for real-time rendering [S1, S4].

**Data Structures:**
*   **SDF Network $S$:** A SIREN (Sinusoidal Representation Network) MLP with 5 layers and 256 hidden units each [S1, S4]. Input: position $x \in \mathbb{R}^3$. Output: scalar $S(x; \theta) \in \mathbb{R}$ [S1, S4].
*   **Radiance Field Network $E$:** A SIREN MLP with 5 layers and 256 hidden units each [S1, S4]. Input: position $x \in \mathbb{R}^3$, view direction $r_d \in \mathbb{R}^3$, and local normal $n \in \mathbb{R}^3$. Output: RGB color $E(x, r_d, n; \theta, \phi) \in \mathbb{R}^3$ [S1, S4].
*   **Fourier Features:** The view direction $r_d$ is embedded using Fourier features $\{\sin(2k\pi r_d), \cos(2k\pi r_d) \mid k \in 1 \dots 4\}$ before entering $E$ [S1, S4].

**Numbered Steps:**

1.  **Ray Generation:** For a pixel at relative projection plane location $u \in [-1, 1]^2$, compute ray origin $r_o$ and direction $r_d$ [S1, S4]:
    $$ r_o = (V^{-1} \cdot [0, 0, 1, 0]^T)_{x,y,z} $$
    $$ r_d = \nu((P \cdot V)^{-1} \cdot [u_x, u_y, 0, 1]^T) $$
    where $\nu(\omega) = \omega_{x,y,z} / \|\omega_{x,y,z}\|$ is vector normalization [S1, S4].

2.  **Sphere Tracing:** Find the zero-level set $S_0$ closest to the camera along the ray using iterative updates [S1, S4]:
    $$ x_0 = r_o $$
    $$ x_{i+1} = x_i + S(x_i)r_d $$
    The surface point is $S_0 = \{x_n \mid S(x_n) \approx 0\}$ for step count $n=16$ [S1, S4]. A small residual $|S(x_n)| < 0.005$ is tolerated [S1, S4]. To save memory, gradients are retained only for the last step $n$ [S1, S4].

3.  **Normal Computation:** Compute the local normal $n$ via automatic differentiation of the SDF at the surface point:
    $$ n = \nabla_x S(x_n; \theta) $$
    [S1, S4].

4.  **Appearance Sampling:** Sample the radiance field at the surface point:
    $$ c = E(x_n, r_d, n; \theta, \phi) $$
    [S1, S4].

5.  **Loss Calculation:** Minimize the total loss $L = L_R + w_E L_E + w_M L_M + w_S L_S$ [S1, S4]:
    *   **Reconstruction Loss ($L_R$):** L1 error for foreground pixels $U_f = U \cap S_0 \cap \{U \mid M_U = 1\}$ [S1, S4]:
        $$ L_R = \frac{1}{|U|} \sum_{c \in I_{U_f}} |E(x, r_d, n; \theta, \phi) - c| $$
        [S1, S4].
    *   **Eikonal Loss ($L_E$):** Enforces metric properties for sphere tracing by regularizing the gradient of $S$ to unit length. Random points $x_r$ are uniformly sampled from a cube encapsulating the object's bounding unit radius sphere [S1, S4]:
        $$ L_E = \frac{1}{|U|} \sum_{x_r} (\|\nabla_x S(x_r; \theta)\|_2^2 - 1)^2 $$
        [S1, S4].
    *   **Soft Mask Loss ($L_M$):** Restricts coarse shape to object mask boundaries for non-foreground pixels $U \setminus U_f$. $S_{min}$ is the minimum SDF value along the ray [S1, S4]:
        $$ L_M = \frac{1}{\alpha |U|} \sum_{m \in M_{U \setminus U_f}} \text{BCE}(\text{sigmoid}(-\alpha S_{min}), m) $$
        where $\alpha$ is a softness parameter [S1, S4].
    *   **Smoothness Loss ($L_S$):** Linearizes angular behavior to prevent overfitting [S1, S4]:
        $$ L_S = \frac{1}{|U|} \sum \|\nabla^2_{r_d} E(x, r_d, n; \theta, \phi)\|_2^2 $$
        [S1, S4].

6.  **Optimization:** Parameters $\theta$ and $\phi$ are optimized using the Adam solver with initial learning rate $10^{-4}$, decreased by factor of 2 every 40,000 batches, for 150,000 batches [S1, S4]. Batch size is 50,000 rays [S1, S4]. Weights: $w_E = 0.1$, $w_M = 100$, $w_S = 0.01$ [S1, S4].

7.  **Mesh Extraction (Optional for Real-Time):**
    *   Voxelize $S$ at $512^3$ resolution and extract mesh using marching cubes [S1, S4].
    *   Offset the iso-surface by 0.5% of the object radius in the outside direction [S1, S4].
    *   Resample $E$ to synthesize projective textures $T_i$ for $N$ camera poses [S1, S4].

8.  **Real-Time Rendering (NLR-RAS):**
    *   Rasterize the mesh using OpenGL [S1, S4].
    *   Compute angles $\tau_i$ between the rendering camera ray and rays to each projective texture viewpoint [S1, S4].
    *   Blend contributions from the first $k=5$ textures sorted by $\tau_i$ ascending [S1, S4]:
        $$ \hat{w}_i = \frac{1}{\tau_i} \left(1 - \frac{\tau_i}{\tau_k}\right) $$
        $$ w_i = \frac{\hat{w}_i}{\sum_{j=1}^k \hat{w}_j} $$
        [S1, S4].
    *   Discard occluded textures by setting $w_i=0$ if the pre-rendered depth indicates occlusion [S1, S4].

**Parameters:**
*   $n=16$: Sphere tracing steps [S1, S4].
*   $w_E = 0.1, w_M = 100, w_S = 0.01$: Loss weights [S1, S4].
*   $\alpha$: Softness parameter for mask loss [S1, S4].
*   Hidden units: 256 per layer [S1, S4].
*   Layers: 5 [S1, S4].

**Edge Cases and Pitfalls:**
*   **Overfitting:** SIRENs tend to overfit to training views. Mitigated by large batch size (50,000) and uniform ray distribution [S1, S4].
*   **High-Frequency Artifacts:** Large values of $w_S$ cause high-frequency artifacts in $S$ [S1, S4].
*   **Initialization:** $S$ is initialized to a unit sphere of radius 0.5 by pretraining to a procedural shape [S1, S4].

## Subtopics

### [NLR++ Training Loss and Supervision](15227-nlr-training-loss-and-supervision.md)
Defines the end-to-end differentiable optimization objective for the Neural Lumigraph Representation (NLR++), which jointly reconstructs shape via a Signed Distance Function (SDF) and appearance via a radiance field from posed 2D images. It distinguishes itself from standard volumetric rendering losses by introducing a soft mask loss on the SDF minimum along rays and a specific training schedule that decouples shape approximation from feature refinement to accelerate convergence.

## Operating characteristics
*   **Time Complexity:** Training is slower than some baselines; NLR is slower to converge in shape reconstruction than MetaNLR++ and IDR [S3, unofficial].
*   **Memory Complexity:** Model size is compact. On DTU scan 65 at 1600x1200, NLR-ST is 2.07 MB and NLR-RAS is 34.68 MB, compared to NeRF's 2.27 MB and IDR's 11.13 MB [S4].
*   **Performance:** NLR-ST (sphere-traced) renders at 13s per image on DTU scan 65, while NLR-RAS (rasterized) achieves real-time performance (>60 fps at HD) [S4]. This is 10x faster than NeRF (150s) [S4].
*   **Image Quality:** NLR-RAS achieves 33.33 dB PSNR on DTU Scan 65, comparable to NeRF (34.87 dB) and superior to Colmap (32.13 dB) and IDR (33.50 dB) [S4].
*   **Strengths:** Real-time rendering capability via mesh export; high image quality; compact model size; view-dependent textures [S1, S4].
*   **Limitations:** Slower convergence during training compared to some implicit surface methods; requires posed images and masks [S1, S4].

## Background
Neural Lumigraph Rendering was introduced by Kellnhofer et al. in CVPR 2021 [S1, S4]. It builds on sinusoidal representation networks (SIREN) [S1, S4] and unstructured lumigraph rendering techniques from Buehler et al. [S4]. Fourier features for angular resolution are adopted from prior work [S4]. The method was implemented in PyTorch and rendered using OpenGL [S4].

## Sources
[S1] Neural Lumigraph Rendering, https://doi.org/10.1109/cvpr46437.2021.00427, fetched 2026-10-26
[S2] ICARUS: A Specialized Architecture for Neural Radiance Fields Rendering, https://doi.org/10.48550/arxiv.2203.01414, fetched 2026-10-26
[S3] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
[S4] Neural lumigraph rendering, https://openaccess.thecvf.com/content/CVPR2021/html/Kellnhofer_Neural_Lumigraph_Rendering_CVPR_2021_paper.html, fetched 2026-10-31
