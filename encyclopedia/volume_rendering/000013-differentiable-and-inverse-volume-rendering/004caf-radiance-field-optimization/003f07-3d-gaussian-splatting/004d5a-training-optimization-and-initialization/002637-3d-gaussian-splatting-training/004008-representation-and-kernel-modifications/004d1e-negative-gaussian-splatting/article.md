# Negative Gaussian Splatting

Also called: NegGS, Diff-Gaussian

## Overview
Negative Gaussian Splatting (NegGS) extends the 3D Gaussian Splatting framework by incorporating "negative Gaussians"—primitives with negative color values—into the scene representation [S1, S2]. This modification allows the model to represent high-frequency elements, rapid color transitions, and complex nonlinear structures (such as shadows and donut-shaped densities) that are difficult to capture with standard positive-only Gaussians [S1, S2]. Unlike standard Gaussian Splatting, which requires many components to approximate complex shapes, NegGS uses the cancellation effect between positive and negative primitives to achieve greater flexibility with fewer total components [S1, S2].

## How it works
NegGS operates within the differentiable rendering framework of 3D Gaussian Splatting [Parent](10145-representation-and-kernel-modifications.md). It modifies the attribute space of the Gaussian primitives to allow negative color coefficients, effectively creating a "Diff-Gaussian" distribution defined by the ratio of two Gaussian probability density functions (PDFs) [S1, S2].

### Inputs and Outputs
*   **Inputs:** A set of camera poses, input images, and an initial point cloud (e.g., from COLMAP).
*   **Outputs:** A scene representation consisting of a joint family of Gaussians $G_{NeGS}$ and optimized parameters (position, covariance, opacity, and color) for each Gaussian.

### Data Structures
The scene is represented as a joint family of tuples:
$$ G_{NeGS} = G_{GS} \cup G_{-GS} $$
where:
*   $G_{GS}$ contains tuples with positive color values $c_i$.
*   $G_{-GS}$ contains tuples with negative color values $c_i$ [S1].

Each Gaussian primitive $i$ retains the standard 3D Gaussian Splatting attributes:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$.
*   Covariance matrix $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$, parameterized by rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$.
*   Opacity $\alpha_i \in [0, 1]$.
*   Color $\mathbf{c}_i$, which can be positive or negative. In implementation, this is tracked via a binary mask to distinguish between $G_{GS}$ and $G_{-GS}$ [S1].

### Rendering and Optimization
The rendering process follows the standard differentiable rasterization pipeline of 3D Gaussian Splatting [Parent](10145-representation-and-kernel-modifications.md). The key distinction is that the color contribution of primitives in $G_{-GS}$ is subtracted during alpha blending. This allows two Gaussians with identical spatial parameters but inverse colors to cancel each other out, providing greater flexibility in shape representation and color modulation [S1].

The optimization strategy involves:
1.  **Initialization:** The initial number of negative Gaussians is a critical hyperparameter. It is typically set to approximately 20% of the total Gaussians [S1].
2.  **Dynamic Adaptation:** During training, Gaussians with low opacity are pruned (reduced), and new Gaussians are introduced into empty 3D regions to dynamically adapt the number of components [S1].
3.  **Gradient Descent:** Standard gradient-based optimization is used to minimize the difference between rendered and ground truth images.

### Mathematical Formulation: Diff-Gaussian
The theoretical basis for NegGS is the "Diff-Gaussian" distribution, formed by dividing the PDFs of two Gaussians $f_{m_0, \Sigma_0}$ and $f_{m_1, \Sigma_1}$ [S1, S2]. The optimal coefficient $c_{opt}$ for this distribution is defined as the minimum ratio of the two densities:

$$ c_{opt} := \min_{x} \frac{f_{m_0, \Sigma_0}(x)}{f_{m_1, \Sigma_1}(x)} = \frac{f_{m_0, \Sigma_0}(w)}{f_{m_1, \Sigma_1}(w)} $$

if and only if $w$ satisfies [S1]:

$$ \Sigma_1^{-1}(w - m_1) = \Sigma_0^{-1}(w - m_0) $$

This condition arises from minimizing the quadratic function $q(x) = x^T A x + 2b^T x$, where [S1]:
*   $A = \Sigma_1^{-1} - \Sigma_0^{-1}$ (Note: Source [S1] text states $A = \Sigma_1^{-1} - \Sigma_0$ in one place and derives $A = \Sigma_1^{-1} - \Sigma_0^{-1}$ in the proof steps; the proof derivation $x^T (\Sigma_1^{-1} - \Sigma_0^{-1})x$ is mathematically consistent with the log-ratio minimization. We follow the proof derivation: $A = \Sigma_1^{-1} - \Sigma_0^{-1}$).
*   $b = \Sigma_0^{-1} m_0 - \Sigma_1^{-1} m_1$.

For diagonal covariance matrices $\Sigma_0 = \text{diag}(\sigma_1^2, \dots, \sigma_n^2)$ and $\Sigma_1 = \text{diag}(h_1 \sigma_1^2, \dots, h_n \sigma_n^2)$ with $0 < h_i \le 1$, the coordinates $w_i$ of the null density vector (where the ratio is minimized) are [S1]:
1.  $w_i = \frac{m_{1,i} - h_i m_{0,i}}{1 - h_i}$ if $h_i < 1$.
2.  $w_i \in \mathbb{R}$ if $h_i = 1$ and $m_{1,i} = m_{0,i}$.

This allows the imposition of a null density on a $k$-dimensional affine space, enabling the approximation of complex nonlinear structures [S1].

### Pseudocode
The following pseudocode outlines the initialization and rendering logic for NegGS, assuming a standard 3D Gaussian Splatting rasterizer is available.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class NegativeGaussianSplatting:
    def __init__(self, initial_poses, initial_points, num_gaussians, neg_ratio=0.2):
        """
        Initialize NegGS scene.
        :param initial_poses: Camera poses.
        :param initial_points: Initial 3D points (e.g., from COLMAP).
        :param num_gaussians: Total number of Gaussians to initialize.
        :param neg_ratio: Fraction of Gaussians to be negative (default 0.2).
        """
        self.num_gaussians = num_gaussians
        self.neg_count = int(num_gaussians * neg_ratio)
        self.pos_count = num_gaussians - self.neg_count
        
        # Initialize positions and scales from input points
        self.positions = initial_points[:num_gaussians]
        self.scales = initialize_scales(initial_points[:num_gaussians])
        self.rotations = initialize_rotations(num_gaussians)
        self.opacities = initialize_opacities(num_gaussians)
        
        # Initialize colors. First pos_count are positive, rest are negative.
        # Colors are typically initialized to white (1,1,1) or mean image color.
        self.colors = initialize_colors(num_gaussians)
        
        # Binary mask to track positive (0) vs negative (1) Gaussians
        # 0 for G_GS, 1 for G_-GS
        self.is_negative = torch.zeros(num_gaussians, dtype=torch.bool)
        self.is_negative[self.pos_count:] = True
        
        # Apply negative sign to colors for negative Gaussians
        self.colors[self.is_negative] = -self.colors[self.is_negative]

    def render(self, camera, image_width, image_height):
        """
        Render the scene using differentiable rasterization.
        :param camera: Camera parameters (view/projection matrices).
        :return: Rendered image tensor.
        """
        # Project 3D Gaussians to 2D ellipses
        # This step is identical to standard 3D Gaussian Splatting [Parent]
        projected_gaussians = project_gaussians(
            self.positions, 
            self.scales, 
            self.rotations, 
            camera
        )
        
        # Sort Gaussians by depth for correct alpha blending
        sorted_indices = sort_by_depth(projected_gaussians)
        
        # Initialize output image
        image = torch.zeros((image_height, image_width, 3))
        
        # Rasterize and blend
        # The key difference: color contribution can be negative
        for idx in sorted_indices:
            g = projected_gaussians[idx]
            color = self.colors[idx] # This may be negative
            
            # Standard alpha blending step
            # alpha = g.opacity * kernel_evaluation(g)
            # image += alpha * color * (1 - accumulated_alpha)
            image = alpha_blend(image, g, color)
            
        return image

    def optimize_step(self, ground_truth_image, camera, optimizer):
        """
        Perform one optimization step.
        """
        optimizer.zero_grad()
        
        # Render current scene
        rendered_image = self.render(camera)
        
        # Compute loss (e.g., L1 + SSIM)
        loss = compute_loss(rendered_image, ground_truth_image)
        
        # Backpropagate
        loss.backward()
        
        # Update parameters
        optimizer.step()
        
        # Dynamic adaptation: Prune low opacity, add new Gaussians
        self.prune_and_refine()

    def prune_and_refine(self):
        """
        Adapt the number of Gaussians dynamically.
        """
        # Prune Gaussians with low opacity
        # (Standard GS pruning logic)
        mask = self.opacities > threshold
        self.positions = self.positions[mask]
        self.scales = self.scales[mask]
        self.rotations = self.rotations[mask]
        self.opacities = self.opacities[mask]
        self.colors = self.colors[mask]
        self.is_negative = self.is_negative[mask]
        
        # Add new Gaussians to empty regions
        # (Standard GS densification logic)
        new_gaussians = densify_scene(self.positions, self.opacities, camera)
        if new_gaussians is not None:
            self.positions = torch.cat([self.positions, new_gaussians.positions])
            self.scales = torch.cat([self.scales, new_gaussians.scales])
            self.rotations = torch.cat([self.rotations, new_gaussians.rotations])
            self.opacities = torch.cat([self.opacities, new_gaussians.opacities])
            
            # Assign negative status to new Gaussians based on ratio
            # Note: The ratio is maintained at initialization; 
            # dynamic adjustment of the ratio during training is not explicitly detailed 
            # beyond "numbers ... are treated as hyperparameters" [S1].
            # We assume new Gaussians are added as positive by default 
            # or follow the initial distribution strategy [G].
            new_is_negative = torch.zeros(len(new_gaussians), dtype=torch.bool)
            self.is_negative = torch.cat([self.is_negative, new_is_negative])
            
            # Update colors (positive by default)
            new_colors = initialize_colors(len(new_gaussians))
            self.colors = torch.cat([self.colors, new_colors])
```

### Edge Cases and Pitfalls
*   **Initialization Ratio:** The initial number of negative Gaussians is critical. Too few may fail to capture high-frequency details; too many may destabilize optimization. The optimal value is typically ~20% [S1].
*   **Cancellation Artifacts:** If positive and negative Gaussians have identical parameters and inverse colors, they cancel out. While this is useful for shaping, improper initialization can lead to degenerate zero-contributions that waste computational resources [S1].
*   **Precision:** The subtraction of color values requires careful handling of floating-point precision to avoid artifacts in dark regions or shadows [S2].

## Operating characteristics
*   **Time Complexity:** NegGS leverages the original Gaussian Splatting framework implementation, preserving the original GS rendering speed [S1]. The addition of negative colors does not significantly increase computational overhead compared to standard GS.
*   **Memory Complexity:** Memory usage is comparable to standard GS, as the data structures are identical except for the sign of the color attribute [S1].
*   **Performance:**
    *   On the Mip-NeRF360 dataset, NegGS-30K achieved an average PSNR of 27.39, SSIM of 0.812, and LPIPS of 0.219, compared to GS-30K's 27.21 PSNR, 0.815 SSIM, and 0.214 LPIPS [S1].
    *   On the NeRF Synthetic dataset, NegGS showed slight improvements or parity with GS, with NegGS requiring fewer total Gaussians to achieve comparable results [S1].
*   **Strengths:**
    *   Better modeling of high-frequency elements with rapid color transitions [S1, S2].
    *   Improved representation of shadows [S2].
    *   Ability to approximate complex nonlinear structures (e.g., donut shapes) with fewer components than standard GS [S1, S2].
*   **Limitations:**
    *   Requires careful tuning of the initial ratio of negative Gaussians [S1].
    *   May introduce artifacts if the cancellation logic is not properly managed during optimization [G].
*   **When to Use:** Use NegGS when standard Gaussian Splatting fails to capture high-frequency details, sharp color transitions, or complex nonlinear geometries (like thin structures or shadows) without excessive memory usage.
*   **When Not to Use:** For simple scenes with smooth gradients and low-frequency content, standard GS is sufficient and simpler to configure.

## Background
Negative Gaussian Splatting (NegGS) was introduced as the first method to extend the simple ellipsoid shapes of Gaussian Splatting to more complex nonlinear structures [S2]. It is based on the concept of "Diff-Gaussians," which are distributions formed by dividing the PDFs of two Gaussians [S2]. The method was evaluated on several datasets, including NeRF Synthetic, Mip-NeRF360, Tanks and Temples, Deep Blending, and Shiny Blender [S1]. It was shown to be effective on a single NVIDIA RTX 4090 GPU [S1].

## Sources
[S1] NegGS: Negative Gaussian Splatting, https://doi.org/10.1016/j.ins.2025.121912, fetched 2026-10-26
[S2] NegGS: Negative Gaussian Splatting, https://core.ac.uk/download/630026911.pdf, fetched 2026-10-28
