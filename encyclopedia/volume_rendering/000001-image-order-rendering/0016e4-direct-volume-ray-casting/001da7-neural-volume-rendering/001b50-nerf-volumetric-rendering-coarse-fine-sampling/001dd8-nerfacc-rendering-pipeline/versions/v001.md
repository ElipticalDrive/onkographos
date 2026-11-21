# NerfAcc Rendering Pipeline

Also called: Efficient Sampling for Volumetric Rendering, NeRF Volume Rendering (CPU/PyTorch), NerfAcc

## Overview
The NerfAcc Rendering Pipeline is a modular, high-performance volumetric rendering framework implemented as a PyTorch library that decouples efficient sampling from the radiance field representation [S1]. It solves the computational bottleneck of neural volume rendering by using a **Transmittance Estimator** (such as an Occupancy Grid or Proposal Network) to generate non-uniform sample intervals along rays, which are then integrated via differentiable rendering [S1]. Unlike standard uniform sampling or sibling methods like [NeRF Importance Sampling](1565-nerf-importance-sampling.md) which may sample the entire space, NerfAcc combines occupancy grids to skip empty space and proposal networks to concentrate samples on surfaces, achieving speedups of 1.5× to 20× with minimal code changes [S1].

## How it works
The pipeline operates in three main phases: initialization and coarse sampling, efficient interval-based sampling, and differentiable rendering with estimator updates.

### Inputs and Outputs
*   **Inputs**: Ray origins $r_o \in \mathbb{R}^{N \times 3}$, ray directions $r_d \in \mathbb{R}^{N \times 3}$ (normalized), near/far clipping planes ($near, far$), and a neural radiance field model `nerf` providing `density` and `forward` (RGB + density) functions.
*   **Outputs**: Rendered color $C$, opacity $\alpha$, depth $z$, and auxiliary data (e.g., depth variance) for the batch of rays.

### Data Structures
Samples are represented as **intervals** $(t_0, t_1, r)$ rather than single points $x$, where $t_0$ and $t_1$ are the start and end distances along the $r$-th ray [S1]. This interval representation supports cone-based anti-aliasing (e.g., Mip-NeRF) and avoids redundant queries at interval boundaries [S1].

### Steps

1.  **Initialize Transmittance Estimator**: Create an estimator to guide sampling. This can be an Occupancy Grid (from Instant-NGP) or a Proposal Network (from Mip-NeRF 360) [S1].
    ```python
    estimator = nerfacc.TransmittanceEstimator()
    ```

2.  **Initialize Coarse Samples**: Generate initial sample positions using log-space distribution, then map them to the $[near, far]$ range [S2, unofficial].
    ```python
    # Generate num_steps points in [0, 1] log-space
    z_vals = torch.logspace(0, 1, num_steps, base=2.0, device=device).unsqueeze(0).unsqueeze(0) - 1
    # Map to [near, far]
    z_vals = near + (far - near) * z_vals
    ```

3.  **Define Density Query Function**: Create a wrapper that queries the neural network for density given interval endpoints and ray indices [S1].
    ```python
    def density_fn(t0, t1, r_id):
        """Query density from nerf."""
        # r_o and r_d are batched ray origins and directions
        return nerf.density(r_o[r_id], r_d[r_id], t0, t1)
    ```

4.  **Perform Efficient Sampling**: Use the estimator and density function to generate packed sample intervals $(t_0, t_1, r\_id)$ [S1].
    ```python
    # t0, t1: (all_samples,), r_id: (all_samples,)
    t0, t1, r_id = nerfacc.sampling(
        r_o, r_d, estimator, density_fn=density_fn
    )
    ```

5.  **Define RGB/Density Query Function**: Create a wrapper that returns both color and density for the rendered intervals [S1].
    ```python
    def rgb_density_fn(t0, t1, r_id):
        """Query rgb and density from nerf."""
        return nerf.forward(r_o[r_id], r_d[r_id], t0, t1)
    ```

6.  **Differentiable Volumetric Rendering**: Integrate the samples to compute color, opacity, and depth [S1]. The internal rendering logic follows these formulas:
    *   **Deltas**: Compute interval lengths. For the last interval, use a fixed sample distance if needed [S2, unofficial].
        $$ \delta_i = t_{i+1} - t_i $$
        (Note: In implementation, deltas are computed between adjacent $z$ values, with the last delta potentially set to a constant `sample_dist` [S2, unofficial]).
    *   **Alpha**: Convert density $\sigma$ to opacity $\alpha$ [S2, unofficial].
        $$ \alpha_i = 1 - \exp(-\delta_i \sigma_i) $$
    *   **Weights**: Compute rendering weights $w_i$ using the cumulative product of transmittance [S2, unofficial].
        $$ w_i = \alpha_i \prod_{j=0}^{i-1} (1 - \alpha_j) $$
        To ensure numerical stability, the cumulative product uses shifted alphas:
        $$ \text{alphas\_shifted} = \text{concat}([1, 1 - \alpha + 10^{-15}]) $$
        $$ w_i = \alpha_i \cdot \text{cumprod}(\text{alphas\_shifted})_{i-1} $$
    *   **Color**: Weighted sum of colors $c_i$ [S2, unofficial].
        $$ C = \sum_i w_i c_i $$
    *   **Depth**: Weighted mean of midpoints $z_{mid}$ [S2, unofficial].
        $$ z = \sum_i w_i \frac{t_{i+1} + t_i}{2} $$
    *   **Depth Variance**: Weighted variance of depths [S2, unofficial].
        $$ Var = \sum_i w_i \frac{(t_{i+1}-z)^2 + (t_{i+1}-z)(t_i-z) + (t_i-z)^2}{3} $$

    ```python
    color, opacity, depth, aux = nerfacc.rendering(
        t0, t1, r_id, rgb_density_fn=rgb_density_fn
    )
    ```

7.  **Background Contribution**: If alpha is not premultiplied, add background color based on accumulated opacity [S2, unofficial].
    ```python
    # weights_sum is the sum of weights (opacity)
    if alpha_premultiplied:
        image = color
    else:
        image = color / (weights_sum + 1e-9)
        image = image + (1 - weights_sum).unsqueeze(-1) * bg_color
    ```

8.  **Update Transmittance Estimator**: Periodically update the estimator (e.g., every $n$ steps) using the sampled intervals and auxiliary data to improve future sampling [S1].
    ```python
    estimator.update_every_n_steps(t0, t1, r_id, aux)
    ```

9.  **Loss and Backpropagation**: Compute loss against ground truth and backpropagate [S1].
    ```python
    F.mse_loss(color, color_gt).backward()
    ```

### Parameters and Typical Values
*   `num_steps`: Number of coarse samples per ray (e.g., 64, 128).
*   `near`, `far`: Clipping distances (scene-dependent).
*   `update_every_n_steps`: Frequency of estimator updates (e.g., 10, 100).
*   `bound`: Clamping bound for hashgrid encoders to prevent NaNs (e.g., 1.0, 2.0) [S2, unofficial].

### Edge Cases and Pitfalls
*   **NaN in HashGrid Encoders**: Points must be clamped to $[-bound, bound]$ before querying hashgrid-based encoders to prevent NaN gradients [S2, unofficial].
    ```python
    pts = pts.clamp(-self.bound, self.bound)
    ```
*   **Numerical Stability**: When computing cumulative products for transmittance, add a small epsilon ($10^{-15}$) to $1 - \alpha$ to prevent underflow and NaNs [S2, unofficial].
*   **Interval Boundaries**: The interval-based representation $(t_0, t_1)$ avoids querying the same point twice at boundaries, but requires careful handling of the last interval's delta if it extends beyond the last sampled point [S1][S2, unofficial].

## Subtopics

### NeRF Upsampling (Fine Sampling)
A variant that refines coarse samples by resampling around high-weight intervals. It computes deltas and alphas from coarse samples, derives weights for PDF sampling, calculates midpoints, and samples new $z$ values using `sample_pdf`. New points are queried for density and RGB, and results are gathered using sort indices to maintain order [S2, unofficial]. This step is typically performed after the coarse pass to improve detail without excessive sampling density [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Depends on the number of samples generated by the estimator. By skipping empty space (Occupancy Grid) and concentrating on surfaces (Proposal Network), it reduces the number of neural network queries compared to uniform sampling [S1].
*   **Memory Complexity**: Linear with respect to the number of active samples. The interval-based representation $(t_0, t_1, r)$ is memory-efficient as it avoids storing redundant point coordinates [S1].
*   **Performance**: Reported speedups of 1.5× to 20× over baseline NeRF implementations with minor code modifications [S1].
*   **Strengths**: Modular design allows easy integration with various radiance fields (density-based or SDF-based, static or dynamic) [S1]. Combines the strengths of Occupancy Grids and Proposal Networks [S1].
*   **Limitations**: Requires careful tuning of the estimator parameters. The interval-based approach may complicate direct point-wise analysis if not handled correctly [S1].

## Background
NerfAcc was introduced in the paper "NerfAcc: Efficient Sampling Accelerates NeRFs" (ICCV 2023) [S1]. It was designed to address the computational inefficiency of neural radiance field rendering by providing a universal, decoupled sampling framework. The library fuses operations into CUDA kernels for maximum speed while exposing flexible Python APIs [S1]. It builds on techniques from Instant-NGP (Occupancy Grids) and Mip-NeRF 360 (Proposal Networks) [S1].

## Sources
[S1] NerfAcc: Efficient Sampling Accelerates NeRFs, https://doi.org/10.1109/iccv51070.2023.01699, fetched 2026-10-26
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
