# Specific Method Implementations


## Overview
This family details concrete loss function formulations and training routines used in specific radiance-field optimization methods [G]. It covers implementations ranging from neural implicit fields to Gaussian splatting and sparse voxel representations [G]. These implementations differ in their component terms (e.g., Eikonal, closed-form color, PBR parameters) and their optimization schedules (e.g., delayed updates, early stopping) [G].

## Shared principles
Not given in the sources.

## Subtopics

### [HybridNeRF Optimization Loss](6045-hybridnerf-optimization-loss.md)
The HybridNeRF optimization loss is a composite objective function designed to train neural radiance fields that blend volumetric and surface-based representations. It solves the problem of balancing photometric accuracy with geometric regularity by combining a standard photometric term with an Eikonal regularization term and an interlevel proposal loss. It is distinguished from sibling methods like [CF Loss](10573-cf-loss.md) by its specific integration of the Eikonal constraint to enforce signed distance field (SDF) properties and its use of a proposal network for efficient sampling.

The method optimizes a total loss $\mathcal{L}(\mathbf{r})$ for a ray $\mathbf{r}$, defined as:
$$ \mathcal{L}(\mathbf{r}) = \mathcal{L}_{\text{photo}}(\mathbf{r}) + \lambda_{\text{Eik}} \mathcal{L}_{\text{Eik}}(\mathbf{r}) + \mathcal{L}_{\text{prop}}(\mathbf{r}) $$

1.  **Photometric Loss ($\mathcal{L}_{\text{photo}}$):** Measures the difference between rendered and ground-truth images. Typically implemented as $L_1$ or MSE.
2.  **Eikonal Loss ($\mathcal{L}_{\text{Eik}}$):** Enforces the Eikonal equation $|\nabla S(\mathbf{x})| = 1$ for points sampled in free space or near surfaces, ensuring the predicted field represents a valid SDF.
3.  **Interlevel Proposal Loss ($\mathcal{L}_{\text{prop}}$):** Regularizes the proposal network to ensure consistency between coarse and fine sampling levels.

**Parameters:**
*   $\lambda_{\text{Eik}}$: Weight for the Eikonal loss. Typical value: $0.01$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_nerf_loss(rendered_color, gt_color, sdf_gradient_norm, proposal_loss, lambda_eik=0.01):
    """
    Computes the HybridNeRF optimization loss.
    
    Args:
        rendered_color (Tensor): Rendered pixel color.
        gt_color (Tensor): Ground truth pixel color.
        sdf_gradient_norm (Tensor): Norm of the SDF gradient |grad(phi)|.
        proposal_loss (Tensor): Interlevel proposal loss.
        lambda_eik (float): Weight for the Eikonal loss.
        
    Returns:
        Tensor: Total loss scalar.
    """
    # 1. Photometric Loss
    l_photo = torch.mean(torch.abs(rendered_color - gt_color))
    
    # 2. Eikonal Loss
    eikonal_error = (sdf_gradient_norm - 1.0) ** 2
    l_eik = torch.mean(eikonal_error)
    
    # 3. Total Loss
    total_loss = l_photo + lambda_eik * l_eik + proposal_loss
    
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Surfaceness Threshold:** Uniformly increasing the "surfaceness" parameter $\beta$ degrades rendering quality near fine-grained and transparent structures. The method uses spatially adaptive $\beta(\mathbf{x})$ via a $512^3$ voxel grid to mitigate this.
*   **Gradient Precision:** The Eikonal loss requires accurate gradient computation of the SDF field. Numerical instability can occur if the network does not converge to a valid SDF prior to full optimization.

### [CF Loss](10573-cf-loss.md)
CF Loss is a regularization term used in radiance-field optimization to mitigate shape-radiance ambiguity by enforcing geometric consistency through a closed-form estimation of color. Unlike standard photometric losses that measure pixel-wise color differences, CF Loss computes the error between the rendered image and a color field reconstructed directly from the density and albedo fields without ray marching, thereby providing strong supervision signals specifically for geometric errors. It sets itself apart from sibling losses (e.g., photometric, distortion) by being significantly more sensitive to false geometry, such as thick or floating surfaces, which other losses often ignore.

**Inputs and Outputs:**
*   **Inputs:**
    *   $\mathcal{R}_{cf}$: A set of $|\mathcal{R}_{cf}|$ rays sampled per iteration.
    *   $\sigma(\mathbf{x})$: The density field value at position $\mathbf{x}$.
    *   $\mathbf{c}(\mathbf{x})$: The color (albedo) field value at position $\mathbf{x}$.
    *   $\mathbf{C}_{\text{rendered}}$: The standard volume-rendered color for the rays.
    *   $\mathbf{C}_{\text{gt}}$: The ground-truth image colors for the rays.
*   **Outputs:**
    *   $\mathcal{L}_{cf}$: The scalar loss value.

**Data Structures:**
*   Rays are represented as origin $\mathbf{o}$ and direction $\mathbf{d}$ pairs.
*   The closed-form color field is not stored as a separate grid but computed analytically from the existing density and color fields.

**Method Steps:**
1.  **Sample Rays:** Select $|\mathcal{R}_{cf}|$ rays from the training view.
2.  **Compute Closed-Form Color:** For each ray, compute the color $\mathbf{C}_{cf}$ using the closed-form integration of the density and color fields. The exact mathematical derivation is not explicitly detailed in the source text, but it relies solely on the density field and accumulated transmittance to decouple color from the explicit color network's parameters.
3.  **Calculate Loss:** Compute the $L_1$ norm (or similar) between the closed-form color $\mathbf{C}_{cf}$ and the ground truth $\mathbf{C}_{\text{gt}}$ (or the standard render) for the sampled rays.
4.  **Aggregate:** Average the loss over the $|\mathcal{R}_{cf}|$ rays.
5.  **Weight:** Multiply by the weight factor $\lambda$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cf_loss(rays, density_field, color_field, gt_colors, lambda_cf, num_cf_rays):
    """
    Computes the Closed-Form Color Loss.
    
    Args:
        rays: List of ray origins and directions.
        density_field: Function/Structure to query density sigma(x).
        color_field: Function/Structure to query color c(x).
        gt_colors: Ground truth colors for the rays.
        lambda_cf: Weight factor for the loss.
        num_cf_rays: Number of rays to sample for CF loss calculation.
        
    Returns:
        loss: Scalar CF loss value.
    """
    cf_rays = rays[:num_cf_rays]
    gt_subset = gt_colors[:num_cf_rays]
    total_loss = 0.0
    count = 0
    
    for ray, gt_color in zip(cf_rays, gt_subset):
        # Closed-form color computation is not explicitly detailed in sources [G]
        # Approximated here as standard volume rendering integration for demonstration
        c_cf = 0.0
        T = 1.0
        steps = 32
        step_size = 1.0 / steps
        for i in range(steps):
            x = ray.origin + (i * step_size) * ray.direction
            sigma = density_field(x)
            c = color_field(x)
            alpha = 1.0 - torch.exp(-sigma * step_size)
            c_cf += T * alpha * c
            T *= (1.0 - alpha)
            
        error = torch.mean(torch.abs(c_cf - gt_color))
        total_loss += error
        count += 1
        
    if count > 0:
        return lambda_cf * (total_loss / count)
    return 0.0
```

**Parameters:**
*   $\lambda$ (Weight factor): Controls the influence of CF loss. Typical values: 1, 2, 5, 10. Optimal range is 1-10. Values $\ge 20$ degrade performance.
*   $|\mathcal{R}_{cf}|$ (Number of rays): Number of rays used for CF loss per iteration. Typical values: 10, 25. Higher values improve geometry but increase cost.

**Edge Cases and Pitfalls:**
*   **High Weight:** If $\lambda$ is too large (e.g., 20), the CF loss dominates training, causing PSNR to decrease because the color field cannot be well optimized while only the density field receives strong geometric supervision.
*   **Computational Cost:** Increasing $|\mathcal{R}_{cf}|$ significantly increases training time (e.g., from 17 mins to 43 mins on DTU for 25 rays).

### RocNet Loss Function
RocNet uses a recursive octree structure where the loss function consists of two separate parts: the node labeling loss and the leaf reconstruction loss [S1].

**Formula:**
*   Total loss $L = L_l + L_r$, where $L_l$ is node labeling loss and $L_r$ is leaf reconstruction loss [S1].
*   Node labeling loss $L_l = - \frac{1}{4} \sum_{i=1}^{4} c_i \log(s_i)$, where $c_i$ is ground-truth and $s_i$ is node classifier score for 4 node types. The loss of an octree is the sum of label losses of all nodes [S1].

### Mobile-GS Training Loss
Mobile-GS optimizes a model for real-time rendering on mobile devices using a combination of rendering, distillation, and depth losses [S2, unofficial].

**Formula:**
*   Rendering loss: $\mathcal{L}_{\text{rgb}} = \lambda \mathcal{L}_1(\mathbf{C}, \mathbf{C}_{\text{gt}}) + (1 - \lambda) \mathcal{L}_{\text{DSSIM}}(\mathbf{C}, \mathbf{C}_{\text{gt}})$, where $\lambda$ balances L1 and DSSIM losses [S2, unofficial].
*   Total loss: $\mathcal{L} = \mathcal{L}_{\text{rgb}} + \lambda_{\text{distill}} \mathcal{L}_{\text{distill}} + \lambda_{\text{depth}} \mathcal{L}_{\text{depth}}$ [S2, unofficial].

**Parameters:**
*   $\lambda$: Weight for L1 vs DSSIM in rendering loss. Value: 0.8 [S2, unofficial].
*   $\lambda_{\text{distill}}$: Weight for distillation loss. Value: 1 [S2, unofficial].
*   $\lambda_{\text{depth}}$: Weight for depth loss. Value: 0.1 [S2, unofficial].

### ExBluRF Loss Function
ExBluRF handles extreme motion blur by minimizing photoconsistency loss with total variation and sparsity priors on voxel grids [S3].

**Step:**
*   Apply Total Variation (TV) loss to both density grid $G_\sigma$ and SH coefficient grid $G_{sh}$ [S3].

**Formula:**
*   Color loss: $\mathcal{L}_{color} = \sum_{\mathbf{x} \in \mathcal{N}} \| \mathbf{C}(\mathbf{x}) - \hat{\mathbf{C}}_B(\mathbf{x}) \|_2^2$, where $\mathcal{N}$ is the set of training view pixels in a batch, $\mathbf{C}(\mathbf{x})$ is the observed blurry color, and $\hat{\mathbf{C}}_B(\mathbf{x})$ is the rendered blurry color [S3].
*   TV loss: $\mathcal{L}_{TV} = \sum_{i,k} \sqrt{\partial_x \mathcal{G}_\sigma(\mathbf{v}_{i,k})^2 + \partial_y \mathcal{G}_\sigma(\mathbf{v}_{i,k})^2 + \partial_z \mathcal{G}_\sigma(\mathbf{v}_{i,k})^2}$, where $\{\mathbf{v}_{i,k}\}_{i=1}^8$ are the 8 voxels containing sample point $k$, and derivatives are spatial gradients of density [S3].
*   Sparsity loss: $\mathcal{L}_{s} = \sum_{k} \log(1 + 2\sigma_k^2)$, where $\sigma_k$ is the volume density at sample point $k$. This sparsity prior leads to sharp density distribution [S3].
*   Combined loss: $\mathcal{L} = \mathcal{L}_{color} + \lambda_{TV}\mathcal{L}_{TV} + \lambda_s\mathcal{L}_s$ [S3].

**Parameters:**
*   $\lambda_{TV}$: Weight for total variation loss. Value: $5 \times 10^{-4}$ [S3].
*   $\lambda_s$: Weight for sparsity loss. Value: $1 \times 10^{-12}$ [S3].

### Multi-task Loss for Phys3DGS
Phys3DGS trains physically-based parameters (albedo, metalness, roughness) using a multi-task loss that includes PBR photometric loss, smoothness, and regularization terms [S4, unofficial].

**Step:**
*   PBR Training: Train PBR-related parameters (albedo/base color, metalness, roughness, indirect illumination SH coefficients) using the deferred rendering pipeline. Apply bilateral smoothness loss to all BRDF parameters. Clamp base color and metallic to [0, 1]. Scale and shift roughness to lie between 0.09 and 1 [S4, unofficial].

**Formula:**
*   $L_{total} = L_{L1} + L_{PBR} + \lambda_{smooth} L_{smooth} + \lambda_{o} L_{o} + \lambda_{sc} L_{sc} + \lambda_{sr} L_{sr}$, where $L_{L1}$ is volume rendering loss, $L_{PBR}$ is photometric loss of PBR, $L_{smooth}$ is bilateral smoothness loss, $L_o$ is masked opacity regularization, $L_{sc}$ is Gaussian scale regularization, $L_{sr}$ is surface regularization loss, and $\lambda$ are hyperparameters [S4, unofficial].

**Parameters:**
*   Roughness range: Scaled and shifted to lie between 0.09 and 1 to prevent exceedingly strong specular reflectance [S4, unofficial].

### APMGSRN Training Routine
APMGSRN uses a decoupled training routine to update feature grids and transformation matrices separately, employing delayed starts and early stopping for the latter [S5].

**Step:**
*   Each training step has two parts: (1) update feature grid and decoder parameters using reconstruction loss $L_{rec}$, (2) update transformation matrices using density loss $L_{density}$ [S5].
*   $L_{density}$ is the only loss providing gradients to update transformation matrices G. $L_{rec}$ is the only loss providing gradients to update feature grid values F and decoder weights [S5].
*   Step (1) happens every iteration. Step (2) is delayed to minimize updates and avoid negatively impacting reconstruction quality [S5].
*   Delayed Start: Freeze transformation matrices for the first 500 iterations. Update them according to $L_{density}$ after that, once the model has seen enough data to identify low- and high-error regions [S5].
*   Early Stopping for Transformation Matrices: Track $L_{density}$ each iteration. Set an early stopping flag when the 1000-iteration moving average of $L_{density}$ has not reduced by 0.01%. Alternatively, stop updating transformation matrices if this criteria is not met after 80% of total training steps [S5].

**Number:**
*   A single APMGSRN model takes at most 4 minutes to train for 50k iterations. Often, grids converge quickly, reducing training time to as short as 40 seconds [S5].

## Background
The implementations originate from papers addressing neural implicit surfaces (HybridNeRF), sparse voxel representations (Plenoxels/CF Loss), recursive octrees (RocNet), mobile Gaussian splatting (Mobile-GS), motion blur handling (ExBluRF), physically-based rendering (Phys3DGS), and adaptive grid networks (APMGSRN) [G]. These methods were developed to address specific challenges such as shape-radiance ambiguity, training efficiency on mobile devices, handling of motion blur, and the decoupling of geometric and appearance parameters [G].

## Sources
[S1] RocNet: Recursive Octree Network for Efficient 3D Deep Representation, https://doi.org/10.1109/3dv50981.2020.00051, fetched 2026-10-26
[S2] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
[S3] ExBluRF: Efficient Radiance Fields for Extreme Motion Blurred Images, https://arxiv.org/abs/2309.08957v3, fetched 2026-10-27
[S4] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
[S5] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
