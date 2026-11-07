# Photometric Optimization

Also called: EgoNeRF Training

## Overview
Photometric optimization is a training procedure that minimizes the difference between rendered images and observed target images by updating scene parameters via gradient descent [S1][S2]. It sets itself apart from its siblings by explicitly formulating the inverse problem as an unconstrained optimization task where latent parameters (such as shape vectors, pose angles, and appearance) are updated to minimize the photometric difference between a rendered image and an observed target [S1][S2]. Unlike [NeRF Training Loop](970-nerf-training-loop.md), which may focus on standard coordinate-based MLPs, this approach often jointly optimizes complex latent shape representations and appearance parameters [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A batch of rays $\mathcal{R}$, ground-truth colors $C(\mathbf{r})$ for each ray, and initial scene parameters (feature grids, MLP weights, environment map, or mesh vertices).
*   **Outputs:** Updated scene parameters that minimize the photometric loss.

**Data Structures:**
*   **Feature Grids:** $G_\sigma$ (density) and $G_a$ (albedo/appearance) stored in a balanced spherical grid or voxel structure [S1].
*   **Decoding MLP:** $f_{\text{MLP}}$ mapping grid features to final color [S1].
*   **Environment Map:** $E$ for global illumination [S1].
*   **Mesh Vertices:** Per-vertex deformation vectors $\Delta v$ and global movement vectors $\Delta m$ if geometry is explicitly represented [S2].

**Numbered Steps:**
1.  **Ray Sampling:** Randomly sample a batch of rays $\mathcal{R}$ from the training views [S1].
2.  **Forward Rendering:** Compute the rendered color $\hat{C}(\mathbf{r})$ for each ray using the volume rendering equation or differentiable rasterization [S1][S2].
3.  **Loss Computation:** Calculate the photometric loss $\mathcal{L}$ [S1]:
    $$
    \mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \|_2^2
    $$
    where $\hat{C}(\mathbf{r})$ is the rendered color and $C(\mathbf{r})$ is the ground-truth color [S1].
4.  **Regularization (Optional):** For real-world datasets with imperfect camera poses, add a Total Variation (TV) loss on the feature grid to reduce noise [S1].
5.  **Gradient Update:** Compute gradients $\nabla_\theta \mathcal{L}$ with respect to the parameters $\theta$ (feature grids, MLP weights, environment map, or mesh deformations) and update them using an optimizer (e.g., Adam) [S1][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def photometric_optimization_step(scene_params, rays, gt_colors, optimizer):
    """
    Performs one step of photometric optimization.
    
    Args:
        scene_params: Dictionary containing feature grids (G_sigma, G_a), 
                      decoding MLP (f_MLP), environment map (E), and/or 
                      mesh vertices (v, delta_v, delta_m).
        rays: Batch of rays r, each with origin and direction.
        gt_colors: Ground-truth colors C(r) for the rays.
        optimizer: Optimization object with step() method.
        
    Returns:
        loss: Scalar photometric loss value.
    """
    # 1. Forward Rendering
    # Compute rendered colors based on the current scene representation
    # For NeRF-style: use volume rendering equation
    # For Mesh-style: use differentiable rasterization with shading
    rendered_colors = render(scene_params, rays)
    
    # 2. Compute Photometric Loss
    # L = (1/|R|) * sum(|| C_hat(r) - C(r) ||_2^2)
    loss = mean_squared_error(rendered_colors, gt_colors)
    
    # 3. Optional Regularization (e.g., TV loss for feature grids)
    # Only applied if scene_params contains feature grids and 
    # dataset is real-world with imperfect poses
    if 'G_sigma' in scene_params:
        tv_loss = total_variation_loss(scene_params['G_sigma'])
        loss = loss + tv_loss
        
    # 4. Backward Pass and Update
    # Compute gradients and update parameters
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    return loss.item()
```

**Parameters:**
*   **Learning Rate:** Not given in the sources.
*   **Optimizer:** Adam is implied by the context of "gradient descent" and standard practice, but specific hyperparameters are not provided [S1][S2].
*   **TV Loss Weight:** Not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Imperfect Poses:** In real-world datasets, camera poses may be inaccurate, leading to noise in feature grids. Mitigated by adding TV loss [S1].
*   **Floating Noisy Points:** Removing per-vertex deformation vectors $\Delta v$ causes floating noisy points in the reconstructed mesh, degrading lighting quality [S2].
*   **Uniform Ray-Grid Hitting:** EgoNeRF uses a balanced feature grid to guarantee a nearly uniform ray-grid hitting rate, avoiding the need for coarse-to-fine reconstruction approaches [S1].

## Subtopics

### [Image-based reconstruction optimization](6677-image-based-reconstruction-optimization.md)
Image-based reconstruction optimization is a method that solves for object shapes, poses, textures, and lighting by minimizing image losses using gradients estimated via differentiable rendering pipelines. It sets itself apart from its siblings by explicitly formulating the inverse problem as an unconstrained optimization task where latent parameters (such as shape vectors and pose angles) are updated via gradient descent to minimize the photometric difference between a rendered image and an observed target. Unlike [Gradient-Based SE(3) Optimization](7526-gradient-based-se-3-optimization.md), which may focus on rigid body transformations, this approach often jointly optimizes complex latent shape representations and appearance parameters.

### [Gradient-Based SE(3) Optimization](7526-gradient-based-se-3-optimization.md)
This method refines the rigid transformation $\hat{T} \in SE(3)$ between two NeRF scenes (Scene A and Scene B) by minimizing photometric error on rays sampled from high-confidence regions. It differs from single-image alignment methods (like iNeRF) by optimizing over multiple images in the source set $C_A$ and utilizes Viewshed Fields (VF) to mask out unreliable pixels, thereby avoiding NeRF artifacts.

### Exploitation Stage Optimization Vectors
Optimization parameters used during the explicit mesh shading pass to refine geometry and scene elements (materials, lighting) [S2]. Removing the per-vertex deformation vector $\Delta v$ causes floating noisy points in the reconstructed mesh, which degrades lighting quality [S2]. Both $\Delta v$ and $\Delta m$ are crucial for high-quality reconstruction. Ablation results show full model PSNR/SSIM/LPIPS significantly outperforms variants without $\Delta v$ or $\Delta m$ on both NeRF-synthetic and RefNeRF-synthetic datasets [S2].

### NeRFLiX
NeRFLiX constructs training data $\{I_{gt}, I_{1r}, I_{2r}\}$ from an original image sequence $\{I, I_{1r}, I_{2r} | I_{gt}\}$, where $I_{ir}$ uses high-quality supervised views [S3]. It extracts useful information from two high-quality supervised views $I_{1r}$ and $I_{2r}$ to recover the enhanced view $\hat{I}$ [S3].

## Operating characteristics
*   **Time Complexity:** Depends on the number of rays $|\mathcal{R}|$, samples per ray $N$, and the complexity of the scene representation (MLP depth, grid size). The forward pass is $O(|\mathcal{R}| \cdot N)$, and the backward pass is similar.
*   **Memory Complexity:** Standard reverse-mode AD requires storing $O(kn)$ intermediate states, leading to high memory usage [S5, unofficial]. Adjoint methods or analytic inversion can reduce this to $O(n)$ [S1][S3][S5, unofficial].
*   **Strengths:** Directly optimizes for visual fidelity. Can handle complex scene representations including explicit geometry and materials [S2]. Joint optimization of appearance and geometry can lead to high-quality reconstructions [S2].
*   **Limitations:** Requires differentiable rendering. Sensitive to initialization and camera pose accuracy [S1]. May suffer from local minima if the scene representation is not well-regularized [S1].
*   **When to Use:** When high-fidelity image synthesis is the primary goal and differentiable rendering is available. Suitable for both synthetic and real-world datasets [S1][S2].
*   **When Not to Use:** When computational resources are extremely limited and approximate methods are acceptable. When ground-truth images are not available or are severely corrupted.

## Background
Photometric optimization is a fundamental technique in differentiable rendering and inverse graphics. It has been widely used in NeRF-based methods for novel view synthesis [S1] and in inverse rendering for decomposing scene elements such as geometry, spatially-varying materials, and lighting [S2]. The method relies on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [Parent]. EgoNeRF, a notable application, uses a balanced spherical grid to optimize feature grids, decoding MLP, and environment map jointly [S1]. IRCasTRF extends this to inverse rendering by optimizing cascaded tensorial radiance fields, lighting, and materials [S2]. NeRFLiX applies photometric optimization for image enhancement using supervised views [S3].

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
[S2] IRCasTRF: Inverse Rendering by Optimizing Cascaded Tensorial Radiance Fields, Lighting, and Materials From Multi-view Images, https://doi.org/10.1145/3581783.3612010, fetched 2026-10-27
[S3] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
