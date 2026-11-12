# Evaluation and Reconstruction Metrics


## Overview
This family provides quantitative measures to assess the geometric accuracy, temporal stability, and visual fidelity of differentiable volume rendering and radiance field optimizations. Unlike sibling regularization losses that guide the training process, these metrics are primarily used for post-hoc evaluation or as specific diagnostic objectives. They distinguish themselves by targeting specific failure modes of volume rendering: [Temporal Instability Metric](1921-temporal-instability-metric.md) detects flickering in dynamic sequences, [Inverse Mean Residual Color (IMRC) Metric](10556-inverse-mean-residual-color-imrc-metric.md) evaluates geometry without ground-truth meshes, [Across View-direction Inconsistency (AVI) Metric](11106-across-view-direction-inconsistency-avi-metric.md) penalizes spatial-temporal discontinuities between adjacent views, and [Ray Diversity Objective](13003-ray-diversity-objective.md) guides active view selection.

## Shared principles
This family does not share a single mathematical mechanism, as the metrics operate on different domains (time, geometry, view-space). However, they all rely on the differentiability of the forward rendering operator to compute gradients with respect to scene parameters if used as losses, or on the comparison of rendered outputs against reference data (ground truth, proxy geometry, or consistency constraints) [44-radiance-field-optimization.md]. The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [44-radiance-field-optimization.md]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [44-radiance-field-optimization.md].

Differentiation strategies fall into three main categories [44-radiance-field-optimization.md]:
1. **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [44-radiance-field-optimization.md].
2. **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [44-radiance-field-optimization.md].
3. **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [44-radiance-field-optimization.md].

## Subtopics

### [Temporal Instability Metric](1921-temporal-instability-metric.md)
The Temporal Instability (TI) metric quantifies temporal coherence in volume rendering or novel view synthesis by measuring frame-to-frame variations in reconstruction error. It serves as a diagnostic for visual flickering artifacts that static per-frame metrics like RMSE may miss. Lower values indicate smoother temporal consistency. Choose this metric when evaluating dynamic reconstructions or novel view synthesis sequences where temporal flickering is a primary concern.

### [Inverse Mean Residual Color (IMRC) Metric](10556-inverse-mean-residual-color-imrc-metric.md)
The Inverse Mean Residual Color (IMRC) metric evaluates the geometric quality of a radiance field without requiring ground-truth geometry, by measuring the transmittance-density-weighted residual color in decibels. Unlike PSNR, which assesses overall image quality, or Chamfer Distance (CD), which requires ground-truth point clouds and suffers from information loss by considering only a single iso-surface, IMRC evaluates the entire density field and is more consistent with human user rankings for geometry. Choose this metric when ground-truth geometry is unavailable but a geometric quality assessment is needed.

### [Across View-direction Inconsistency (AVI) Metric](11106-across-view-direction-inconsistency-avi-metric.md)
The Across View-direction Inconsistency (AVI) metric quantifies the degree of geometrical inconsistency between every pair of test frames corresponding to adjacent viewing directions in a rendered sequence. It is designed to evaluate the temporal and spatial coherence of volume rendering methods, particularly in the context of super-resolution, by measuring how well adjacent views align when warped to one another. Unlike standard image quality metrics such as PSNR or SSIM which evaluate individual frames in isolation, AVI explicitly penalizes artifacts like ghosting and jitter that arise from inconsistent geometry across the view space. Choose this metric when evaluating the consistency of multi-view reconstructions, especially for super-resolution or novel view synthesis.

### [Ray Diversity Objective](13003-ray-diversity-objective.md)
The Ray Diversity Objective quantifies the variation of rays projected from various viewing poses observing a particular face of an object mesh, aiming to yield better reconstruction by ensuring highly varied novel views. It sets itself apart from sibling metrics by explicitly combining a geometric diversity measure (variance of ray principal axes) with a quality penalty (outlier score based on grazing angles) to guide active view planning. Choose this metric when planning camera poses for active reconstruction or when evaluating the diversity of sampling in a scene.

#### Leak Metric
- **Definition**: Measures the fraction of a method's near-plane foreground energy that incorrectly falls on the culled side where the reference is background. Combines proxy reconstruction error, renderer mismatch, soft tails, and overshooting culls [S1, unofficial].
- **Step**: 1. Compute method foreground mask $F(R)$ using $\theta_{fg} = 0.04$. 2. Define near-plane region $N$ as pixels in $F(R)$ with $|s| \leq 60$. 3. Define leak region $L$ as pixels in $N$ with $s > 2$ that are not in reference foreground $F(G)$. 4. Sum $\max_c R_c(\mathbf{x})$ over $L$ and $N$ across all grazing views, then take ratio [S1, unofficial].
- **Formula**: $N = F(R) \cap \{|s| \leq 60\}$, $L = N \cap \{s > 2\} \setminus F(G)$. $\text{Leak} = \frac{\sum_{\mathbf{x} \in \mathcal{L}} \max_{\mathbf{x}} R_c(\mathbf{x})}{\sum_{\mathbf{x} \in \mathcal{N}} \max_{\mathbf{x}} R_c(\mathbf{x})}$ [S1, unofficial].
- **Parameter**: Foreground threshold $\theta_{fg} = 0.04$. Cut-edge margin $\delta = 2$ (for $s > 2$). Near-plane half-width $w = 60$ (for $|s| \leq 60$) [S1, unofficial].
- **Pitfall**: Averaging per-view ratios on thin grazing bands causes numerical instability. Pooling sums across all views before taking the ratio stabilizes the metric [S1, unofficial].

#### Band Fidelity Metric (Sb)
- **Definition**: A fidelity metric measuring the structural similarity between a method's render and a reference render over the exposed cut face (reference foreground) [S1, unofficial].
- **Step**: Compute scalar luminance $Y(x) = \frac{1}{3} \sum_c I_c(x) \in [0, 1]$ at each pixel $x$. Compute foreground mask $F(I) = \{x : \max_c I_c(x) > \theta_{fg}\}$ with $\theta_{fg} = 0.04$. Compute SSIM between method render and reference restricted to $F(G)$. Average the SSIM values across all perpendicular views of all planes in the evaluated set [S1, unofficial].
- **Parameter**: Foreground threshold $\theta_{fg} = 0.04$. Band $\mathcal{B}$ for $S_b$ is the reference foreground $F(G)$ [S1, unofficial].
- **Pitfall**: Band PSNR is omitted from main tables because operator differences are only hundredths of a decibel and leakage can offset small brightness bias [S1, unofficial].

#### Model Validation using RMSE and EMD
- **Definition**: A validation procedure assessing model generalization on a hold-out set of 2,000 volume-rendered images not used for training [S2]. EMD (Earth Mover’s Distance) is used to measure higher-level similarity between color histograms, mitigating misalignment in histogram space [S2].
- **Parameter**: RMSE (Root Mean Squared Error) is used as the primary evaluation metric for image similarity [S2]. EMD Cost Normalization: The cost between histogram bins is normalized such that the maximum cost is 1 [S2].
- **Number**: Training Time: 16.5 hours. Rendering Time: 0.007 seconds per image. TF Exploration Time: 0.06 seconds. TF Sensitivity Time: 0.49 seconds [S2].

## Background
The evaluation of volume rendering and radiance fields has evolved from simple pixel-wise error metrics to more sophisticated measures that account for geometric consistency, temporal stability, and perceptual quality. Early methods relied heavily on PSNR and SSIM, which evaluate individual frames in isolation [11106-across-view-direction-inconsistency-avi-metric.md]. As differentiable rendering became prevalent, metrics like IMRC were introduced to evaluate geometry without ground-truth meshes, addressing the limitations of Chamfer Distance [10556-inverse-mean-residual-color-imrc-metric.md]. For dynamic scenes, Temporal Instability was proposed to detect flickering that static metrics miss [1921-temporal-instability-metric.md]. In the context of active view planning and Gaussian Splatting, metrics like Ray Diversity and Leak/Band Fidelity were developed to ensure robust reconstruction and accurate clipping [13003-ray-diversity-objective.md][S1, unofficial]. These metrics are widely used in medical volume visualization, novel view synthesis, and 3D Gaussian Splatting optimization [S1, unofficial][S2].

## Sources
[S1] XClipGS: Exact Half-Space Clipping for Medical Volume Gaussian Splatting, https://doi.org/10.48550/arxiv.2608.07760, fetched 2026-10-26 (unofficial)
[S2] A Generative Model for Volume Rendering, https://doi.org/10.1109/tvcg.2018.2816059, fetched 2026-10-26
