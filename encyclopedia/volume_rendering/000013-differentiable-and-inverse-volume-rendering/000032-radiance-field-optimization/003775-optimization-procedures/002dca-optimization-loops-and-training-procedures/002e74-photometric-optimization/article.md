# Photometric Optimization

Also called: EgoNeRF Training

## Overview
Photometric optimization is an inverse rendering technique that recovers latent 3D scene parameters—such as geometry, pose, texture, and lighting—from one or more observed images by minimizing the difference between rendered and target images using gradient-based optimization [S2]. It differs from direct feed-forward reconstruction methods by iteratively refining parameters via backpropagation through a differentiable renderer, allowing for the incorporation of shape priors and complex lighting models [S2]. In the context of radiance-field optimization, it specifically refers to the joint minimization of a photometric loss $\mathcal{L}$ between the rendered color $\hat{C}(\mathbf{r})$ and the ground-truth color $C(\mathbf{r})$ for a batch of rays $\mathcal{R}$ [G].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A batch of rays $\mathcal{R}$ sampled from training images; ground-truth colors $C(\mathbf{r})$ for those rays; initial scene parameters $\theta$ (e.g., feature grids, MLP weights, environment map).
*   **Outputs:** Optimized scene parameters $\theta^*$ that minimize the photometric error.

**Data Structures:**
The specific data structures depend on the representation being optimized. For EgoNeRF [S1], this includes:
*   Feature grids $G_\sigma$ (density) and $G_a$ (appearance).
*   Decoding MLP $f_{\text{MLP}}$.
*   Environment map $E$.
*   For IRCasTRF [S2], optimization parameters include per-vertex deformation vectors $\Delta v$ and global movement vectors $\Delta m$.

**Numbered Steps:**
1.  **Ray Sampling:** Randomly sample a batch of rays $\mathcal{R}$ from the training camera poses [S1].
2.  **Forward Pass:** Render the predicted color $\hat{C}(\mathbf{r})$ for each ray $\mathbf{r} \in \mathcal{R}$ using the current parameters $\theta$ and the differentiable volume rendering equation [G].
3.  **Loss Computation:** Calculate the photometric loss $\mathcal{L}$ [S1]:
    $$
    \mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left\| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \right\|_2^2
    $$
    where $|\mathcal{R}|$ is the number of rays in the batch [S1].
4.  **Regularization (Optional):** For real-world datasets with imperfect camera poses, add a Total Variation (TV) loss on the feature grid to reduce noise [S1].
5.  **Backpropagation:** Compute gradients $\nabla_\theta \mathcal{L}$ with respect to all trainable parameters $\theta$ [G].
6.  **Update:** Update parameters $\theta$ using an optimizer (e.g., Adam) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def photometric_optimization_step(theta, ray_batch, gt_colors, optimizer, use_tv_loss=False, feature_grid=None):
    """
    Performs one step of photometric optimization.
    
    Args:
        theta: Current scene parameters (grids, MLP, env map, etc.)
        ray_batch: Batch of rays R
        gt_colors: Ground truth colors C(r) for the rays
        optimizer: Optimization object with update method
        use_tv_loss: Boolean flag to enable TV regularization
        feature_grid: Reference to feature grid for TV loss calculation
        
    Returns:
        Updated theta
    """
    # 1. Forward Pass: Render colors
    # Assumes a differentiable renderer function exists that takes theta and rays
    rendered_colors = render(theta, ray_batch)
    
    # 2. Loss Computation: Photometric MSE
    # L = (1/|R|) * sum(||C_hat(r) - C(r)||_2^2)
    photometric_loss = mean_squared_error(rendered_colors, gt_colors)
    
    total_loss = photometric_loss
    
    # 3. Regularization (Optional)
    if use_tv_loss and feature_grid is not None:
        # Add TV loss to reduce noise in feature grids for imperfect poses
        tv_loss = compute_total_variation(feature_grid)
        total_loss = total_loss + tv_loss
        
    # 4. Backpropagation and Update
    # Compute gradients and update parameters
    optimizer.zero_grad()
    total_loss.backward()
    optimizer.step()
    
    return theta
```

**Parameters:**
*   **Learning Rate:** Not given in the sources.
*   **Batch Size:** Not given in the sources.
*   **TV Loss Weight:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Imperfect Poses:** In real-world datasets, camera poses are often imperfect. Without regularization, this leads to noise in the reconstructed scene. The TV loss on feature grids mitigates this [S1].
*   **Missing Deformation Vectors:** In inverse rendering pipelines that optimize geometry explicitly (e.g., IRCasTRF), removing the per-vertex deformation vector $\Delta v$ causes floating noisy points in the reconstructed mesh, which degrades lighting quality [S2]. Both $\Delta v$ and global movement vector $\Delta m$ are crucial for high-quality reconstruction [S2].
*   **Coarse-to-Fine Requirement:** Unlike some feature grid-based methods, EgoNeRF does not require a coarse-to-fine reconstruction approach for robust optimization because its balanced feature grid guarantees a nearly uniform ray-grid hitting rate [S1].

## Subtopics

### [Image-based reconstruction optimization](6677-image-based-reconstruction-optimization.md)
Image-based reconstruction optimization is an inverse rendering technique that recovers latent 3D scene parameters—such as geometry, pose, texture, and lighting—from one or more observed images by minimizing the difference between rendered and target images using gradient-based optimization. It differs from direct feed-forward reconstruction methods by iteratively refining parameters via backpropagation through a differentiable renderer, allowing for the incorporation of shape priors and complex lighting models.

### [Gradient-Based SE(3) Optimization](7526-gradient-based-se-3-optimization.md)
This method refines the rigid transformation $\hat{T} \in SE(3)$ between two NeRF scenes (Scene A and Scene B) by minimizing photometric error on rays sampled from high-confidence regions. It differs from single-image alignment methods (like iNeRF) by optimizing over multiple images in the source set $C_A$ and utilizes Viewshed Fields (VF) to mask out unreliable pixels, thereby avoiding NeRF artifacts.

### Exploitation Stage Optimization Vectors
Optimization parameters used during the explicit mesh shading pass to refine geometry and scene elements (materials, lighting) [S2]. Removing the per-vertex deformation vector $\Delta v$ causes floating noisy points in the reconstructed mesh, which degrades lighting quality [S2]. Both $\Delta v$ and $\Delta m$ are crucial for high-quality reconstruction; ablation results show the full model significantly outperforms variants without these vectors on NeRF-synthetic and RefNeRF-synthetic datasets [S2].

### NeRFLiX
NeRFLiX constructs training data $\{I_{gt}, I_{1r}, I_{2r}\}$ from an original image sequence $\{I, I_{1r}, I_{2r} | I_{gt}\}$, where $I_{ir}$ uses high-quality supervised views [S3]. It extracts useful information from two high-quality supervised views $I_{1r}$ and $I_{2r}$ to recover the enhanced view $\hat{I}$ [S3].

## Operating characteristics
*   **Time Complexity:** Depends on the number of rays $|\mathcal{R}|$, the depth of the rendering (number of samples per ray), and the complexity of the scene representation (e.g., MLP depth, grid size). Each iteration requires a forward pass through the renderer and a backward pass for gradient computation.
*   **Memory Complexity:** Proportional to the size of the scene representation (feature grids, MLP weights) and the batch size of rays.
*   **Strengths:** Jointly optimizes all scene parameters, allowing for consistent recovery of geometry, appearance, and lighting. Can handle complex lighting models and shape priors [S2].
*   **Limitations:** Computationally expensive due to iterative optimization. Sensitive to initialization and camera pose accuracy. Requires careful regularization (e.g., TV loss) for real-world data with imperfect poses [S1].
*   **When to Use:** When high-fidelity reconstruction of geometry, materials, and lighting is required from multi-view images, and computational resources allow for iterative optimization.
*   **When Not to Use:** For real-time applications or when only a single forward pass is feasible.

## Background
Photometric optimization is a fundamental technique in inverse rendering and differentiable volume rendering. It has been widely used in methods such as EgoNeRF [S1], IRCasTRF [S2], and NeRFLiX [S3]. The approach builds on the principle of minimizing the photometric loss between rendered and ground-truth images to recover scene parameters.

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
[S2] IRCasTRF: Inverse Rendering by Optimizing Cascaded Tensorial Radiance Fields, Lighting, and Materials From Multi-view Images, https://doi.org/10.1145/3581783.3612010, fetched 2026-10-27
[S3] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
