# Photometric Optimization

Also called: EgoNeRF Training

## Overview
Photometric optimization is the core inverse rendering procedure that minimizes the discrepancy between rendered colors $\hat{C}(\mathbf{r})$ and ground-truth colors $C(\mathbf{r})$ for a batch of rays $\mathcal{R}$ [S1]. It differs from [NeRF Training Loop](970-nerf-training-loop.md) by its specific application to egocentric or large-scale scenes where balanced feature grids allow for robust optimization without coarse-to-fine strategies [S1]. It serves as the foundational loss minimization step for [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md) and [Staggered Optimization Scheme](5832-staggered-optimization-scheme.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** A batch of rays $\mathcal{R}$ sampled from training views; ground-truth colors $C(\mathbf{r})$ for each ray; current parameters for feature grids $G_\sigma, G_a$, decoding MLP $f_{\text{MLP}}$, and environment map $E$ [S1].
*   **Outputs:** Updated parameters $\theta$ that minimize the photometric loss [S1].

**Data Structures:**
*   **Feature Grids:** $G_\sigma$ (density) and $G_a$ (appearance) stored on a balanced spherical grid [S1]. The grid structure ensures a nearly uniform ray-grid hitting rate [S1].
*   **Decoding MLP:** $f_{\text{MLP}}$ takes grid samples and view direction to predict color [S1].
*   **Environment Map:** $E$ captures global illumination [S1].

**Numbered Steps:**
1.  **Ray Sampling:** Randomly sample a batch of rays $\mathcal{R}$ from the training camera poses [S1].
2.  **Forward Pass:** For each ray $\mathbf{r} \in \mathcal{R}$, perform volume rendering using the current parameters ($G_\sigma, G_a, f_{\text{MLP}}, E$) to compute the predicted color $\hat{C}(\mathbf{r})$ [S1].
3.  **Loss Computation:** Calculate the photometric loss $\mathcal{L}$ using the mean squared error between predicted and ground-truth colors [S1]:
    $$
    \mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left\| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \right\|_2^2
    $$
    where $|\mathcal{R}|$ is the number of rays in the batch [S1].
4.  **Regularization (Real-world):** If optimizing on real-world datasets with imperfect camera poses, add a Total Variation (TV) loss on the feature grids $G_\sigma, G_a$ to reduce noise [S1].
5.  **Backpropagation:** Compute gradients of $\mathcal{L}$ with respect to all parameters $\theta = \{G_\sigma, G_a, f_{\text{MLP}}, E\}$ [S1].
6.  **Update:** Update parameters using an optimizer (e.g., Adam) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def photometric_optimization_step(rays, ground_truth_colors, feature_grid_sigma, feature_grid_appearance, mlp, env_map, optimizer, use_tv_loss=False):
    """
    Performs one step of photometric optimization for radiance fields.
    
    Args:
        rays: Batch of rays R.
        ground_truth_colors: Ground truth colors C(r) for each ray.
        feature_grid_sigma: Density feature grid G_sigma.
        feature_grid_appearance: Appearance feature grid G_a.
        mlp: Decoding MLP f_MLP.
        env_map: Environment map E.
        optimizer: Optimization state.
        use_tv_loss: Boolean flag for adding TV regularization.
        
    Returns:
        Updated parameters and loss value.
    """
    # 1. Forward Pass: Render colors
    # Note: Volume rendering implementation details are assumed to be in the 
    # rendering module, using G_sigma, G_a, mlp, and env_map.
    predicted_colors = volume_render(rays, feature_grid_sigma, feature_grid_appearance, mlp, env_map)
    
    # 2. Compute Photometric Loss
    # L = (1/|R|) * sum(||C_hat(r) - C(r)||_2^2)
    photometric_loss = mean_squared_error(predicted_colors, ground_truth_colors)
    
    # 3. Add Regularization if needed (for real-world datasets)
    total_loss = photometric_loss
    if use_tv_loss:
        # TV loss on feature grids to reduce noise
        tv_loss_sigma = total_variation(feature_grid_sigma)
        tv_loss_appearance = total_variation(feature_grid_appearance)
        total_loss = total_loss + tv_loss_sigma + tv_loss_appearance
        
    # 4. Backpropagation
    gradients = compute_gradients(total_loss, [feature_grid_sigma, feature_grid_appearance, mlp.parameters(), env_map])
    
    # 5. Update Parameters
    optimizer.step(gradients)
    
    return total_loss, predicted_colors
```

**Parameters:**
*   **Learning Rate:** Not given in the sources [S1].
*   **Batch Size:** Not given in the sources [S1].
*   **TV Loss Weight:** Not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Imperfect Poses:** On real-world datasets, imperfect camera poses can introduce noise; mitigated by adding TV loss on feature grids [S1].
*   **Grid Hitting Rate:** The balanced feature grid design ensures uniform ray-grid hitting, avoiding the need for coarse-to-fine strategies required by other methods [S1].

## Subtopics

### [Image-based reconstruction optimization](6677-image-based-reconstruction-optimization.md)
Optimization of latent shape vector, pose, texture, and lighting parameters to minimize the difference between a rendered image and an observed image using a differentiable renderer. It focuses on joint optimization of geometry and appearance attributes [6677-image-based-reconstruction-optimization.md].

### [Gradient-Based SE(3) Optimization](7526-gradient-based-se-3-optimization.md)
A gradient-descent optimization process to refine the transformation $\hat{T}$ between Scene A and Scene B by minimizing photometric loss on rays sampled from high viewshed score regions. It specifically targets pose refinement rather than scene representation [7526-gradient-based-se-3-optimization.md].

### Exploitation Stage Optimization Vectors
Optimization parameters used during the explicit mesh shading pass to refine geometry and scene elements (materials, lighting). Specifically, it involves per-vertex deformation vectors $\Delta v$ and global movement vectors $\Delta m$. Removing $\Delta v$ causes floating noisy points in the reconstructed mesh, degrading lighting quality. Both vectors are crucial for high-quality reconstruction, with ablation studies showing significant performance drops without them [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays $|\mathcal{R}|$ and the depth of the volume rendering integration [S1].
*   **Memory Complexity:** Dominated by the storage of feature grids $G_\sigma, G_a$ and the environment map $E$ [S1].
*   **Strengths:** The balanced feature grid guarantees a nearly uniform ray-grid hitting rate, allowing for robust optimization without coarse-to-fine reconstruction approaches [S1].
*   **Limitations:** Requires careful handling of real-world data noise via additional regularization (TV loss) [S1].
*   **When to use:** Suitable for large-scale egocentric view synthesis and inverse rendering tasks where uniform sampling efficiency is critical [S1].
*   **When not:** Not explicitly stated, but coarse-to-fine methods may be preferred if grid balancing is not applicable [S1].

## Background
Originated in the context of EgoNeRF, a method for egocentric view synthesis using a balanced spherical grid [S1]. The approach was developed to address challenges in large-scale environment capture where traditional NeRF methods struggle with pose accuracy and sampling efficiency [S1]. It builds on the principles of differentiable volume rendering and inverse rendering [S1].

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
[S2] IRCasTRF: Inverse Rendering by Optimizing Cascaded Tensorial Radiance Fields, Lighting, and Materials From Multi-view Images, https://doi.org/10.1145/3581783.3612010, fetched 2026-10-27
