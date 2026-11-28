# Superpixel-guide Efficient Sampling (SES)


## Overview
Superpixel-guide Efficient Sampling (SES) is a training strategy for Neural Radiance Fields (NeRF) and neural fields that uses SLIC superpixels as the smallest processing units to guide ray sampling [S1]. It addresses the slow convergence and scattered sampling issues in high-resolution NeRF training by focusing computational resources on complex regions with high rendering errors [S1]. Unlike quadtree-based methods which suffer from redundant computations and fragmented information due to fixed geometric shapes, SES resolves inadequate partitioning issues and reduces computational overhead per instance by adapting to image content [S1]. The method builds on [Adaptive Ray Sampling](14469-adaptive-ray-sampling.md) by replacing fixed hierarchical partitions with content-aware superpixel masks [S1].

## How it works
SES operates in an iterative loop of random sampling, error recording, secondary segmentation, and guided sampling [S1]. It integrates a superpixel loss term into the standard optimization process [S1].

### Inputs and Outputs
*   **Inputs:** Input RGB images $I$, initial neural field parameters (e.g., Instant-NGP or DVGO), hyperparameters $\beta_0, \lambda_{mse}, \lambda_{sp}, t, \lambda_{error}, \lambda_{std}$ [S1].
*   **Outputs:** Optimized neural field parameters, rendered images [S1].

### Data Structures
*   **Superpixel Map:** A 2D array where each element $(u,v)$ stores the superpixel ID $k$ assigned to that pixel [S1].
*   **Error Map:** A 2D array storing the per-pixel rendering error $r_j$ for the most recent training batch [S1].
*   **Sampling Probability Map:** A 2D array storing the probability $P(u,v)$ of selecting pixel $(u,v)$ for the next ray batch [S1].

### Parameters
*   $\beta_0$: Weight between RGB image and render error in secondary segmentation. Synthetic datasets: $0.8$. Real datasets: $0.65$ [S1].
*   $\lambda_{sp}$: Coefficient for superpixel loss. Synthetic datasets: $10^{-6}$. Real datasets: $10^{-4}$ [S1].
*   $\lambda_{mse}$: Coefficient for MSE loss. Typically $1$ [S1].
*   $t$: Minimum error threshold to exclude negligible errors (blank/well-learned regions) [S1].
*   $\lambda_{error}, \lambda_{std}$: Coefficients for sampling probabilities to balance weights [S1].

### Steps

1.  **Initialization:** Apply the SLIC algorithm $F$ to partition the input RGB image $I$ into superpixels $L$:
    $$ \ell_i = F(T_{CIE}(I)), \quad \ell_i \in L $$
    where $T_{CIE}$ transforms the image from RGB to CIE color space [S1].

2.  **Initial Training (Random Sampling):** Perform standard random ray sampling and training. After each training session/iteration, record the rendering error $r_j$ for each sampled pixel $j$ within a superpixel [S1].

3.  **Superpixel Error Calculation:** Calculate the average error $\ell_{error}^i$ for each superpixel $i$:
    $$ \ell_{error}^i = \frac{1}{N} \sum_{j \in N} r_j $$
    where $N$ is the set of sampled pixels within superpixel $i$ [S1].

4.  **Secondary Segmentation (Re-segmentation):** Identify areas with the largest rendering errors and their adjacent regions. Use a masked SLIC algorithm to re-segment these specific regions [S1]. This step considers both RGB color and rendering error simultaneously [S1]. The render error is replicated three times to meet dimensional requirements for CIE transformation [S1]. The combined feature vector $C_k$ for the masked SLIC is:
    $$ C_k = \beta_0 \cdot T_{CIE}(I) + (1 - \beta_0) \cdot T_{CIE}(R) $$
    where $R$ is the portion of render error to be recalculated, and $\beta_0$ is the weight [S1].

5.  **Sampling Probability Calculation:** Calculate the sampling probability $\ell_p^i$ for each superpixel $i$ based on its error and standard deviation:
    $$ \ell_p^i = \lambda_{error} \cdot \ell_{std\_p}^i + \lambda_{std} \cdot \ell_{error}^i $$
    where $\ell_{std\_p}^i$ is the standard deviation of errors within the superpixel [S1]. The probability is normalized to lie between $0$ and $1$, and the threshold $t$ is applied to exclude negligible errors [S1].

6.  **Guided Sampling:** Use the calculated sampling probabilities to direct pixel-level ray sampling [S1]. Regenerate sampled rays according to the probability map [S1]. Accumulate rendering errors during this phase [S1].

7.  **Loss Computation:** Optimize neural field parameters using pixel-level Mean Squared Error (MSE) loss $\mathcal{L}_{mse}$ and superpixel loss $\mathcal{L}_{sp}$:
    $$ \mathcal{L}_{sp} = \ell_{error}^i + 0.001 \cdot \ell_{std\_p}^i $$
    Total loss:
    $$ \mathcal{L} = \lambda_{mse} \cdot \mathcal{L}_{mse} + \lambda_{sp} \cdot \mathcal{L}_{sp} $$
    Due to the strong bias of $\mathcal{L}_{sp}$, $\lambda_{sp}$ is set significantly smaller than $\lambda_{mse}$ (three to four orders of magnitude) [S1].

8.  **Position Encoding Adjustment:** If using frequency-based position encoding, the method may adjust the maximum frequency $K$ based on superpixel complexity:
    $$ \gamma_K'(x) = \gamma_K(x) * M(i) $$
    where $g_i(x)$ denotes the position encoding algorithm, $K$ represents the maximum frequency, and $\gamma_K(x)$ defines the specific formulation [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def train_SES(input_images, neural_field, hyperparams):
    beta_0 = hyperparams['beta_0']
    lambda_sp = hyperparams['lambda_sp']
    lambda_mse = hyperparams['lambda_mse']
    t = hyperparams['t']
    lambda_error = hyperparams['lambda_error']
    lambda_std = hyperparams['lambda_std']

    # Step 1: Initial Segmentation
    cie_images = [rgb_to_cie(img) for img in input_images]
    superpixel_maps = [slic_segment(cie_img) for cie_img in cie_images]

    # Step 2: Initial Random Sampling Training
    error_map = np.zeros_like(input_images[0][..., 0])
    for _ in range(1000):  # Initial training iterations [G]
        rays = generate_random_rays()
        colors, errors = render_rays(neural_field, rays)
        update_parameters(neural_field, errors, lambda_mse)
        error_map += errors

    # Step 3: Superpixel Error Calculation
    superpixel_errors = {}
    for sp_id in np.unique(superpixel_maps[0]):
        mask = (superpixel_maps[0] == sp_id)
        superpixel_errors[sp_id] = np.mean(error_map[mask])

    # Step 4: Secondary Segmentation
    high_error_sp_ids = np.argsort(list(superpixel_errors.values()))[-10:]  # Top regions [G]
    for sp_id in high_error_sp_ids:
        region_mask = (superpixel_maps[0] == sp_id)
        region_rgb = input_images[0][region_mask]
        region_error = error_map[region_mask]
        region_error_3d = np.stack([region_error] * 3, axis=-1)
        cie_region_rgb = rgb_to_cie(region_rgb)
        cie_region_error = rgb_to_cie(region_error_3d)
        combined_features = beta_0 * cie_region_rgb + (1 - beta_0) * cie_region_error
        new_sp = masked_slic(combined_features, region_mask)
        superpixel_maps[0][region_mask] = new_sp

    # Step 5: Sampling Probability Calculation
    sampling_probs = {}
    for sp_id in np.unique(superpixel_maps[0]):
        mask = (superpixel_maps[0] == sp_id)
        err = np.mean(error_map[mask])
        std = np.std(error_map[mask])
        prob = lambda_error * std + lambda_std * err
        sampling_probs[sp_id] = np.clip(prob - t, 0, 1)  # Normalize and apply threshold [G]

    # Step 6 & 7: Guided Sampling and Loss
    for _ in range(5000):  # Guided training iterations [G]
        rays = sample_rays_by_probability(sampling_probs, superpixel_maps[0])
        colors, errors = render_rays(neural_field, rays)
        mse_loss = np.mean((colors - ground_truth) ** 2)
        sp_loss = np.mean(error_map) + 0.001 * np.std(error_map)
        total_loss = lambda_mse * mse_loss + lambda_sp * sp_loss
        update_parameters(neural_field, total_loss)
        error_map = errors

def rgb_to_cie(rgb): return rgb  # Standard RGB to CIE transform [G]
def slic_segment(img): return np.zeros_like(img[..., 0], dtype=int)  # Standard SLIC [G]
def masked_slic(features, mask): return np.zeros_like(mask, dtype=int)  # SLIC restricted to mask [G]
def generate_random_rays(): return np.random.rand(1000, 3)  # Uniform random ray generation [G]
def render_rays(field, rays): return np.random.rand(1000, 3), np.random.rand(1000)  # Forward pass [G]
def update_parameters(field, loss, lr): pass  # Gradient descent step [G]
def sample_rays_by_probability(probs, sp_map): return np.random.rand(1000, 3)  # Weighted sampling [G]
```

### Edge Cases and Pitfalls
*   **Threshold $t$:** If $t$ is too high, well-learned regions may be excluded from sampling entirely, potentially leading to stagnation in those areas [S1].
*   **Weight $\beta_0$:** Synthetic data prefers a higher RGB weight ($0.8$), while real data prefers a higher error weight ($0.65$) [S1].
*   **Loss Bias:** The superpixel loss $\mathcal{L}_{sp}$ has a strong bias. If $\lambda_{sp}$ is not set sufficiently small (three to four orders of magnitude smaller than $\lambda_{mse}$), it can dominate the gradient and destabilize training [S1].
*   **Dimensionality:** The render error must be replicated three times to match the RGB channel dimension before CIE transformation [S1].

## Operating characteristics
*   **Time Complexity:** The method adds overhead for SLIC segmentation and probability calculation. However, it reduces the total number of volume queries by focusing on complex regions, potentially offsetting this cost. The overhead of SLIC is generally lower than quadtree partitioning for large-scale images [S1].
*   **Memory Complexity:** Requires storing superpixel maps and error maps. The memory footprint is comparable to standard NeRF training with additional storage for segmentation data [S1].
*   **Performance:** Integrated with Instant-NGP and DVGO baselines, SES improves rendering quality for high-resolution inputs (2k, 4k) compared to standard random sampling [S1].
*   **Strengths:** Resolves inadequate partitioning issues of quadtrees; reduces computational overhead per instance; effective for large input scales [S1].
*   **Limitations:** Requires careful tuning of $\beta_0$ and $\lambda_{sp}$; dependent on the quality of SLIC segmentation; scaling to extremely large scenes beyond the tested 2k/4k resolutions is not given in the sources [S1].

## Background
SES was proposed by Zhongwei Xuan et al. in 2024 for the ACM Multimedia (MM '24) conference [S1]. It was developed to address the limitations of existing adaptive sampling methods, such as quadtrees, which increase computational cost with input scale and cause fragmented information [S1]. The method is implemented on CUDA to leverage hardware performance and has been tested on synthetic and real datasets using NVIDIA RTX 4090 GPUs [S1]. It is integrated into popular neural field methods like Instant-NGP (via nerf-template/torch-ngp) and DVGO [S1].

## Sources
[S1] Superpixel-based Efficient Sampling for Learning Neural Fields from Large Input, https://doi.org/10.1145/3664647.3681299, fetched 2026-11-08
