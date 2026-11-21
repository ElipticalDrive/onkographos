# NeuS

Also called: NeuS, Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction

## Overview
NeuS is a neural surface reconstruction method that represents a scene geometry as the zero-level set of a learnable Signed Distance Function (SDF) and trains it using a novel volume rendering scheme [S1, unofficial][S3]. It solves the problem of inherent geometric bias (errors) in reconstructed surfaces that occurs when standard volume rendering is naively applied to SDF-associated density [S1, unofficial][S2]. Unlike [Surface NeRF](7385-surface-nerf.md) which uses occupancy grids, or [VolSDF](10715-volsdf.md) which uses a fixed variance, NeuS introduces an unbiased volume rendering formulation with a trainable sharpness parameter, allowing for high-quality surface extraction without foreground mask supervision [S1, unofficial][S2].

## How it works
NeuS builds on [Radiance-field optimization](44-radiance-field-optimization.md) by replacing the density-outputting MLP with an SDF-outputting MLP [S3].

**Inputs and Outputs:**
*   **Input:** A set of posed RGB images $\{I_k\}$ of a 3D object.
*   **Output:** A reconstructed triangular mesh extracted from the learned SDF zero-level set, and rendered novel views.

**Data Structures:**
The scene is represented by two Multi-Layer Perceptrons (MLPs) [S1, unofficial]:
1.  **SDF Network ($f$):** Maps spatial position $\mathbf{x} \in \mathbb{R}^3$ to a scalar signed distance value $d = f(\mathbf{x})$ [S1, unofficial][S4].
    *   Architecture: 8 hidden layers, hidden size 256.
    *   Activation: Softplus with $\beta=100$ (replacing ReLU) [S1, unofficial].
    *   Skip connection: From input to the output of the 4th layer [S1, unofficial].
    *   Output: Tuple $(d, \mathbf{g})$ where $d$ is the SDF value and $\mathbf{g} \in \mathbb{R}^{15}$ is a geometry feature vector [S4].
2.  **Color Network ($c$):** Maps position $\mathbf{x}$, viewing direction $\mathbf{v}$, normal $\mathbf{n}$, and feature $\mathbf{g}$ to RGB color [S1, unofficial].
    *   Architecture: 4 hidden layers, hidden size 256.
    *   Inputs: Spatial location $\mathbf{p}$, view direction $\mathbf{v}$, normal $\mathbf{n} = \nabla f(\mathbf{p})$, and feature $\mathbf{g}$ [S1, unofficial].
    *   Positional Encoding: Applied to $\mathbf{p}$ (6 frequencies) and $\mathbf{v}$ (4 frequencies) [S1, unofficial].
3.  **Variance Network:** A single learnable parameter `variance` (or scalar $s$) controlling the sharpness of the SDF-to-alpha conversion [S6, unofficial].
    *   Inverse temperature: $inv\_s = \exp(variance \times 10.0)$ [S6, unofficial].
    *   Clipped to $[10^{-6}, 10^6]$ to prevent numerical instability [S6, unofficial].

**Numbered Steps:**

1.  **Ray Sampling:** For a camera ray $\mathbf{p}(t) = \mathbf{o} + t\mathbf{v}$, sample $n$ points. NeuS uses hierarchical sampling:
    *   Uniformly sample 64 points initially [S1, unofficial].
    *   Perform importance sampling for $k=4$ iterations, adding 16 points per iteration, totaling 128 points [S1, unofficial].
    *   For the 'w/o mask' setting, sample an extra 32 points outside the bounding sphere to represent the background using NeRF++ [S1, unofficial].
2.  **SDF and Gradient Query:** For each sampled point $\mathbf{p}_i$, query the SDF network to get $f(\mathbf{p}_i)$ and the geometry feature $\mathbf{g}_i$. Compute the normal $\mathbf{n}_i = \nabla f(\mathbf{p}_i) / \|\nabla f(\mathbf{p}_i)\|$ [S1, unofficial][S4].
3.  **Alpha Computation (Unbiased Volume Rendering):** Compute the opacity $\alpha_i$ for the segment between section points $q_i$ and $q_{i+1}$ using the cumulative distribution function (CDF) of the S-density [S1, unofficial].
    *   Define the S-density function (derivative of Sigmoid):
        $$ \phi_s(x) = \frac{s e^{-sx}}{(1 + e^{-sx})^2} $$
        where $s$ is a trainable parameter (or derived from `variance`) [S1, unofficial][S5].
    *   Define the Sigmoid CDF:
        $$ \Phi_s(x) = \frac{1}{1 + e^{-sx}} $$
    *   Compute alpha for segment $i$:
        $$ \alpha_i = \max\left( \frac{\Phi_s(f(q_i)) - \Phi_s(f(q_{i+1}))}{\Phi_s(f(q_i))}, 0 \right) $$
        [S1, unofficial]. Note: $10^{-5}$ is added to numerator and denominator to prevent division by zero [S6, unofficial].
    *   *Alternative Implementation (Instant-NSR-PL):* Uses a dynamic $inv\_s$ and cosine annealing for better convergence [S6, unofficial]:
        ```python
        def get_alpha(sdf, normal, dirs, dists, cos_anneal_ratio):
            inv_s = clip(exp(variance * 10.0), 1e-6, 1e6)
            true_cos = dot(dirs, normal)
            # Cosine annealing for early training stability
            iter_cos = -(relu(-true_cos * 0.5 + 0.5) * (1 - cos_anneal_ratio) + relu(-true_cos) * cos_anneal_ratio)
            estimated_next_sdf = sdf + iter_cos * dists * 0.5
            estimated_prev_sdf = sdf - iter_cos * dists * 0.5
            prev_cdf = sigmoid(estimated_prev_sdf * inv_s)
            next_cdf = sigmoid(estimated_next_sdf * inv_s)
            p = prev_cdf - next_cdf
            c = prev_cdf
            alpha = clip((p + 1e-5) / (c + 1e-5), 0.0, 1.0)
            return alpha
        ```
4.  **Color Computation:** Compute the color $c_i$ at the midpoint $p_i = \mathbf{o} + \frac{t_i + t_{i+1}}{2}\mathbf{v}$ using the Color Network with inputs $\mathbf{p}_i, \mathbf{v}, \mathbf{n}_i, \mathbf{g}_i$ [S1, unofficial].
5.  **Volume Rendering:** Accumulate color along the ray using standard alpha compositing:
    $$ C(\mathbf{o}, \mathbf{v}) = \sum_i T_i c_i $$
    where $T_i = \prod_{j < i} (1 - \alpha_j)$ is the transmittance [S1, unofficial].
6.  **Loss Calculation:** Minimize the total loss:
    $$ \mathcal{L} = \mathcal{L}_{\text{color}} + \beta \mathcal{L}_{\text{eikonal}} $$
    *   $\mathcal{L}_{\text{color}}$: Huber loss between rendered color and ground truth pixel [S4].
    *   $\mathcal{L}_{\text{eikonal}}$: Regularizes the SDF gradient magnitude to 1: $\mathcal{L}_{\text{eikonal}} = \frac{1}{mn} \sum_{k,i} (\|\mathbf{n}_{k,i}\| - 1)^2$ [S4].
7.  **Optimization:** Train using ADAM optimizer.
    *   Learning rate: Linear warmup from 0 to $5 \times 10^{-4}$ in first 5k iterations, then cosine decay to minimum $2.5 \times 10^{-5}$ [S1, unofficial].
    *   Total iterations: 300k [S1, unofficial].
    *   Weight normalization is used to stabilize training [S1, unofficial].

**Parameters:**
*   $s$ (or `variance`): Trainable sharpness parameter. Standard deviation of $\phi_s$ is $1/s$, approaching 0 as training converges [S1, unofficial].
*   $\beta$: Weight for Eikonal loss. Typical value not explicitly stated in sources, but described as indispensable [S4].
*   Softplus $\beta$: 100 [S1, unofficial].
*   `init_val`: Initial value for variance parameter [S6, unofficial].
*   `modulate`: Boolean to enable modulation of $inv\_s$ during training [S6, unofficial].

**Edge Cases and Pitfalls:**
*   **Geometric Bias:** Naive application of standard volume rendering to SDF density causes inherent geometric errors. NeuS formulation avoids this in the first-order approximation [S1, unofficial].
*   **Spectral Bias:** Random initialization causes axis-aligned artifacts due to positional encoding spectral bias. Geometric initialization avoids this [S1, unofficial].
*   **Textureless Regions:** Textureless areas are ambiguous and may fail to reconstruct correctly [S1, unofficial].
*   **Numerical Stability:** $inv\_s$ must be clipped to $[10^{-6}, 10^6]$ [S6, unofficial].

## Subtopics

### [HAME-NeRF](9325-hame-nerf.md)
HAME-NeRF is a two-stage framework for extracting high-fidelity, textured surface meshes from RGB images by defining volume density as a Secant Hyperbolic Function applied to a Signed Distance Function (SDF). It builds on [NeuS](6060-neus.md) by introducing a spatially-varying sharpness parameter $\beta(\mathbf{x})$ learned via an MLP, which enhances flexibility for complex surfaces compared to methods using fixed $\beta$ values. Unlike [NeuS](6060-neus.md) which focuses on differentiable volume rendering for surface representation, HAME-NeRF explicitly optimizes a mesh through differentiable rendering and iterative refinement, achieving superior geometric accuracy and visual quality on synthetic benchmarks compared to [NeRF2Mesh](52-nerf2mesh.md).

### [D-NeuS](10217-d-neus.md)
D-NeuS is a volume rendering-based neural implicit surface reconstruction method that recovers fine-level geometric details by extending NeuS with two additional loss functions: a geometry bias loss and a multi-view feature consistency loss. It addresses the ambiguity in geometry and appearance representation inherent in volume rendering by ensuring consistency between the rendered surface points (derived from alpha compositing) and the underlying signed distance field (SDF) zero-crossings. Unlike its parent NeuS, which suffers from geometry bias due to the non-linear transformation of SDF to density, D-NeuS explicitly penalizes this bias and enforces feature consistency on surface points rather than all sampled points, distinguishing it from siblings like HAME-NeRF and NeuS2.

### NeuS2
NeuS2 is a fast neural surface reconstruction approach for multi-view RGB input that achieves two orders of magnitude improvement in training speed over NeuS without compromising reconstruction quality, applicable to both static and dynamic scenes [S1, unofficial]. It sets itself apart by parameterizing the neural network-encoded Signed Distance Field (SDF) using multi-resolution hash tables of learnable feature vectors, as introduced in Instant-NGP, and by calculating second-order derivatives using a novel lightweight formula tailored to ReLU-based MLPs to leverage CUDA parallelism and reduce memory footprint [S1, unofficial].

### [HF-NeuS](12039-hf-neus.md)
HF-NeuS is an improvement on NeuS that separates low-frequency details into a base SDF and high-frequency details into a displacement function to increase reconstruction quality. It builds on [NeuS](6060-neus.md) by addressing the smoothness limitations of standard implicit SDF representations, which often fail to capture sharp geometric features.

### NeuDA
NeuDA is a neural implicit surface reconstruction method that utilizes hierarchical deformable anchors to encode local geometry, building on the NeuS [6060-neus.md] framework. It distinguishes itself by replacing standard positional encoding with Hierarchical Positional Encoding (HPE) and employing a shallower SDF network (4 layers vs. 8 for NeuS) [S1, unofficial]. This design achieves high-fidelity surface reconstruction with lower computational cost and sharper surfaces compared to siblings like [HAME-NeRF](9325-hame-nerf.md), [D-NeuS](10217-d-neus.md), [NeuS2](10220-neus2.md), and [HF-NeuS](12039-hf-neus.md) [S1, unofficial].

### SparseNeus
SparseNeus is a method improving NeuS for sparse-view SDF reconstruction using a geometry encoding volume with learnable image features as a hybrid representation method [S3].

## Operating characteristics
*   **Time Complexity:** NeuS is slower than standard NeRF because SDF queries require more network evaluations per ray (gradient computation) [S2]. Rendering a 1600x1200 image takes ~320 seconds on an Nvidia 2080Ti GPU in default settings, or ~60 seconds with sphere tracing acceleration [S1, unofficial]. On an A100 GPU at 1080p, it renders at 0.8 ± 0.2 FPS [S2].
*   **Memory Complexity:** Requires storing two MLPs (SDF and Color) and their gradients. The SDF network is 8 layers, Color is 4 layers [S1, unofficial].
*   **Performance:** On the NeRF Synthetic dataset, NeuS achieved PSNR = 28.4 ± 0.8 dB [S2]. On the DTU dataset, it achieved the most accurate surface reconstruction with Chamfer distance = 0.62 ± 0.08 mm [S2].
*   **Strengths:** Unbiased surface reconstruction; high geometric accuracy; works without foreground mask supervision; robust optimization for complex structures [S1, unofficial].
*   **Limitations:** Slow training and rendering speed compared to hash-encoded variants; struggles with textureless regions; sensitive to initialization (requires geometric init to avoid artifacts) [S1, unofficial][S2].
*   **When to use:** When high-quality, watertight surface meshes are required from multi-view images and training time is not the primary constraint.
*   **When not to use:** When real-time rendering is needed (use [Instant-NSR](12440-instant-nsr.md) or [NeuS2](10220-neus2.md)) or when input views are very sparse (use [SparseNeus](#sparseneus)) [S3].

## Background
NeuS was introduced in 2021 by Yen-Chen Lin et al. in the paper "NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction" [S1, unofficial][S3]. It was developed to address the limitations of previous implicit surface representations (like UNISURF) and the geometric bias inherent in applying standard NeRF volume rendering to SDFs [S1, unofficial]. The method combines the robustness of volume rendering with the accuracy of SDF-based surface extraction. It has become a foundational method for neural surface reconstruction, leading to numerous variants such as Instant-NSR, NeuS2, and HF-Neus [S3, S4].

## Sources
[S1] NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction, https://doi.org/10.48550/arxiv.2106.10689, fetched 2026-10-26 (unofficial)
[S2] Neural Rendering Techniques, https://doi.org/10.5281/zenodo.19614729, fetched 2026-10-26
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
[S5] Review of Key Technologies and Application Scenarios of Neural Rendering, https://doi.org/10.54254/2755-2721/2026.36116, fetched 2026-10-26
[S6] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S7] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
