# Viewing Direction Gaussian Splatting


## Overview
Viewing Direction Gaussian Splatting (VDGS) is a hybrid neural rendering method that combines the explicit geometric representation of 3D Gaussian Splatting (GS) with the view-dependent appearance modeling of Neural Radiance Fields (NeRF) [S1]. It solves the limitation of classical GS, where color and opacity are static per primitive, by using a neural network to modulate opacity (and optionally color) based on the viewing direction, thereby accurately rendering specular reflections, transparency, and glass without increasing training time significantly [S1]. Unlike classical GS, VDGS achieves NeRF-quality view-dependent effects while maintaining the fast training and inference speeds of Gaussian rasterization [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of camera poses and corresponding 2D images from a dataset; initial 3D points (from Structure from Motion or random initialization).
*   **Outputs:** A differentiable rendering of the scene for arbitrary camera viewpoints, represented by a set of 3D Gaussians and a neural network.

**Data Structures:**
1.  **Gaussian Primitives ($G$):** A collection of $n$ 3D Gaussian distributions. Each Gaussian $i$ is defined by:
    *   Mean position $\mathbf{m}_i \in \mathbb{R}^3$ (trainable).
    *   Covariance matrix $\mathbf{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (trainable, positive definite).
    *   Base color $\mathbf{c}_i \in \mathbb{R}^3$ (RGB, trainable).
    *   Base opacity $\sigma_i \in \mathbb{R}$ (trainable, scalar).
2.  **Neural Network ($F_{VDGS}$):** A Multi-Layer Perceptron (MLP) with parameters $\Theta$.
    *   **Input:** Hash-encoded Gaussian mean $\mathbf{m}_i$ and viewing direction $\mathbf{d} \in \mathbb{R}^2$ (spherical coordinates $\theta, \psi$).
    *   **Output:** Opacity update factor $\Delta \sigma(\mathbf{d}) \in \mathbb{R}$.
    *   **Hash Grid:** 12 levels of 3D embedding (resolutions 16 to 512), hash table size $2^{14}$ (16384), feature dimension 2, LeakyReLU activation [S1].
    *   **MLP Architecture:** 2-layer MLP [S1].

**Formulas:**
The vanilla NeRF model is defined as $F_{NeRF}(\mathbf{x}, \mathbf{d}; \Theta) = (\mathbf{c}, \sigma)$ [S1]. VDGS modifies this by separating shape and view-dependent appearance.

The VDGS function is defined as:
$$ F_{VDGS}(\mathbf{m}, \mathbf{d}; \Theta) = \Delta \sigma(\mathbf{d}) $$
where $\mathbf{m}$ is the Gaussian mean and $\mathbf{d}$ is the viewing direction [S1].

The final representation is a set of tuples:
$$ (G, F_{VDGS}) = \{ (\mathcal{N}(\mathbf{m}_i, \mathbf{\Sigma}_i), \sigma_i \cdot \Delta \sigma_i(\mathbf{d}), \mathbf{c}_i) \}_{i=1}^n $$
where $\Delta \sigma_i(\mathbf{d}) = F_{VDGS}(\mathbf{m}_i, \mathbf{d}; \Theta)$ [S1].

Rendering computes the pixel color $C(\mathbf{p})$ by blending $N$ ordered overlapping Gaussian samples along the ray passing through pixel $\mathbf{p}$:
$$ C = \sum_{i \in \mathcal{N}} \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
where $\alpha_i$ is the effective opacity of sample $i$:
$$ \alpha_i = 1 - \exp\left( \sigma_i \cdot F_{VDGS}(\mathbf{m}_i, \mathbf{d}; \Theta) \cdot \delta_i \right) $$
Here, $\delta_i$ is the distance between adjacent samples along the ray, and $\sigma_i$ is the base opacity of the Gaussian [S1].

**Parameters:**
*   Hash Grid Levels: 12 [S1].
*   Hash Table Size: $2^{14}$ (16384) [S1].
*   Feature Dimension: 2 [S1].
*   Activation Function: LeakyReLU [S1].
*   MLP Layers: 2 [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize the model with Structure from Motion (SfM) points (from COLMAP or randomly generated) to establish initial 3D Gaussian positions and covariances [S1].
2.  **Hash Encoding:** Encode the mean position $\mathbf{m}_i$ of each Gaussian using a 3D structured hash grid [S1].
3.  **View-Dependent Update:** Feed the hashed Gaussian mean $\mathbf{m}_i$ and the viewing direction $\mathbf{d}$ into the MLP $F_{VDGS}$ to compute the opacity update $\Delta \sigma_i(\mathbf{d})$ [S1].
4.  **Effective Opacity Calculation:** Calculate the effective opacity $\alpha_i$ for each Gaussian sample using the base opacity $\sigma_i$, the NeRF-based density update $\Delta \sigma_i(\mathbf{d})$, and the sample distance $\delta_i$ [S1].
5.  **Rasterization:** Render the pixel color $C(\mathbf{p})$ by blending $N$ ordered overlapping Gaussian samples using alpha compositing [S1].
6.  **Optimization:** Jointly optimize all parameters (Gaussian mean, covariance, color, base opacity, and MLP weights $\Theta$) using differential Gaussian rasterization [S1].
7.  **Pruning and Densification:** During optimization, eliminate Gaussians with opacity below a specific limit every hundred iterations. Generate new Gaussians in unoccupied areas of 3D space to fill vacant areas [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class HashGrid:
    """
    Simplified hash grid encoder.
    In practice, this uses a CUDA kernel for efficient lookup.
    """
    def __init__(self, num_levels=12, resolution_range=(16, 512), hash_size=2**14, feature_dim=2):
        self.num_levels = num_levels
        self.hash_size = hash_size
        self.feature_dim = feature_dim
        # Initialize hash table with random features
        self.table = torch.randn(hash_size, num_levels, feature_dim)

    def forward(self, x):
        # x: (N, 3) coordinates
        # Returns: (N, num_levels * feature_dim)
        # Note: Actual implementation requires CUDA for hashing logic.
        # This is a placeholder for the mathematical operation.
        # For implementation, one would use a library like tiny-cuda-nn.
        pass

class VDGS_MLP(nn.Module):
    def __init__(self, input_dim, hidden_dim=64, output_dim=1):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.LeakyReLU(),
            nn.Linear(hidden_dim, output_dim)
        )

    def forward(self, x):
        return self.net(x)

class VDGS:
    def __init__(self, num_gaussians, hash_grid, mlp):
        self.num_gaussians = num_gaussians
        self.hash_grid = hash_grid
        self.mlp = mlp
        # Gaussian parameters
        self.means = nn.Parameter(torch.randn(num_gaussians, 3))
        self.covariances = nn.Parameter(torch.eye(3).unsqueeze(0).repeat(num_gaussians, 1, 1))
        self.colors = nn.Parameter(torch.randn(num_gaussians, 3))
        self.base_opacities = nn.Parameter(torch.ones(num_gaussians))

    def get_view_dependent_opacity(self, means, view_directions):
        """
        Compute the opacity update factor Delta_sigma based on viewing direction.
        means: (N, 3)
        view_directions: (N, 2) spherical coords (theta, psi)
        """
        # Hash encode means
        hashed_means = self.hash_grid(means) # (N, 12 * 2)
        
        # Concatenate with view direction
        # Note: The paper says "camera position and Gaussian center... fed into MLP".
        # However, the formula F_VDGS(m, d) suggests mean and direction.
        # We assume the MLP takes hashed mean and view direction.
        # The input dimension to MLP is hash_dim + view_dim
        input_tensor = torch.cat([hashed_means, view_directions], dim=-1)
        
        # MLP outputs Delta_sigma
        delta_sigma = self.mlp(input_tensor) # (N, 1)
        return delta_sigma

    def render(self, ray_origins, ray_directions, image_width, image_height):
        """
        Render a single image.
        This is a simplified conceptual implementation.
        Real implementation uses differential Gaussian rasterization.
        """
        # 1. Project Gaussians to 2D and sort by depth
        # 2. For each pixel, gather overlapping Gaussians
        # 3. Compute alpha and color
        
        # Placeholder for rasterization logic
        # In practice, this calls a CUDA kernel
        pass

    def step(self, batch):
        """
        One optimization step.
        """
        # 1. Get camera parameters and image
        # 2. Render predicted image
        pred_image = self.render(...)
        
        # 3. Compute loss against ground truth
        loss = (pred_image - batch.gt_image).mean()
        
        # 4. Backpropagate
        loss.backward()
        
        # 5. Prune and densify Gaussians every 100 iterations
        # (Implementation details omitted as they are heuristic-based)
        pass
```

**Edge Cases and Pitfalls:**
*   **Joint Color and Opacity Update:** The authors found that updating both color and opacity via the MLP yields slightly worse results than updating only opacity. This is attributed to the dependency of opacity on color, making joint training ineffective [S1].
*   **Opacity vs. Color Multiply:** "VDGS Opacity Multiply" (updating only opacity) yields the best scores on real scenes, while "VDGS Color Multiply" yields the best scores on the NeRF Synthetic dataset [S1]. The final model uses Opacity Multiply [S1].
*   **Precision:** The hash grid provides extremely fast queries, but precision depends on the hash table size and resolution levels [S1].

## Subtopics

### Differentiable Primitive Assembly
Uses set-theoretic operations for rendering. Chooses this when explicit geometry is not required or when dealing with non-Gaussian primitives. [Differentiable Primitive Assembly](13657-differentiable-primitive-assembly.md)

### Point-NeRF Representation
Uses point clouds with MLPs for local queries. Chooses this when memory efficiency for sparse scenes is prioritized over the dense rasterization speed of Gaussians. [Point-NeRF Representation](15538-point-nerf-representation.md)

### Frequency-Modulated Point Cloud Rendering
Uses frequency modulation on point clouds. Chooses this for high-frequency detail reconstruction in point-based methods. [Frequency-Modulated Point Cloud Rendering](15903-frequency-modulated-point-cloud-rendering.md)

## Operating characteristics
*   **Time Complexity:** Training time is comparable to classical Gaussian Splatting, significantly faster than NeRF. Inference is also fast, comparable to GS [S1].
*   **Memory Complexity:** Depends on the number of Gaussians and the hash grid size. The hash grid is fixed size ($2^{14}$ entries), while Gaussians grow during densification [S1].
*   **Measured Performance (RTX 3090, VDGS-30K*):**
    *   **Mip-NeRF360:** SSIM 0.813, PSNR 27.64, LPIPS 0.220. Training: 46m 11s. Inference: 41.35 FPS [S1].
    *   **Tanks and Temples:** SSIM 0.851, PSNR 24.02, LPIPS 0.176. Training: 29m 45s. Inference: 38.53 FPS [S1].
    *   **Deep Blending:** SSIM 0.906, PSNR 29.54, LPIPS 0.243. Training: 41m 29s. Inference: 44.72 FPS [S1].
    *   **Shiny Blender:** Avg PSNR 30.62 [S1].
*   **Strengths:** Accurately models light reflections, transparency, and glass; eliminates view-dependent artifacts; fast training and inference compared to NeRF [S1].
*   **Limitations:** Requires careful tuning of pruning/densification; joint color/opacity update is less effective than opacity-only update [S1].
*   **When to use:** When view-dependent effects (shiny surfaces, transparency) are critical, but NeRF training times are prohibitive.
*   **When not to use:** When the scene is purely diffuse and static, classical GS may be sufficient and simpler.

## Background
VDGS was proposed by researchers from the Faculty of Mathematics and Computer Science, Jagiellonian University, and the Department of Engineering, University of Cambridge [S1]. It was published in December 2023 (arXiv:2312.13729) [S1]. The method addresses the limitations of both NeRF (slow, hard to locate geometry) and GS (hard to condition, static appearance) by hybridizing them [S1]. It builds on the differential Gaussian rasterization pipeline introduced in classical GS [S1].

## Sources
[S1] Gaussian Splatting with NeRF-based Color and Opacity, https://doi.org/10.48550/arxiv.2312.13729, fetched 2026-10-29
