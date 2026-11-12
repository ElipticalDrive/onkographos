# Total Variation Regularization

Also called: Efficient Total Variation (TV) Loss, TV Loss, Total Variation Loss

## Overview
Total Variation (TV) regularization is a loss term used in explicit radiance field representations (such as voxel grids) to suppress visual artifacts like "floaters" (spurious geometry) and "holes" (missing geometry) [S1, unofficial]. It encourages spatial smoothness by penalizing large gradients between neighboring voxels [S2, unofficial]. Unlike implicit MLP-based methods, explicit representations are more prone to these artifacts due to the non-convex nature of optimizing high-dimensional discrete grids [S1, unofficial]. TV regularization is distinct from sibling methods like Distortion Loss (which penalizes depth ambiguity along rays) or MSE/SSIM (which enforce pixel-level photometric consistency) by operating directly on the 3D spatial structure of the volume [S1, 1051-efficient-o-n-distortion-loss.md, 2890-l1-ssim-loss.md, unofficial].

## How it works
**Inputs:**
- A 3D tensor representing the radiance field parameters (e.g., opacity $\alpha$, color $c$, or SH coefficients) with shape $[X, Y, Z, C]$, where $C$ is the channel dimension [S2, unofficial].
- A weight scalar $\lambda_{tv}$ [S1, unofficial][S5].
- A boolean flag `dense_mode` indicating whether to compute gradients for all voxels or only those with non-zero gradients [S1, unofficial].

**Outputs:**
- A scalar loss value (if computing forward pass).
- Gradients added to the input tensor's gradient buffer (in the efficient implementation) [S1, unofficial].

**Data Structures:**
- The input tensor is typically stored in contiguous memory (e.g., PyTorch `Tensor` or CUDA array) [S1, S2, unofficial].
- For the efficient implementation, a custom CUDA kernel is used to fuse operations [S1, unofficial].

**Steps:**

1. **Gradient Computation:** Compute the spatial gradients along the three principal axes (x, y, z) by subtracting adjacent slices of the input tensor [S2, unofficial].
   Let $T$ be the input tensor of shape $[X, Y, Z, C]$.
   $$
   \begin{aligned}
   G_x &= T[:, :-1, :, :] - T[:, 1:, :, :] \\
   G_y &= T[:, :-1, :, :] - T[:, 1:, :, :] \quad \text{(Note: Source [S2, unofficial] uses same syntax for x/y/z, implying axis-specific slicing)} \\
   G_z &= T[:, :, :-1, :] - T[:, :, 1:, :]
   \end{aligned}
   $$
   *Correction based on standard 3D tensor indexing:* If $T$ is $[X, Y, Z, C]$, then:
   $$
   \begin{aligned}
   G_x &= T[:, :, :, :] - T[:, :, :, :] \quad \text{(shifted along X)} \\
   G_y &= T[:, :, :, :] - T[:, :, :, :] \quad \text{(shifted along Y)} \\
   G_z &= T[:, :, :, :] - T[:, :, :, :] \quad \text{(shifted along Z)}
   \end{aligned}
   $$
   Specifically, using Python-like slicing for a tensor $T$ of shape $(D_x, D_y, D_z, C)$:
   ```python
   # Gradient along X axis
   grad_x = T[:, :, :, :] - T[:, :, :, :] # T[:, 1:, :, :] - T[:, :-1, :, :]
   # Gradient along Y axis
   grad_y = T[:, :, :, :] - T[:, :, :, :] # T[:, :, 1:, :] - T[:, :, :-1, :]
   # Gradient along Z axis
   grad_z = T[:, :, :, :] - T[:, :, :, :] # T[:, :, :, 1:] - T[:, :, :, :-1]
   ```
   *Note: Source [S2, unofficial] explicitly states `gradient_x = input_tensor[:, :-1, :, :] - input_tensor[:, 1:, :, :]`. This implies the first dimension is X. We follow this convention.*

2. **Loss Calculation (Standard L2 TV):**
   Square the gradients and sum them across all axes and channels [S2, unofficial].
   $$
   L_{tv} = \sum_{x,y,z,c} \left( (G_x)^2 + (G_y)^2 + (G_z)^2 \right)
   $$
   Take the square root to get the L2 norm [S2, unofficial].
   $$
   L_{tv} = \sqrt{\sum_{x,y,z,c} \left( (G_x)^2 + (G_y)^2 + (G_z)^2 \right)}
   $$
   *Note: Source [S1, unofficial] mentions using Huber loss instead of L1/L2 for better results in explicit modeling. Huber loss $H_\delta(u)$ is defined as:*
   $$
   H_\delta(u) = \begin{cases} 
   \frac{1}{2}u^2 & \text{if } |u| \le \delta \\
   \delta(|u| - \frac{1}{2}\delta) & \text{otherwise}
   \end{cases}
   $$
   *The TV loss is the sum of Huber losses between each grid point and its six nearest neighbors [S1, unofficial].*

3. **Efficient Gradient Injection (CUDA Kernel):**
   To avoid PyTorch API overhead, fuse the gradient computation into a single CUDA kernel [S1, unofficial].
   - Skip the forward pass computation of the loss value if only gradients are needed for optimization [S1, unofficial].
   - Directly add the pre-computed gradients to the PyTorch tensor's gradient buffer [S1, unofficial].

4. **Optimization Step:**
   Call the custom API after the standard backward pass and before the optimizer step [S1, unofficial].
   ```python
   optimizer.zero_grad()
   # Compute total_loss (photometric, etc.)
   total_loss.backward()
   # Add TV gradients
   dvgo_model.total_variation_add_grad(
       tv_weight, dense_mode=(curr_step < 10000))
   optimizer.step()
   ```

**Parameters:**
- $\lambda_{tv}$: Weight for the TV loss. Typical value: 0.05 [S5].
- `dense_mode`: Boolean. If `True`, compute TV for all grid points. If `False`, compute only for grid points with non-zero gradients (sparse mode) [S1, unofficial].
- $\delta$: Huber loss threshold (if using Huber). Not given in the sources.

**Edge Cases and Pitfalls:**
- **Performance:** Computing TV loss densely for all grid points is time-consuming [S1, unofficial]. It is recommended to use `dense_mode=True` only for the first 10,000 iterations, then switch to `False` [S1, unofficial].
- **Precision:** When using log-space regularization (see Subtopics), small opacity values can lead to numerical instability [S3].
- **Boundaries:** The gradient computation reduces the effective volume size by 1 in each dimension. Boundary voxels are handled by the slicing operation (no padding is mentioned) [S2, unofficial].

## Subtopics

### [Plenoxels regularization ablation](901-plenoxels-regularization-ablation.md)
This variant documents the regularization strategies in the Plenoxels framework, combining TV regularization on voxel opacities and spherical harmonic coefficients with sparsity and beta losses [901-plenoxels-regularization-ablation.md]. It distinguishes itself by addressing floaters and holes in sparse voxel-based radiance fields through explicit spatial smoothing rather than geometric constraints.

### [Space-Time Tomographic Projection (STP) Regularizer](3265-space-time-tomographic-projection-stp-regularizer.md)
A regularization term for volume reconstruction that aggregates gradient information from a local neighborhood (patch) of voxels using a structure tensor, rather than just the gradient at a single voxel [3265-space-time-tomographic-projection-stp-regularizer.md]. It solves the problem of isotropic smoothing by allowing anisotropic regularization based on local structure.

### [Anisotropic Total Variation](3273-anisotropic-total-variation.md)
A regularization prior defined as the sum of absolute differences along the three principal axes (x, y, z) for each voxel [3273-anisotropic-total-variation.md]. It is represented as $\tilde{g}(K\mathbf{x})$ where $\tilde{g}(\cdot)$ is the $l_1$ norm and $K=D$ is the forward difference matrix.

### Total Variation Prior on Log Opacity
A regularization term that enforces sparse spatial gradients on the log of voxel opacities to recover sharp boundaries between opaque and transparent regions and reduce smoke-like artifacts [S3].
**Formula:**
$$
J_{\text{tv}}(\mathbf{V}_\alpha) = \frac{1}{N} \sum_{\mathbf{x}} \lambda_{\text{tv}} \left\| \frac{\partial}{\partial \mathbf{x}} \log \mathbf{V}_\alpha(\mathbf{x}) \right\|
$$
where $\mathbf{V}_\alpha(\mathbf{x})$ is the voxel opacity at center $\mathbf{x}$, $N$ is the number of voxels, and $\lambda_{\text{tv}}$ is the weight [S3].
**Implementation:** The derivative is discretized using the rectangle rule twice with samples on the right of each step interval and backwards differences [S3].
**Pitfall:** Artifacts tend to be mostly transparent; applying the prior in log space increases sensitivity to small $\alpha$ values [S3].

### Isotropic Total Variation
A regularization prior that sums the magnitude of the gradient at each voxel [S4, unofficial].
**Formula:**
$$
h(\mathbf{f}) = \sum_{i,j,k} \sqrt{|\mathbf{f}_{i+1,j,k} - \mathbf{f}_{i,j,k}|^2 + |\mathbf{f}_{i,j+1,k} - \mathbf{f}_{i,j,k}|^2 + |\mathbf{f}_{i,j,k+1} - \mathbf{f}_{i,j,k}|^2}
$$
The norm is defined as $\|\mathbf{w}\|_E = \|\sqrt{\mathbf{E}^T \mathbf{w}^2}\|_1 = \sum_v \|\mathbf{w}^v\|_2$, where $\mathbf{w}^v$ is the gradient at voxel $v=(i,j,k)$ [S4, unofficial]. The prior is expressed as $h(\mathbf{x}) = \|\mathbf{Df}\|_E = \tilde{g}(\mathbf{Df})$ [S4, unofficial].
**Proximal Operator:**
$$
\text{prox}_{\mu \tilde{g}^*}(\mathbf{q}) = \Pi_{B^*}(\mathbf{q}) = \frac{\mathbf{q}}{\mathbf{E}_{\max}\left(\sqrt{\mathbf{E}^T \mathbf{q}^2}, 1\right)}
$$
where division and max are component-wise [S4, unofficial].

### R2-Gaussian Optimization
Optimization of radiative Gaussians using stochastic gradient descent with photometric losses and 3D total variation regularization [S5].
**Steps:**
1. Randomly query a tiny density volume $V_{tv} \in R^{D \times D \times D}$ (same spacing as target output) and minimize its total variation [S5].
2. Remove empty Gaussians and densify (clone or split) those with large loss gradients [S5].
3. Do not prune large Gaussians to handle homogeneous areas like human organs [S5].
4. When densifying, halve the densities of both original and replicated Gaussians to mitigate sudden performance drops [S5].
**Formula:**
$$
L_{total} = L_1(I_r, I_m) + \lambda_{ssim} L_{ssim}(I_r, I_m) + \lambda_{tv} L_{tv}(V_{tv})
$$
where $L_1$ is photometric loss, $L_{ssim}$ is D-SSIM loss, and $L_{tv}$ is 3D total variation regularization [S5].
**Parameters:** $\lambda_{ssim} = 0.25$, $\lambda_{tv} = 0.05$ [S5]. Learning rates: Position 0.0002, Density 0.01, Scale 0.005, Rotation 0.001, exponentially decaying to 0.1 of initial values [S5].

## Operating characteristics
- **Time Complexity:** Computing TV loss densely is $O(N)$ where $N$ is the number of voxels, but the constant factor is high due to memory access patterns and kernel launch overheads [S1, unofficial]. Using the efficient CUDA kernel and sparse mode (after 10k iterations) reduces this significantly [S1, unofficial].
- **Memory Complexity:** $O(N)$ for storing the input tensor and gradients [S2, unofficial].
- **Measured Performance:** In R2-Gaussian, TV regularization helps suppress artifacts in tomographic reconstruction [S5]. In Plenoxels, it stabilizes optimization of sparse voxel grids [S1, unofficial].
- **Strengths:** Effective at removing floaters and holes in explicit representations [S1, unofficial]. Encourages spatial smoothness [S2, unofficial].
- **Limitations:** Computationally expensive if computed densely for all iterations [S1, unofficial]. May oversmooth fine details if the weight is too high [S1, unofficial].
- **When to use:** When using explicit radiance field representations (voxels, Gaussians) prone to artifacts [S1, unofficial]. When training tomographic reconstruction models [S5].
- **When not to use:** When using implicit MLPs (where TV is less common or different) [S1, unofficial]. When computational resources are extremely limited and dense TV is required for convergence [S1, unofficial].

## Background
Total Variation regularization originated in image processing (Rudin-Osher-Fatemi model) and has been adapted for volume rendering and tomographic reconstruction [S1, S4, unofficial]. In the context of radiance fields, it was introduced to address the specific artifacts (floaters, holes) arising from explicit representations [S1, unofficial]. The "Efficient TV Loss" variant was proposed to optimize the training time of these explicit models by fusing CUDA kernels and skipping forward passes [S1, unofficial]. The log-opacity variant was introduced to handle transparent artifacts in dynamic volumes [S3].

## Sources
[S1] Improved Direct Voxel Grid Optimization for Radiance Fields Reconstruction, https://doi.org/10.48550/arxiv.2206.05085, fetched 2026-10-26 (unofficial)
[S2] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S3] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
[S4] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
[S5] R$^2$-Gaussian: Rectifying Radiative Gaussian Splatting for Tomographic Reconstruction, https://arxiv.org/abs/2405.20693v2, fetched 2026-10-26
