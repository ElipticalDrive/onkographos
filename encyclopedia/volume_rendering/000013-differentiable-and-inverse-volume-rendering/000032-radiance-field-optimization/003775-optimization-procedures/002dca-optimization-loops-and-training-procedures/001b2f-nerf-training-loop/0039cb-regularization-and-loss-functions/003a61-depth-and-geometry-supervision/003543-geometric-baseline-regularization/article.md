# Geometric Baseline Regularization

Also called: Geometric Baseline, Geometric Loss

## Overview
Geometric Baseline Regularization is a composite loss function designed to prevent degenerate solutions in Neural Radiance Field (NeRF) optimization, particularly in few-view settings where networks may collapse to reproducing training images [S1]. It combines photometric reconstruction with three geometric priors: foreground absorption, frustum consistency, and depth distribution compactness [S1]. It differs from sibling methods like [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by relying on internal geometric consistency priors rather than external depth map supervision, and from [Depth Structural Consistency Regularization](9371-depth-structural-consistency-regularization.md) by using explicit frustum intersection counts rather than structural edge preservation [S1].

## How it works
The method optimizes the density $\sigma$ and color $\mathbf{c}$ fields by minimizing a total loss $\mathcal{L}_{\text{geom}}$ that integrates photometric accuracy with geometric constraints [S1].

**Inputs and Outputs**
*   **Inputs**: Training images $\mathcal{I}$, camera poses, ray origins $\mathbf{o}$ and directions $\mathbf{r}$, sample points $t_i$ along rays, and the number of training views $N_{\text{views}}$.
*   **Outputs**: Updated network parameters $\theta$ (defining $\sigma$ and $\mathbf{c}$).

**Data Structures**
*   **Ray Samples**: For each ray $\mathbf{r}$, a list of $N$ samples. Each sample $i$ has:
    *   $t_i$: Distance from camera center.
    *   $\sigma_i$: Predicted density at $t_i$.
    *   $\mathbf{c}_i$: Predicted color at $t_i$.
    *   $\delta_i = t_{i+1} - t_i$: Interval length.
*   **Rendering Weights**: $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$, where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance [S1].
*   **Frustum Counts**: $n_i$, the number of training view frustums containing the point at $t_i$ [S1].

**Formulas**
The total loss is defined as [S1]:
$$ \mathcal{L}_{\text{geom}} = \mathcal{L}_{\text{photo}} + \lambda_{\text{fg}} \mathcal{L}_{\text{fg}} + \lambda_{\text{fr}} \mathcal{L}_{\text{fr}} + \lambda_{\text{dist}} \mathcal{L}_{\text{dist}} $$

1.  **Photometric Loss** $\mathcal{L}_{\text{photo}}$: The L2 difference between rendered colors $\mathbf{C}_i$ and input images $I_i$ [S1]:
    $$ \mathcal{L}_{\text{photo}}(\sigma, \mathbf{c}) = \sum_{i=1}^{\mathcal{I}} ||I_i - \mathbf{C}_i||_2 $$
    where $\mathbf{C}_i = \sum_{j=1}^N w_j \mathbf{c}_j + c_b (1 - \sum_{j=1}^N w_j)$ and $c_b$ is the background color (typically white) [S1].

2.  **Foreground Loss** $\mathcal{L}_{\text{fg}}$: Encourages rays to be fully absorbed by scene geometry (weights sum to unity) [S1]:
    $$ \mathcal{L}_{\text{fg}} = \left( 1 - \sum_{i=1}^N w_i \right)^2 $$

3.  **Frustum Loss** $\mathcal{L}_{\text{fr}}$: Penalizes density placed in only one view frustum to prevent "image plane" collapse [S1]:
    $$ \mathcal{L}_{\text{fr}} = \sum_{i=1}^N w_i \mathbf{1}(n_i \le 1) $$
    where $\mathbf{1}(\cdot)$ is the indicator function and $n_i$ is the count of training frustums containing the point at $t_i$ [S1].

4.  **Distribution Loss** $\mathcal{L}_{\text{dist}}$: Regularizes the rendering weights to have a compact distribution around the expected depth [S1]:
    $$ \mathcal{L}_{\text{dist}} = \frac{1}{\mathbf{D}(\mathbf{r})} \left( \sum_{i,j} w_i w_j \left| \frac{t_i + t_{i+1}}{2} - \frac{t_j + t_{j+1}}{2} \right| + \frac{1}{3} \sum_{i=1}^N w_i^2 (t_{i+1} - t_i) \right) $$
    where $\mathbf{D}(\mathbf{r})$ is the expected depth [S1]:
    $$ \mathbf{D}(\mathbf{r}) = \frac{\sum_{i=1}^N w_i t_i}{\sum_{i=1}^N w_i} $$
    Note: The source deviates from the original formulation by dividing by $\mathbf{D}(\mathbf{r})$ to increase regularization strength for geometry close to the camera [S1].

**Parameters**
*   $\lambda_{\text{fg}}$: Coefficient for foreground loss. [G] Typically set to 1.0 or omitted if not explicitly tuned, as the source emphasizes $\lambda_{\text{dist}}$ scheduling.
*   $\lambda_{\text{fr}}$: Coefficient for frustum loss. [G] Typically set to 1.0 or omitted if not explicitly tuned.
*   $\lambda_{\text{dist}}$: Coefficient for distribution loss. Scheduled as follows [S1]:
    *   Steps $0$ to $2500$: $0$.
    *   Steps $2500$ to $8000$: Linearly increases from $0$ to max value.
    *   Max Value: $1 \times 10^{-4}$ for DTU dataset, $1.5 \times 10^{-5}$ for LLFF dataset.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometric_baseline_loss(rendered_colors, input_images, weights, t, deltas, n_frustums, lambda_dist_schedule, step):
    """
    Computes the Geometric Baseline Loss.
    
    Args:
        rendered_colors: Tensor of shape (num_rays, 3), rendered RGB values.
        input_images: Tensor of shape (num_rays, 3), ground truth RGB values.
        weights: Tensor of shape (num_rays, num_samples), rendering weights w_i.
        t: Tensor of shape (num_rays, num_samples), sample distances t_i.
        deltas: Tensor of shape (num_rays, num_samples), interval lengths delta_i.
        n_frustums: Tensor of shape (num_rays, num_samples), count of frustums n_i.
        lambda_dist_schedule: Object providing lambda_dist value for current step.
        step: Current training step integer.
        
    Returns:
        scalar loss value.
    """
    # 1. Photometric Loss (L2)
    # Note: Source uses L2 norm sum, typically mean squared error is used in practice.
    # We follow source definition: sum of L2 norms.
    photo_loss = torch.sum(torch.norm(rendered_colors - input_images, dim=1))
    
    # 2. Foreground Loss
    # Sum of weights along each ray
    weight_sum = torch.sum(weights, dim=1) # Shape: (num_rays,)
    # (1 - sum(w_i))^2
    fg_loss = torch.mean(torch.pow(1.0 - weight_sum, 2))
    
    # 3. Frustum Loss
    # Indicator: 1 if n_i <= 1, else 0
    frustum_mask = (n_frustums <= 1).float()
    # Sum of w_i * indicator
    fr_loss = torch.sum(weights * frustum_mask)
    
    # 4. Distribution Loss
    # Compute expected depth D(r)
    # D(r) = sum(w_i * t_i) / sum(w_i)
    numerator_depth = torch.sum(weights * t, dim=1)
    denominator_depth = torch.sum(weights, dim=1)
    # Avoid division by zero
    denominator_depth = torch.clamp(denominator_depth, min=1e-8)
    expected_depth = numerator_depth / denominator_depth # Shape: (num_rays,)
    
    # Compute midpoints
    # t_{i+1} is not directly available, assume t includes boundaries or use t + delta/2
    # Source uses (t_i + t_{i+1})/2. If t_i are sample centers, t_{i+1} is next sample.
    # If t_i are boundaries, t_{i+1} is next boundary.
    # Standard NeRF sampling: t_i are boundaries. Midpoint = (t_i + t_{i+1})/2.
    # Let's assume t is boundaries.
    t_next = torch.cat([t[:, 1:], t[:, -1:] + deltas[:, -1:]], dim=1) # Approximation if t is centers
    # If t is centers, midpoint is t. If t is boundaries, midpoint is (t_i + t_{i+1})/2.
    # Source formula: | (t_i + t_{i+1})/2 - (t_j + t_{j+1})/2 |.
    # We compute midpoints m_i = (t_i + t_{i+1}) / 2.
    # Assuming t is sorted boundaries.
    m = (t + t_next) / 2.0
    
    # Term 1: Sum over i,j of w_i w_j |m_i - m_j|
    # This is computationally expensive O(N^2). 
    # Efficient implementation: E[|X - Y|] for distribution w.
    # However, we implement directly as per source structure for clarity.
    # Reshape for broadcasting: w (R, S, 1), m (R, S, 1)
    w_expanded = weights.unsqueeze(2) # (R, S, 1)
    m_expanded = m.unsqueeze(1)       # (R, 1, S)
    w_j = weights.unsqueeze(1)        # (R, 1, S)
    m_j = m.unsqueeze(2)              # (R, S, 1)
    
    diff = torch.abs(m_expanded - m_j)
    term1 = torch.sum(w_expanded * w_j * diff)
    
    # Term 2: (1/3) * sum(w_i^2 * delta_i)
    term2 = (1.0 / 3.0) * torch.sum(weights**2 * deltas)
    
    # Combine terms
    dist_loss_raw = term1 + term2
    
    # Divide by expected depth D(r)
    # Source: 1/D(r) * ( ... )
    # This implies per-ray division, then sum or mean?
    # Source formula shows D(r) in denominator outside sum.
    # Usually implies sum over rays of (term / D(r)).
    dist_loss_per_ray = dist_loss_raw / expected_depth
    dist_loss = torch.sum(dist_loss_per_ray)
    
    # Get lambda_dist
    lambda_dist = lambda_dist_schedule.get_value(step)
    
    # Total Loss
    # lambda_fg and lambda_fr are not scheduled in source, assume 1.0 or provided
    lambda_fg = 1.0
    lambda_fr = 1.0
    
    total_loss = photo_loss + lambda_fg * fg_loss + lambda_fr * fr_loss + lambda_dist * dist_loss
    
    return total_loss
```

**Edge Cases and Pitfalls**
*   **Division by Zero**: The expected depth $\mathbf{D}(\mathbf{r})$ in the denominator of $\mathcal{L}_{\text{dist}}$ can be zero if no density is rendered. A small epsilon should be added to the denominator [G].
*   **Few-View Collapse**: Without $\mathcal{L}_{\text{fr}}$, NeRFs may collapse to "image planes" where each camera sees only its own training image. This loss specifically penalizes density contained in only one frustum [S1].
*   **Depth Scaling**: The division by $\mathbf{D}(\mathbf{r})$ in $\mathcal{L}_{\text{dist}}$ increases regularization strength for geometry close to the camera. This is a deliberate deviation from the original formulation [S1].

## Operating characteristics
*   **Time Complexity**: The distribution loss $\mathcal{L}_{\text{dist}}$ involves a double sum over samples ($\sum_{i,j}$), leading to $O(N^2)$ complexity per ray if implemented naively. This can be a bottleneck for large $N$ [S1].
*   **Memory Complexity**: Requires storing weights, depths, and frustum counts for all samples. The double sum in $\mathcal{L}_{\text{dist}}$ may require $O(N^2)$ temporary memory if not optimized [S1].
*   **Performance**: Provides a "very strong baseline" for NeRF optimization, significantly improving geometry in few-view settings compared to photometric loss alone [S1].
*   **Strengths**: Prevents degenerate solutions without requiring external depth supervision or multi-view stereo priors [S1].
*   **Limitations**: Does not enforce global geometric consistency (e.g., smoothness) as effectively as methods using external depth maps. The $O(N^2)$ term can be slow [S1].
*   **When to use**: Few-view reconstruction, scenes with large empty spaces, or when external depth data is unavailable [S1].
*   **When not to use**: When high-fidelity geometry is required and external depth maps are available (use [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) instead), or when computational resources are extremely limited due to the $O(N^2)$ term [S1].

## Background
The Geometric Baseline Regularization was introduced in the context of regularizing NeRFs with Denoising Diffusion Models, specifically in the paper "DiffusioNeRF" [S1]. It serves as a foundational regularization strategy that was later used as a baseline for comparing diffusion-based regularization techniques [S1]. The method addresses the well-known issue of NeRFs collapsing to degenerate solutions in sparse view settings [S1].

## Sources
[S1] DiffusioNeRF: Regularizing Neural Radiance Fields with Denoising Diffusion Models, https://doi.org/10.1109/cvpr52729.2023.00407, fetched 2026-10-26
