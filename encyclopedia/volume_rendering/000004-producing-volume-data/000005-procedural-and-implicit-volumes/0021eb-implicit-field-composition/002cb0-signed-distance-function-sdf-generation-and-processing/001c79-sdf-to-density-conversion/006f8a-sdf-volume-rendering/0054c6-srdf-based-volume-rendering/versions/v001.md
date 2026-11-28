# SRDF-based Volume Rendering

Also called: RS. Den, Ray-Distance Volume Rendering, Ray-Specific SRDF Volume Rendering, SRDF

## Overview
SRDF-based Volume Rendering is a neural scene reconstruction technique that models volume density using a Ray-Specific Signed Distance Function (SRDF) rather than a global Signed Distance Function (SDF) [S1]. It solves the problem of false local maxima in volume density caused by surfaces not intersecting the camera ray, which leads to inaccurate depth and noisy surface reconstruction in multi-object indoor scenes [S1]. Unlike standard SDF-based methods where density is derived from a global field, SRDF computes the shortest distance to the surface *along the sampled ray direction*, ensuring that density peaks only around the actual surface intersection [S1]. This method is distinct from [NeUDF](3821-neudf.md) by explicitly decoupling the ray-specific density modeling from the global surface representation [S1].

## How it works
The method predicts both a global SDF for surface extraction and a ray-specific SRDF for volume density. It uses three Multi-Layer Perceptrons (MLPs): a geometry MLP ($f_g$), an SRDF MLP ($f_s$), and a color MLP ($f_c$) [S1].

### Inputs and Outputs
*   **Inputs:** Camera ray origin $\mathbf{o}$, ray direction $\mathbf{r}$, and $N$ sampled points $\{\mathbf{p}_i = \mathbf{o} + z_i \mathbf{r}\}$ along the ray, where $z_i$ is the depth [S1].
*   **Outputs:** Rendered color $\mathbf{C}(\mathbf{r})$, rendered depth $D$, rendered normal $\mathbf{N}$, and a watertight mesh extracted from the SDF [S1].

### Data Structures and Network Layout
1.  **Geometry MLP ($f_g$):**
    *   **Input:** Encoded position or features of point $\mathbf{p}$ [S1].
    *   **Output:** SDF value $d_\Omega(\mathbf{p})$ and geometry feature vector $\mathbf{F}_g$ [S1].
    *   **Architecture:** 8 layers, hidden dimension 256, Softplus activation [S1].
2.  **SRDF MLP ($f_s$):**
    *   **Input:** Concatenation of view direction $\mathbf{r}$, geometry feature $\mathbf{F}_g$, and point position $\mathbf{p}$ [S1].
    *   **Output:** SRDF value $\tilde{d}_\Omega(\mathbf{p}, \mathbf{r})$ [S1].
    *   **Architecture:** 2 layers, intermediate output 256 channels, Softplus activation [S1].
3.  **Color MLP ($f_c$):**
    *   **Input:** View direction $\mathbf{r}$, geometry feature $\mathbf{F}_g$, point position $\mathbf{p}$, and 3D unit normal $\mathbf{n} = \nabla d_\Omega(\mathbf{p})$ [S1].
    *   **Output:** Color $\mathbf{c}$ [S1].
    *   **Architecture:** 2 layers, intermediate output 256 channels [S1].

### Definitions and Formulas
**SRDF Definition:**
The SRDF $\tilde{d}_\Omega(\mathbf{p}, \mathbf{r})$ is the signed distance from point $\mathbf{p}$ to the surface $\mathcal{M}$ along the ray direction $\mathbf{r}$ [S1]:
$$
\tilde{d}_\Omega(\mathbf{p}, \mathbf{r}) = (-1)^{\mathbf{1}_\Omega(\mathbf{p})} \min_{(\mathbf{p} + \rho \mathbf{r}) \in \mathcal{M} ; \rho \in \mathbb{R}} |\rho|
$$
where $\mathbf{1}_\Omega(\mathbf{p}) = 1$ if $\mathbf{p} \in \Omega$ (inside) and $0$ otherwise [S1].

**Volume Density from SRDF:**
The volume density $\sigma^{\text{SRDF}}(\mathbf{p}, \mathbf{r})$ is computed using the SRDF value [S1]:
$$
\sigma^{\text{SRDF}}(\mathbf{p}, \mathbf{r}) = \alpha \Psi_{\beta} \left( -\tilde{d}_\Omega(\mathbf{p}, \mathbf{r}) \right)
$$
where $\alpha$ and $\beta$ are learnable parameters, and $\Psi_{\beta}(s)$ is the Cumulative Distribution Function (CDF) of the Laplace distribution with zero mean and scale $\beta$ [S1]:
$$
\Psi_{\beta}(s) = \begin{cases} \frac{1}{2} \exp \left( \frac{s}{\beta} \right) & \text{if } s \leq 0 \\ 1 - \frac{1}{2} \exp \left( -\frac{s}{\beta} \right) & \text{if } s > 0 \end{cases}
$$
[S1]

**Standard Volume Rendering:**
Color is accumulated along the ray [S1]:
$$
\mathbf{C}(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}(\mathbf{p}_i, \mathbf{r})
$$
where transmittance $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ and opacity $\alpha_i = 1 - \exp(-\sigma(\mathbf{p}_i) \delta_i)$, with $\delta_i$ being the interval between neighboring points [S1].

**Rendered Depth and Normal:**
$$
D = \sum_{i=1}^N T_i \alpha_i z_i, \quad \mathbf{N} = \sum_{i=1}^N T_i \alpha_i \mathbf{n}_i
$$
where $\mathbf{n}_i$ is the normal at $\mathbf{p}_i$ [S1].

### Training Losses
The method minimizes a combination of losses [S1]:

1.  **Color Loss:** Standard photometric loss between rendered color and ground truth [S1].
2.  **Depth Loss:** Aligns rendered depth $D$ with ground truth $\bar{D}$ using scale $w$ and shift $q$ [S1]:
    $$
    \mathcal{L}_d = \frac{1}{N_s} \sum_{\mathbf{r} \in \mathcal{S}^2} \| (w D + q) - \bar{D} \|_2
    $$
    where $N_s$ is the number of rays [S1].
3.  **Normal Loss:** Aligns rendered normal $\mathbf{N}$ with ground truth $\bar{\mathbf{N}}$ (from pre-trained Omnidata) [S1]:
    $$
    \mathcal{L}_n = \frac{1}{N_s} \sum_{\mathbf{r} \in \mathcal{S}^2} (\| \mathbf{N} - \bar{\mathbf{N}} \|_1 + \| 1 - \mathbf{N}^T \bar{\mathbf{N}} \|_1)
    $$
    [S1]
4.  **Eikonal Loss:** Regularizes the gradient of the SDF [S1]:
    $$
    \mathcal{L}_e = \frac{1}{N_{ei}} \sum_{\mathbf{p} \in \mathcal{X}} (\| \nabla d_\Omega(\mathbf{p}) \|_2 - 1)^2
    $$
    where $N_{ei}$ is the number of sampled points [S1].
5.  **Smooth Loss:** Encourages smoothness of the SDF gradient [S1]:
    $$
    \mathcal{L}_s = \frac{1}{N_{sm}} \sum_{\mathbf{p} \in \mathcal{X}} \| \nabla d_\Omega(\mathbf{p}) - \nabla d_\Omega(\mathbf{p} + \varepsilon) \|_2
    $$
    where $\varepsilon$ is a small perturbation and $N_{sm}$ is the number of points [S1].
6.  **SRDF-SDF Consistency Loss:** Ensures sign consistency between SRDF and SDF [S1]. The hyperparameter $k$ is set to 12 [S1].
7.  **Self-Supervised Visibility Loss:** Distinguishes visible/occluded points using priors from both SDF and SRDF [S1].

### Parameters
*   **$\alpha, \beta$:** Learnable parameters in the density transformation [S1].
*   **$k$:** Hyperparameter in SRDF-SDF consistency loss, set to 12 [S1].
*   **Loss Weights:** $\lambda_d = 0.1$, $\lambda_n = 0.05$, $\lambda_e = 0.05$, $\lambda_s = 0.005$, $\lambda_{con} = 1.0$, $\lambda_{vis} = 0.001$ [S1].
*   **Learning Rates:** $5 \times 10^{-4}$ for geometry/color MLPs, $1 \times 10^{-5}$ for SRDF branch, $1 \times 10^{-2}$ for feature grids [S1].

### Algorithm Steps
1.  Sample $N$ points $\{\mathbf{p}_i = \mathbf{o} + z_i \mathbf{r}\}$ along camera ray $\mathbf{r}$ [S1].
2.  Input point $\mathbf{p}$ into geometry MLP $f_g$ to predict SDF $d_\Omega(\mathbf{p})$ and geometry feature $\mathbf{F}_g$ [S1].
3.  Input $\mathbf{p}$, view direction $\mathbf{r}$, and $\mathbf{F}_g$ into SRDF MLP $f_s$ to predict SRDF $\tilde{d}_\Omega(\mathbf{p}, \mathbf{r})$ [S1].
4.  Compute volume density $\sigma^{\text{SRDF}}(\mathbf{p}, \mathbf{r})$ using the formula above [S1].
5.  Compute color $\mathbf{c}$ using $f_c$ with inputs $\mathbf{r}, \mathbf{F}_g, \mathbf{p}, \mathbf{n}=\nabla d_\Omega(\mathbf{p})$ [S1].
6.  Render color $\mathbf{C}(\mathbf{r})$, depth $D$, and normal $\mathbf{N}$ using standard volume rendering equations with $\sigma^{\text{SRDF}}$ [S1].
7.  Compute losses (Color, Depth, Normal, Eikonal, Smooth, Consistency, Visibility) and update network weights [S1].
8.  **Post-Training:** Extract watertight mesh from $d_\Omega(\mathbf{p})$ using Marching Cubes [S1].

### Edge Cases and Pitfalls
*   **SDF to SRDF Conversion:** Generating SRDF directly from the predicted SDF (instead of a dedicated SRDF branch) yields inaccurate weights and false surfaces because the SDF prediction for a single point is influenced by all nearby objects [S1].
*   **SRDF without SDF:** Using only SRDF for density without an SDF branch leads to coarse surfaces and inaccurate absolute depth due to noise in pre-trained depth labels and scale inconsistency [S1].
*   **White Walls:** Standard SDF methods (like MonoSDF) generate high weights and negative SDF values near white walls, causing false surfaces [S1].
*   **Multi-modal Weights:** SDF-based methods produce weights with two local maxima along a ray, leading to inaccurate RGB values [S1]. SRDF produces uni-modal weights peaking only around the surface [S1].

## Operating characteristics
*   **Performance:** On ScanNet (MLP), it outperforms MonoSDF by 1.0% in recall, 9.8% in precision, and 6.1% in F-score [S1]. On Replica (Grid), it improves Chamfer distance by 9.91% and F-score by 2.88% [S1]. On Tanks and Temples, it achieves an F-score 1.15% higher (Grid) and 1.24% higher (MLP) than MonoSDF [S1].
*   **View Synthesis:** Achieves higher PSNR than MonoSDF due to consistent weights with actual observations [S1].
*   **Strengths:** Better reconstruction of thin/small objects (e.g., chair legs); uni-modal weight distribution; reduced noise in multi-object scenes [S1].
*   **Limitations:** Requires pre-trained models (Omnidata) for ground truth depth/normal supervision; sensitive to depth label noise [S1].
*   **Hardware:** Optimized on NVIDIA A100 GPU [S1].
*   **Complexity:** Uses MLPs (8 layers for geometry, 2 for SRDF/Color) or Grid representations (2 layers) [S1].

## Background
The method was proposed by Yin et al. in the paper "Ray-Distance Volume Rendering for Neural Scene Reconstruction" [S1]. It addresses limitations of VolSDF and NeuS, which use global SDFs for density, leading to ambiguities in indoor scenes with multiple objects [S1]. The method is applied to extensions of NeuS, such as NeuRIS, and builds on the concept of Ray-Specific Signed Distance Functions [S1].

## Sources
[S1] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
