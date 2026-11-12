# Evaluation and Reconstruction Metrics


## Overview
This family comprises quantitative measures used to assess the fidelity, stability, and geometric correctness of volume rendering and inverse volume rendering outputs against ground truth or reference data. While the parent family [Regularization losses](1413-regularization-losses.md) focuses on optimizing scene parameters, these metrics evaluate the final rendered images or volumes. They distinguish themselves from sibling regularization techniques by serving as diagnostic or validation tools rather than direct optimization objectives, although some (like RMSE) can serve dual purposes. They address problems such as temporal flickering, structural fidelity on cut faces, and generalization error in learned volume representations.

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses).

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [S1, unofficial][S5, unofficial]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S1, unofficial][S5, unofficial].

Differentiation strategies fall into three main categories:
1.  **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [S5, unofficial].
2.  **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [S1, unofficial][S3].
3.  **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [S5, unofficial].

## Subtopics

### [Temporal Instability Metric](1921-temporal-instability-metric.md)
The Temporal Instability (TI) metric quantifies the temporal coherence of a volume rendering or novel view synthesis method by measuring the frame-to-frame variation in reconstruction error. It serves as a diagnostic for visual flickering artifacts, which are often imperceptible in static per-frame metrics like RMSE or PSNR. Unlike the feature-space Flicker metric, which compares temporal changes in feature representations, the TI metric operates directly in the froxel (voxel) space, measuring the variance of the error signal over time. It is distinct from structural similarity metrics as it focuses on signal stability rather than perceptual similarity.

### [Temporal Instability Metric](1921-temporal-instability-metric.md)
The Leak metric measures the fraction of a method's near-plane foreground energy that incorrectly falls on the culled side where the reference is background. It combines proxy reconstruction error, renderer mismatch, soft tails, and overshooting culls.

**Definition:**
The metric evaluates "leakage" of foreground energy into the background region on grazing views.

**Steps:**
1.  Compute method foreground mask $F(R)$ using $\theta_{fg} = 0.04$.
2.  Define near-plane region $N$ as pixels in $F(R)$ with $|s| \leq 60$, where $s$ is the signed pixel distance to the near-edge line proxy.
3.  Define leak region $L$ as pixels in $N$ with $s > 2$ that are not in reference foreground $F(G)$.
4.  Sum $\max_c R_c(\mathbf{x})$ over $L$ and $N$ across all grazing views, then take the ratio.

**Formula:**
$$
N = F(R) \cap \{|s| \leq 60\}, \quad L = N \cap \{s > 2\} \setminus F(G)
$$
$$
\text{Leak} = \frac{\sum_{\mathbf{x} \in \mathcal{L}} \max_{\mathbf{x}} R_c(\mathbf{x})}{\sum_{\mathbf{x} \in \mathcal{N}} \max_{\mathbf{x}} R_c(\mathbf{x})}
$$

**Parameters:**
- Foreground threshold $\theta_{fg} = 0.04$.
- Cut-edge margin $\delta = 2$ (for $s > 2$).
- Near-plane half-width $w = 60$ (for $|s| \leq 60$).

**Pitfalls:**
Averaging per-view ratios on thin grazing bands causes numerical instability. Pooling sums across all views before taking the ratio stabilizes the metric.

### [Temporal Instability Metric](1921-temporal-instability-metric.md)
The Band Fidelity Metric ($S_b$) measures the structural similarity between a method's render and a reference render over the exposed cut face (reference foreground).

**Definition:**
It assesses how well the method preserves the structure of the reference foreground on perpendicular views.

**Steps:**
1.  Compute scalar luminance $Y(x) = \frac{1}{3} \sum_c I_c(x) \in [0, 1]$ at each pixel $x$.
2.  Compute foreground mask $F(I) = \{x : \max_c I_c(x) > \theta_{fg}\}$ with $\theta_{fg} = 0.04$.
3.  Compute SSIM between method render and reference restricted to $F(G)$.
4.  Average the SSIM values across all perpendicular views of all planes in the evaluated set.

**Parameters:**
- Foreground threshold $\theta_{fg} = 0.04$.
- Band $\mathcal{B}$ for $S_b$ is the reference foreground $F(G)$.

**Pitfalls:**
Band PSNR is omitted from main tables because operator differences are only hundredths of a decibel and leakage can offset small brightness bias.

### [Temporal Instability Metric](1921-temporal-instability-metric.md)
This validation procedure assesses model generalization on a hold-out set of 2,000 volume-rendered images not used for training.

**Definition:**
- **RMSE (Root Mean Squared Error):** Used as the primary evaluation metric for image similarity.
- **EMD (Earth Mover’s Distance):** Used to measure higher-level similarity between color histograms, mitigating misalignment in histogram space.

**Parameters:**
- EMD Cost Normalization: The cost between histogram bins is normalized such that the maximum cost is 1.

**Performance Data:**
- Training Time: 16.5 hours.
- Rendering Time: 0.007 seconds per image.
- TF Exploration Time: 0.06 seconds.
- TF Sensitivity Time: 0.49 seconds.

## Background
The evaluation of volume rendering has evolved from simple pixel-wise error metrics to more complex structural and temporal measures. Early work in learned volume rendering utilized RMSE and EMD to validate generalization on hold-out sets, noting that RMSE alone may fail to capture higher-level statistics and is sensitive to pose errors [S2]. More recent developments in differentiable volume rendering and Gaussian Splatting have introduced metrics tailored to specific artifacts, such as temporal instability in novel view synthesis [S1, unofficial] and leakage errors in half-space clipping methods [S1, unofficial]. These metrics often operate on scalar luminance and use foreground masks to isolate regions of interest, such as cut faces or near-plane regions, to provide more meaningful assessments of rendering quality [S1, unofficial].

## Sources
[S1] XClipGS: Exact Half-Space Clipping for Medical Volume Gaussian Splatting, https://doi.org/10.48550/arxiv.2608.07760, fetched 2026-10-26 (unofficial)
[S2] A Generative Model for Volume Rendering, https://doi.org/10.1109/tvcg.2018.2816059, fetched 2026-10-26
