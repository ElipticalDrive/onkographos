# NerfAcc Rendering Pipeline

Also called: Efficient Sampling for Volumetric Rendering, NeRF Volume Rendering (CPU/PyTorch), NerfAcc, Neural Radiance Fields (NeRF) with Coarse/Fine Sampling and NDC, Neural Volume Rendering (Coarse/Fine Sampling), Volume Rendering with Neural Fields

## Overview
The NerfAcc Rendering Pipeline is a modular, CUDA-accelerated framework for differentiable volumetric rendering that decouples the sampling strategy from the radiance field representation [S1]. It solves the computational bottleneck of dense ray marching by using a transmittance estimator to generate sparse, importance-sampled intervals along rays, enabling significant speedups (1.5× to 20×) over standard NeRF implementations [S1]. Unlike [NeRF Importance Sampling](1565-nerf-importance-sampling.md) which relies on fixed hierarchical passes, or [VF-NeRF](3138-vf-nerf.md) which uses voxel-based acceleration, NerfAcc provides a unified interface for integrating occupancy grids and proposal networks to skip empty space and concentrate samples on surfaces [S1].

## How it works
The pipeline operates in two main phases: efficient sampling to determine where to query the neural network, and differentiable rendering to integrate the results.

### Inputs and Outputs
**Inputs:**
*   `nerf`: A neural network model providing `density_fn` (returns $\sigma$) and `rgb_density_fn` (returns $\sigma$ and RGB).
*   `rays_o`: Ray origins, shape $(N_{rays}, 3)$ [S1].
*   `rays_d`: Normalized ray directions, shape $(N_{rays}, 3)$ [S1].
*   `estimator`: A `TransmittanceEstimator` instance (e.g., Occupancy Grid or Proposal Network) [S1].
*   `color_gt`: Ground truth RGB images for loss computation.

**Outputs:**
*   `color`: Rendered RGB image, shape $(N_{rays}, 3)$ [S1].
*   `opacity`: Accumulated opacity, shape $(N_{rays}, 1)$ [S1].
*   `depth`: Expected depth, shape $(N_{rays}, 1)$ [S1].
*   `aux`: Auxiliary data (e.g., weights, depths) for estimator updates [S1].

### Data Structures
Samples are represented as packed intervals rather than single points to support conical ray approximations [S1]:
*   `t0`: Start of interval along ray, shape $(N_{samples},)$ [S1].
*   `t1`: End of interval along ray, shape $(N_{samples},)$ [S1].
*   `r_id`: Ray index for each sample, shape $(N_{samples},)$ [S1].

### Parameters
*   `near`: Near plane depth. Typical value: 0.5 [S3, unofficial].
*   `far`: Far plane depth. Typical value: 3.0 [S3, unofficial].
*   `N_samples`: Number of coarse samples per ray. Default: 64 [S3, unofficial].
*   `N_importance`: Number of fine samples. If $>0$, fine sampling is performed. Default: 0 [S3, unofficial].
*   `perturb`: Jitter factor for coarse sampling. Default: 0 [S3, unofficial].
*   `noise_std`: Standard deviation for sigma noise. Default: 1 [S3, unofficial].
*   `chunk`: Batch size for network inference. Default: 32768 [S3, unofficial].
*   `white_back`: Boolean. If True, assumes white background. Default: False [S3, unofficial].

### Algorithm Steps

1.  **Initialize Estimator**: Create a transmittance estimator.
    ```python
    estimator = nerfacc.TransmittanceEstimator()
    ```
2.  **Define Query Functions**:
    ```python
    def density_fn(t0, t1, r_id):
        # Query raw sigma from NeRF
        return nerf.density(rays_o[r_id], rays_d[r_id], t0, t1)

    def rgb_density_fn(t0, t1, r_id):
        # Query sigma and RGB from NeRF
        return nerf.forward(rays_o[r_id], rays_d[r_id], t0, t1)
    ```
3.  **Efficient Sampling**: Use the estimator to generate sparse sample intervals. This step replaces fixed uniform or hierarchical sampling with importance sampling based on transmittance estimates.
    ```python
    # t0, t1, r_id are 1D tensors of packed samples
    t0, t1, r_id = nerfacc.sampling(
        rays_o, rays_d, estimator, density_fn=density_fn
    )
    ```
4.  **Differentiable Rendering**: Integrate the sampled intervals to compute color, opacity, and depth.
    ```python
    color, opacity, depth, aux = nerfacc.rendering(
        t0, t1, r_id, rgb_density_fn=rgb_density_fn
    )
    ```
5.  **Update Estimator**: Periodically update the transmittance estimator using the rendered auxiliary data to improve future sampling.
    ```python
    # Update every N training steps
    estimator.update_every_n_steps(t0, t1, r_id, aux)
    ```
6.  **Loss and Backprop**: Compute loss and backpropagate gradients to the NeRF and estimator.
    ```python
    loss = F.mse_loss(color, color_gt)
    loss.backward()
    ```

### Formulas
The rendering integration follows the standard volume rendering equation [S2, S3, unofficial]:

1.  **Alpha (Opacity)**:
    $$ \alpha_i = 1 - \exp(-\delta_i \sigma_i) $$
    Where $\delta_i$ is the distance between samples and $\sigma_i$ is the density. Noise is often added: $\sigma_i \leftarrow \text{relu}(\sigma_i + \mathcal{N}(0, \text{noise\_std}))$ [S3, unofficial].

2.  **Weight**:
    $$ w_i = \alpha_i \prod_{j=0}^{i-1} (1 - \alpha_j) $$
    Implemented efficiently using cumulative products:
    ```python
    alphas_shifted = torch.cat([torch.ones_like(alphas[:, :1]), 1 - alphas + 1e-10], dim=-1)
    weights = alphas * torch.cumprod(alphas_shifted, -1)[:, :-1]
    ```

3.  **Color**:
    $$ C = \sum_i w_i c_i $$

4.  **Depth**:
    $$ z = \sum_i w_i \frac{z_{i+1} + z_i}{2} $$

### Edge Cases and Pitfalls
*   **Division by Zero**: When computing ray-box intersections or NDC transformations, add epsilon ($10^{-7}$) to direction components to avoid division by zero [S3, unofficial].
*   **NaN in Hash Grids**: Clamp query points to the valid bounds of the hash grid encoder to prevent NaNs: `pts = pts.clamp(-bound, bound)` [S2, unofficial].
*   **Gradient Leakage**: When performing fine sampling (if implemented via `sample_pdf`), detach the sampled $z$ values from the computation graph to prevent gradients from flowing back to the coarse weights [S3, unofficial].
*   **Zero Weight Bins**: In PDF sampling, add epsilon to weights to prevent division by zero in CDF normalization [S3, unofficial].

## Subtopics

### [NeuSample](8053-neusample.md)
NeuSample replaces the coarse-to-fine hierarchical sampling stage with a lightweight "neural sample field" that maps a camera ray directly into a distribution of sample points [8053-neusample.md]. It enables single-shot inference without an intermediate coarse pass, reducing inference time by ~25% compared to standard NeRF while maintaining quality [8053-neusample.md]. Choose this when inference latency is critical and the overhead of a coarse pass is prohibitive.

### NeRF Upsampling (Fine Sampling)
A variant where additional samples are generated by stratified sampling (PDF sampling) based on the weights of the coarse pass [S2, unofficial]. Midpoints of coarse intervals are computed, and new $z$ values are sampled using `sample_pdf`. Gradients are detached from these new samples. This is used when higher frequency detail is needed in regions of high opacity [S2, unofficial].

### Occupancy Grid Sampling
A strategy using an Occupancy Grid to skip empty space within a predefined sphere radius (e.g., $r=1.5$ for synthetic scenes) [S4]. Points outside the bounding sphere have zero density. The occupancy threshold is typically 0.005 [S4]. Choose this for scenes with significant empty space to reduce query count.

### Neural Volume Renderer
A Python/PyTorch-based renderer supporting progressive rendering in a checkerboard pattern to achieve interactive frame rates (30-400 fps) [S5]. It upscales the finest-available image from an image hierarchy masked with fully evaluated pixels. Choose this for interactive visualization tools where immediate feedback is prioritized over full-frame accuracy [S5].

## Operating characteristics
*   **Time Complexity**: Depends on the sparsity of the scene and the effectiveness of the transmittance estimator. Typically $O(N_{samples})$ where $N_{samples} \ll N_{total}$ due to skipping empty space [S1].
*   **Memory Complexity**: Low, as it only stores active samples and their attributes. The estimator (Occupancy Grid or Proposal Network) adds memory overhead proportional to the grid size or network parameters [S1].
*   **Performance**: Speedups of 1.5× to 20× over standard NeRF implementations are reported [S1]. Interactive frame rates of 30-400 fps are achievable with progressive rendering [S5].
*   **Strengths**: Modular design allows easy integration of different radiance fields (density-based, SDF-based) and sampling strategies (Occupancy Grid, Proposal Network) [S1]. CUDA-accelerated kernels minimize Python overhead [S1].
*   **Limitations**: Requires careful tuning of estimator update frequency and parameters. Performance degrades if the estimator fails to accurately predict transmittance (e.g., in highly complex or dynamic scenes) [S1].
*   **When to Use**: When training or rendering NeRFs is too slow, or when integrating custom radiance fields with efficient sampling is desired [S1].
*   **When Not to Use**: For simple scenes where uniform sampling is sufficient and the overhead of maintaining an estimator is not justified.

## Background
NerfAcc was introduced by Li et al. in 2023 [S1]. It was designed to address the computational inefficiencies of early NeRF implementations by providing a universal, decoupled sampling framework. The library integrates concepts from Instant-NGP (Occupancy Grids) and Mip-NeRF 360 (Proposal Networks) into a unified transmittance estimator [S1]. It has become a standard tool for accelerating NeRF training and inference in research and applications [S1, S5].

## Sources
[S1] NerfAcc: Efficient Sampling Accelerates NeRFs, https://doi.org/10.1109/iccv51070.2023.01699, fetched 2026-10-26
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S3] GitHub - nitthilan/neural_sparse_voxel_field: Neural scene ..., https://github.com/nitthilan/neural_sparse_voxel_field, fetched 2026-10-28 (unofficial)
[S4] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
[S5] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
