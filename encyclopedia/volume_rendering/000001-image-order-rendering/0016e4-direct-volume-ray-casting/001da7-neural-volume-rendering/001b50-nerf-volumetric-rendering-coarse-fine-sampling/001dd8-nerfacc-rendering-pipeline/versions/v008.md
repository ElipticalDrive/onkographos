# NerfAcc Rendering Pipeline

Also called: Efficient Sampling for Volumetric Rendering, NeRF Volume Rendering (CPU/PyTorch), NerfAcc, Neural Radiance Fields (NeRF) with Coarse/Fine Sampling and NDC, Neural Volume Rendering (Coarse/Fine Sampling), Volume Rendering with Neural Fields

## Overview
NerfAcc is a standalone PyTorch library providing efficient sampling for volumetric rendering, decoupled from the radiance field representation [S1]. It accelerates Neural Radiance Fields (NeRF) by supporting Occupancy Grids and Proposal Networks to skip empty space and concentrate samples on surfaces [S1]. It builds on [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md) by replacing fixed uniform sampling with a transmittance estimator that dynamically updates during training [S1].

## How it works
**Inputs:** Ray origins $\mathbf{r}_o$ (shape $N_{rays}, 3$), normalized ray directions $\mathbf{r}_d$ (shape $N_{rays}, 3$), near/far bounds, and a neural radiance field model (NeRF) providing density $\sigma$ and RGB color.
**Outputs:** Rendered color, opacity, depth, and auxiliary data for estimator updates.

**Data Structures:**
*   **Packed Samples:** Represented as intervals $(t_0, t_1, r)$, where $t_0, t_1$ are start/end points along the $r$-th ray [S1]. This interval representation supports cone-based rays for anti-aliasing [S1].
*   **Ray Array:** Shape $(N_{rays}, 8)$. Columns 0-2: origin; 3-5: direction; 6: near depth; 7: far depth [S3, unofficial].

**Steps:**
1.  **Initialize Estimator:** Create a transmittance estimator (e.g., Proposal Network or Occupancy Grid) [S1].
    ```python
    estimator = nerfacc.TransmittanceEstimator()
    ```
2.  **Define Query Functions:**
    *   Density function: Queries NeRF for $\sigma$ given intervals [S1].
        ```python
        def density_fn(t0, t1, r_id):
            return nerf.density(r_o[r_id], r_d[r_id], t0, t1)
        ```
    *   RGB/Density function: Queries NeRF for $\sigma$ and RGB [S1].
        ```python
        def rgb_density_fn(t0, t1, r_id):
            return nerf.forward(r_o[r_id], r_d[r_id], t0, t1)
        ```
3.  **Efficient Sampling:** Sample intervals along rays using the estimator and density function [S1].
    ```python
    t0, t1, r_id = nerfacc.sampling(r_o, r_d, estimator, density_fn=density_fn)
    ```
4.  **Differentiable Rendering:** Integrate RGB and density over the sampled intervals [S1].
    ```python
    color, opacity, depth, aux = nerfacc.rendering(t0, t1, r_id, rgb_density_fn=rgb_density_fn)
    ```
5.  **Update Estimator:** Update the transmittance estimator with the sampled intervals and auxiliary data every $n$ steps [S1].
    ```python
    estimator.update_every_n_steps(t0, t1, r_id, aux)
    ```
6.  **Loss and Backprop:** Compute loss (e.g., MSE) and backpropagate [S1].
    ```python
    F.mse_loss(color, color_gt).backward()
    ```

**Formulas:**
*   **Opacity:** $\alpha = 1 - \exp(-\delta \sigma)$ [S2, unofficial].
*   **Weight:** $w_i = \alpha_i \prod_{j=0}^{i-1} (1 - \alpha_j)$ [S2, unofficial].
*   **Color:** $C = \sum w_i c_i$ [S2, unofficial].
*   **Depth:** $z = \sum w_i \frac{z_{i+1} + z_i}{2}$ [S2, unofficial].

**Parameters:**
*   `near`: Near plane depth. Typical: 0.5 [S3, unofficial].
*   `far`: Far plane depth. Typical: 3.0 [S3, unofficial].
*   `chunk`: Batch size for inference. Default: 1024*32 [S3, unofficial].
*   `white_back`: Boolean. If True, adds background weight to RGB [S3, unofficial].

**Edge Cases:**
*   **Division by Zero:** In ray-box intersection, add epsilon ($10^{-7}$) to direction components [S3, unofficial].
*   **NaN in Hash Grid:** Clamp points to $[-bound, bound]$ [S2, unofficial].
*   **Gradient Leakage:** Detach fine samples from coarse weights to prevent backpropagation [S3, unofficial].

## Subtopics

### [NeuSample](8053-neusample.md)
NeuSample replaces the coarse-to-fine stage with a neural sample field mapping rays directly to sample distributions, enabling single-shot inference and reducing inference time by ~25%.

### [Occupancy Grid Sampling](8562-occupancy-grid-sampling.md)
Uses a hash-based occupancy grid to skip empty space during ray marching, restricting samples to non-empty voxels within a bounding sphere. Unlike NeuSample, it relies on static spatial hashing.

### NeRF Upsampling (Fine Sampling)
Computes deltas and alphas from coarse samples, derives weights for PDF sampling, and samples new z_vals using `sample_pdf` on midpoints and weights [S2, unofficial].

### Neural Volume Renderer
A PyTorch renderer using NerfAcc for CUDA-accelerated marching, supporting progressive checkerboard rendering for 30-400 fps interactivity [S4].

## Operating characteristics
*   **Speedup:** 1.5× to 20× faster training compared to standard NeRF [S1].
*   **Memory:** Minimizes GPU memory via chunked inference and progressive rendering [S4].
*   **Interactivity:** Achieves 30-400 fps with progressive rendering [S4].
*   **Strengths:** Decoupled from radiance field, supports multiple acceleration structures (Occupancy Grid, Proposal Network) [S1].
*   **Limitations:** Requires integration of custom CUDA kernels for maximum speed [S1].

## Background
NerfAcc was introduced in 2023 to accelerate NeRF training by decoupling sampling from the radiance field [S1]. It builds on Instant-NGP and Mip-NeRF 360 concepts [S1].

## Sources
[S1] NerfAcc: Efficient Sampling Accelerates NeRFs, https://doi.org/10.1109/iccv51070.2023.01699, fetched 2026-10-26
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S3] GitHub - nitthilan/neural_sparse_voxel_field: Neural scene ..., https://github.com/nitthilan/neural_sparse_voxel_field, fetched 2026-10-28 (unofficial)
[S4] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
