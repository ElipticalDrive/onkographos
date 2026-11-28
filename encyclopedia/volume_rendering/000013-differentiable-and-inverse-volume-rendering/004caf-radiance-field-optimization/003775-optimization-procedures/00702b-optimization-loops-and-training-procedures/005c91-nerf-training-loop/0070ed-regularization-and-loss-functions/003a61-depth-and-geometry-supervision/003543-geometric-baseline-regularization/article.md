# Geometric Baseline Regularization

Also called: Geometric Baseline, Geometric Loss

## Overview
Geometric Baseline Regularization is a composite loss function designed to prevent degenerate solutions in Neural Radiance Field (NeRF) optimization, particularly in few-view settings where the network might collapse to copying training images [S1]. It serves as a strong baseline by combining photometric reconstruction with geometric priors that enforce compact depth distributions, full absorption, and multi-view consistency [S1]. Unlike sibling methods that may rely on external depth supervision or complex structural constraints, this method derives regularization directly from the volume rendering process [S1].

## How it works
The method optimizes the density $\sigma$ and color $\mathbf{c}$ fields by minimizing a total loss $\mathcal{L}_{\text{geom}}$ that combines a photometric term with three geometric regularizers [S1].

**Inputs and Outputs**
- **Inputs**: Training images $\mathcal{I}$, camera poses, ray origins $\mathbf{o}$ and directions $\mathbf{d}$, sample depths $t_i$ along rays, and hyperparameters $\lambda_{\text{fg}}, \lambda_{\text{fr}}, \lambda_{\text{dist}}$ [S1].
- **Outputs**: Updated network parameters $\theta = (\sigma, \mathbf{c})$ [S1].

**Data Structures**
- **Rendering Weights**: $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$, where $T_i = \prod_{j=1}^{i-1} (1 - \rho_j)$ is the accumulated transmittance, $\rho_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity, and $\delta_i = t_{i+1} - t_i$ [S1].
- **Expected Depth**: $\mathbf{D}(\mathbf{r}) = \frac{\sum_{i=1}^N w_i t_i}{\sum_{i=1}^N w_i}$ [S1].

**Formulas**
The total loss is defined as [S1]:
$$ \mathcal{L}_{\text{geom}} = \mathcal{L}_{\text{photo}} + \lambda_{\text{fg}} \mathcal{L}_{\text{fg}} + \lambda_{\text{fr}} \mathcal{L}_{\text{fr}} + \lambda_{\text{dist}} \mathcal{L}_{\text{dist}} $$

1.  **Photometric Loss**: L2 difference between input image $I_i$ and rendered color $\mathbf{C}_i$ [S1]:
    $$ \mathcal{L}_{\text{photo}}(\sigma, \mathbf{c}) = \sum_{i=1}^{\mathcal{I}} ||I_i - \mathbf{C}_i||_2 $$

2.  **Foreground Loss ($\mathcal{L}_{\text{fg}}$)**: Encourages weights to sum to unity, ensuring rays are fully absorbed by scene geometry [S1]:
    $$ \mathcal{L}_{\text{fg}} = \left( 1 - \sum_{i=1}^N w_i \right)^2 $$

3.  **Frustum Loss ($\mathcal{L}_{\text{fr}}$)**: Penalizes density placed in only one view frustum to prevent "image plane" collapse [S1]:
    $$ \mathcal{L}_{\text{fr}} = \sum_i w_i \mathbf{1}(n_i \le 1) $$
    where $n_i$ is the number of training view frustums containing the point $\mathbf{r}(t_i)$, and $\mathbf{1}(\cdot)$ is the indicator function [S1].

4.  **Distribution Loss ($\mathcal{L}_{\text{dist}}$)**: Regularizes weights to have a compact distribution, deviating from the original formulation by dividing by expected depth to increase strength for geometry close to the camera [S1]:
    $$ \mathcal{L}_{\text{dist}} = \frac{1}{\mathbf{D}(\mathbf{r})} \left( \sum_{i,j} w_i w_j \left| \frac{t_i + t_{i+1}}{2} - \frac{t_j + t_{j+1}}{2} \right| + \frac{1}{3} \sum_{i=1}^N w_i^2 (t_{i+1} - t_i) \right) $$
    *Note: The source text contains a typo using $w_k$ in the pairwise sum; it is corrected here to $w_j$ to match the summation indices [S1].*

**Parameters**
- $\lambda_{\text{fg}}, \lambda_{\text{fr}}$: Coefficients for foreground and frustum losses. Values not explicitly given in the sources [S1].
- $\lambda_{\text{dist}}$: Weight for the distribution regularizer. It is 0 for the first 3000 steps, then linearly increases to $1 \times 10^{-4}$ for the DTU dataset and $1.5 \times 10^{-5}$ for the LLFF dataset at 8000 steps [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometric_baseline_loss(rendered_colors, target_images, weights, depths, 
                            frustum_counts, lambda_fg, lambda_fr, lambda_dist, 
                            step, max_steps=8000, dataset='DTU'):
    """
    Computes the Geometric Baseline Loss.
    weights: list of lists, shape [num_rays, N]
    depths: list of lists, shape [num_rays, N+1] (N sample depths + ray end)
    frustum_counts: list of lists, shape [num_rays, N]
    """
    # 1. Photometric Loss
    photo_loss = 0.0
    for img, pred in zip(target_images, rendered_colors):
        photo_loss += sum((x - y) ** 2 for x, y in zip(img, pred)) ** 0.5

    # 2. Foreground Loss: Encourage sum of weights to be 1
    fg_loss = 0.0
    for w in weights:
        fg_loss += (1.0 - sum(w)) ** 2

    # 3. Frustum Loss: Penalize weights in only one frustum
    fr_loss = 0.0
    for w, n in zip(weights, frustum_counts):
        for w_i, count in zip(w, n):
            if count <= 1:
                fr_loss += w_i

    # 4. Distribution Loss
    dist_loss = 0.0
    for w, d in zip(weights, depths):
        numerator = sum(w_i * d_i for w_i, d_i in zip(w, d[:-1]))
        denominator = sum(w)
        # Avoid division by zero
        if denominator > 1e-8:
            D = numerator / denominator
        else:
            D = 1.0  # Fallback [G]

        if D < 1e-8:
            continue

        # Term 1: Sum over pairs i, j of |mid_i - mid_j| * w_i * w_j
        pair_sum = 0.0
        num_samples = len(w)
        for i in range(num_samples):
            mid_i = (d[i] + d[i + 1]) / 2.0
            for j in range(num_samples):
                mid_j = (d[j] + d[j + 1]) / 2.0
                pair_sum += w[i] * w[j] * abs(mid_i - mid_j)

        # Term 2: 1/3 * sum(w_i^2 * delta_i)
        delta_sum = 0.0
        for i in range(num_samples):
            delta = d[i + 1] - d[i]
            delta_sum += (w[i] ** 2) * delta

        ray_dist = (pair_sum + (1.0 / 3.0) * delta_sum) / D
        dist_loss += ray_dist

    # Schedule lambda_dist: starts at 3000 steps per source [S1]
    if step < 3000:
        current_lambda_dist = 0.0
    else:
        progress = (step - 3000) / (max_steps - 3000)
        max_val = 1e-4 if dataset == 'DTU' else 1.5e-5
        current_lambda_dist = progress * max_val

    total_loss = photo_loss + lambda_fg * fg_loss + lambda_fr * fr_loss + current_lambda_dist * dist_loss
    return total_loss
```

**Edge Cases and Pitfalls**
- **Division by Zero**: The distribution loss divides by expected depth $\mathbf{D}(\mathbf{r})$ [S1].
- If a ray has no absorption (sum of weights $\approx 0$), depth is undefined [G].
- **Few-View Collapse**: Without $\mathcal{L}_{\text{fr}}$, NeRFs may collapse to copying training images [S1].
- **Depth Scaling**: The division by $\mathbf{D}(\mathbf{r})$ in $\mathcal{L}_{\text{dist}}$ increases the regularizer's strength for geometry close to the camera, which is a deviation from the original formulation [S1].

## Operating characteristics
- **Time Complexity**: Not given in the sources [S1].
- **Memory Complexity**: Not given in the sources [S1].
- **Strengths**: Provides a robust baseline for few-view reconstruction without requiring external depth maps or point clouds [S1]. It effectively prevents degenerate solutions [S1].
- **Limitations**: The effectiveness depends on careful tuning of $\lambda_{\text{dist}}$ schedule [S1].

## Background
The Geometric Baseline Regularization is a loss combination used as a strong baseline for NeRF optimization, detailed in the DiffusioNeRF paper [S1]. It addresses degenerate solutions in few-view settings by regularizing the density and color fields through derived ray metrics [S1].

## Sources
[S1] DiffusioNeRF: Regularizing Neural Radiance Fields with Denoising Diffusion Models, https://doi.org/10.1109/cvpr52729.2023.00407, fetched 2026-10-26
