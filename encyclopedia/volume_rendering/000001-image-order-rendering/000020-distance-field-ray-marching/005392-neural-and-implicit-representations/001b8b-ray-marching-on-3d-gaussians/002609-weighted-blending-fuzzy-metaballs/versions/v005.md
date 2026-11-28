# Weighted Blending (Fuzzy Metaballs)
Also called: Blending intersections, Fuzzy Metaballs rendering, Hidden-surface solution for Fuzzy Metaballs, Sort-free Gaussian rendering, Weighted Blending of Intersections

## Overview
Weighted Blending is a sort-free, image-order rendering technique for 3D Gaussians that resolves the hidden-surface problem by computing a weighted average of all ray-Gaussian intersection depths [S1, unofficial][S2]. Unlike NeRF-style methods that sort intersections and use closer Gaussians to attenuate further ones, this method blends all intersections simultaneously without sorting, extending Order Independent Transparency (OIT) principles to 3D opaque surfaces [S1, unofficial][S2].

## How it works
**Inputs:** Ray direction $\vec{v}$ (normalized), set of $N$ 3D Gaussians [S1, unofficial].
**Outputs:** Blended intersection depth $t_f$, opacity $\alpha$, and optionally normal $\vec{n}_f$ or color $\vec{c}_f$ [S1, unofficial].

**Data Structures:**
- Gaussian $i$: $\mu_i$ (vector), $\Sigma_i^{-1}$ (matrix), $\lambda_i$ (scalar) [S1, unofficial].
- Per-ray accumulators: $t_f$ (scalar), $\alpha$ (scalar), $\vec{n}_f$ (vector), $\vec{c}_f$ (vector) [S1, unofficial].

**Steps:**
1. For each Gaussian $i$, compute the intersection distance $t_i$ along the ray $\vec{v}$ using the linear approach (point of maximum likelihood) [S1, unofficial]:
   $$ t_i = \frac{\mu_i^T \Sigma_i^{-1} \vec{v}}{\vec{v}^T \Sigma_i^{-1} \vec{v}} $$
2. Compute the unnormalized log distance $d_i$ for Gaussian $i$ [S1, unofficial]:
   $$ d_i = -\frac{1}{2} \left[ (t_i \vec{v} - \mu_i)^T \Sigma_i^{-1} (t_i \vec{v} - \mu_i) \right] + \log(\lambda_i) $$
3. Compute the unnormalized Gaussian density $\delta_i$ [S1, unofficial]:
   $$ \delta_i = \exp(d_i) $$
4. Compute the weight $w_i$ for each intersection [S1, unofficial][S2]:
   $$ w_i = \exp \left( \beta_1 d_i \sigma \left( \frac{\beta_3}{\eta} t_i \right) - \frac{\beta_2}{\eta} t_i \right) $$
   where $\sigma(x) = \frac{1}{1 + e^{-x}}$ is the sigmoid function [S1, unofficial]. The term $h(t_i) = \sigma \left( \frac{\beta_3}{\eta} t_i \right)$ down-weights intersections behind the camera [S2].
5. Compute the final blended depth $t_f$ [S1, unofficial][S2]:
   $$ t_f = \frac{1}{\sum_i w_i} \sum_i w_i t_i $$
6. Compute opacity $\alpha$ [S1, unofficial]:
   $$ \alpha = \sigma \left( \beta_4 \sum_i \delta_i + \beta_5 \right) $$
7. For other properties like normal $\vec{n}_i$ or color $\vec{c}_i$, apply the same weighted average using $w_i$ [S1, unofficial].

**Parameters:**
- $\beta_1, \beta_2$: Balance high-quality hits (high $d_i$) vs. camera proximity (low $t_i$) [S1, unofficial][S2].
- $\beta_3$: Controls sigmoid decay for behind-camera intersections [S2].
- $\beta_4, \beta_5$: Control opacity scaling and offset [S1, unofficial].
- $\eta$: Rough scale of the object, used to normalize $t_i$ for scale invariance [S1, unofficial][S2].
- Typical values: Not given in the sources [S2].

**Edge Cases and Pitfalls:**
- Intersections behind the camera ($t_i < 0$) are down-weighted by the sigmoid term in the original model [S2].
- The original formulation assumes $\sum \lambda_i = 1$ [S1, unofficial]. Losing this global normalization causes minor differences in pose estimation but is negligible for shape estimation [S1, unofficial].
- Precision issues may arise if $\lambda_i$ is very small, causing $\log(\lambda_i)$ to underflow [G]. Ensure $\lambda_i$ is bounded away from zero or use log-sum-exp tricks if necessary [G].

## Subtopics

### [Fuzzy Metaballs](7316-fuzzy-metaballs.md)
Fuzzy Metaballs is an algebraic, implicit surface representation that defines geometry as the zero level set of a function composed of multidimensional Gaussians (ellipsoids) rather than isotropic ones. It serves as the foundational object representation for differentiable rendering pipelines, enabling robust optimization by relaxing hard thresholds to ensure smooth gradients during ray-tracing. Unlike Weighted Blending, which focuses on sort-free image-order rendering, Fuzzy Metaballs provides the underlying implicit field definition that allows for the computation of surface normals and distance-based losses.

### Two Parameter Model
A simplified rendering model for shape reconstruction that reduces the original 5 hyperparameters to 2 ($\beta_1, \beta_2$) by removing the sigmoid term for behind-camera contributions and replacing the sigmoid alpha function with a decaying exponential [S1, unofficial].
- **Weight:** $w_i = \exp \left( \beta_1 d_i - \frac{\beta_2}{\eta} t_i \right)$ [S1, unofficial].
- **Alpha:** $\alpha = 1 - \exp \left( - \sum_i \delta_i \right)$ [S1, unofficial].
- **Pitfall:** Losing the global normalization condition ($\sum \lambda_i = 1$) causes minor differences in pose estimation, but these are negligible for shape estimation [S1, unofficial].

### Zero Parameter Model
A hyperparameter-free rendering variant that requires sorting intersections and uses alpha compositing (transmission) to compute weights, enabling differentiable rendering without tunable parameters [S1, unofficial].
- **Step:** Sort all ray-Gaussian intersections by depth $t_i$. Compute accumulated transmission $T_i$ for each intersection based on earlier (closer) intersections [S1, unofficial].
- **Transmission:** $T_i = \exp \left( -\sum_j \delta_j 1[t_j < t_i] \right)$, where $1[\cdot]$ is the indicator function [S1, unofficial].
- **Weight:** $w_i = T_i (1 - \exp(-\delta_i))$ [S1, unofficial].
- **Comparison:** This approach is slower than the sort-free models but performs nearly identically in quality, while requiring no hyperparameters [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ evaluations of Gaussians for each ray, where $N$ is the number of Gaussians [S2].
- **Memory Complexity:** Not given in the sources [S2].
- **Measured Performance:** On a Quadro P2000 (CUDA), Fuzzy Metaballs forward pass takes $3.0 \pm 0.2$ ms and backward pass $9.6 \pm 0.5$ ms for 160x120 images [S2]. On an i5-7287U (CPU), forward pass takes $3.0 \pm 0.15$ ms and backward pass $13.2 \pm 1.4$ ms for 80x60 images [S2].
- **Strengths:** Sort-free, differentiable, scale-invariant (with $\eta$), simple implementation (60 lines of JAX code) [S2].
- **Limitations:** The original model requires tuning 5 hyperparameters [S1, unofficial]. The Zero Parameter Model is slower due to sorting [S1, unofficial].
- **When to use:** When differentiable rendering of 3D Gaussians is required without the overhead of sorting, particularly for shape reconstruction [S1, unofficial][S2].

## Background
Originated from the Fuzzy Metaballs paper [S2] and further detailed in "Flexible Techniques for Differentiable Rendering with 3D Gaussians" [S1, unofficial]. The method was developed to provide a sort-free solution to the hidden-surface problem for 3D Gaussian representations, extending Order Independent Transparency (OIT) to opaque surfaces [S2]. It is used in differentiable rendering pipelines for shape reconstruction from video [S1, unofficial].

## Sources
[S1] Flexible Techniques for Differentiable Rendering with 3D Gaussians, https://doi.org/10.48550/arxiv.2308.14737, fetched 2026-10-26 (unofficial)
[S2] Approximate Differentiable Rendering with Algebraic Surfaces, https://doi.org/10.1007/978-3-031-19824-3_35, fetched 2026-10-26
