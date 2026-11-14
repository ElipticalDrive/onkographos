# Viewing Direction Gaussian Splatting

Also called: VDGS

## Overview
Viewing Direction Gaussian Splatting (VDGS) is a hybrid rendering method that combines the geometric efficiency of 3D Gaussian Splatting (GS) with the view-dependent appearance modeling of Neural Radiance Fields (NeRF) [S1]. It solves the limitation of standard GS, where color and opacity are static per Gaussian, by using a neural network to modulate opacity (and optionally color) based on the viewing direction, thereby accurately rendering specular reflections, transparency, and glass without increasing training time significantly [S1]. Unlike classical GS, VDGS allows for view-dependent artifacts correction and better background regulation while maintaining the fast rasterization pipeline of GS [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images with known camera poses; initial 3D points from Structure from Motion (SfM) or random initialization.
*   **Outputs:** A rendered image $C(p)$ for a given pixel $p$ and camera ray direction $d$.

**Data Structures:**
The scene is represented by a set of $N$ 3D Gaussians. Each Gaussian $i$ is defined by:
*   Mean position $\mathbf{m}_i \in \mathbb{R}^3$.
*   Covariance matrix $\mathbf{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (parameterized as a 6-element vector for rotation and scaling).
*   Base color $\mathbf{c}_i \in \mathbb{R}^3$ (RGB).
*   Base opacity $\sigma_i \in \mathbb{R}$ (scalar).
*   Viewing direction update $\Delta \sigma_i(\mathbf{d}) \in \mathbb{R}$ (scalar, output of the MLP).

The neural network component uses a 3D hash grid for position encoding:
*   Hash Grid Levels: 12 levels.
*   Hash Table Size: $2^{14}$ (16,384).
*   Feature Dimension: 2.
*   Activation Function: LeakyReLU.
*   MLP Architecture: 2-layer MLP.

**Numbered Steps:**

1.  **Initialization:** Initialize the model with $N$ 3D Gaussians. The positions $\mathbf{m}_i$ are derived from Structure from Motion (SfM) points (e.g., from COLMAP) or generated randomly. The covariance matrices $\mathbf{\Sigma}_i$ are initialized to identity or small values. Colors $\mathbf{c}_i$ and base opacities $\sigma_i$ are initialized from the input images or randomly [S1].

2.  **Hash Encoding:** For each Gaussian $i$, encode its mean position $\mathbf{m}_i$ using a 3D structured hash grid. This maps the continuous 3D coordinate to a discrete set of feature vectors in the hash table [S1].

3.  **Neural Update Calculation:** Feed the hashed position features and the viewing direction $\mathbf{d}$ (derived from the camera ray passing through the pixel) into the MLP network $F_{VDGS}$. The network outputs a volume density update $\Delta \sigma_i(\mathbf{d})$ [S1]:
    $$
    \Delta \sigma_i(\mathbf{d}) = F_{VDGS}(\text{Hash}(\mathbf{m}_i), \mathbf{d}; \Theta)
    $$
    where $\Theta$ are the MLP parameters [S1].

4.  **Effective Opacity Calculation:** Compute the effective opacity $\alpha_i$ for the $i$-th Gaussian sample along the ray. This combines the base opacity $\sigma_i$, the distance between adjacent samples $\delta_i$, and the neural update $\Delta \sigma_i(\mathbf{d})$ [S1]:
    $$
    \alpha_i = 1 - \exp\left( - \sigma_i \cdot \Delta \sigma_i(\mathbf{d}) \cdot \delta_i \right)
    $$
    Note: The source defines $\alpha_i$ using $\sigma_i \cdot F_{VDGS} \cdot \delta_i$. In standard volume rendering, $\sigma$ is density. Here, $\sigma_i$ acts as a base density/opacity factor, and $F_{VDGS}$ modulates it. The term $\sigma_i \cdot \Delta \sigma_i(\mathbf{d})$ effectively represents the view-dependent density [S1].

5.  **Rasterization:** Render the pixel color $C(p)$ by blending $N$ ordered overlapping Gaussian samples using alpha compositing. The Gaussians are sorted along the ray. The final color is calculated as [S1]:
    $$
    C(p) = \sum_{i \in \mathcal{N}} \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\mathcal{N}$ is the set of Gaussians intersecting the pixel, sorted by depth [S1].

6.  **Optimization:** Jointly optimize all parameters: Gaussian positions ($\mathbf{m}_i$), covariance matrices ($\mathbf{\Sigma}_i$), colors ($\mathbf{c}_i$), base opacities ($\sigma_i$), and MLP weights ($\Theta$) using differential Gaussian rasterization [S1].

7.  **Pruning and Densification:** During optimization, eliminate Gaussians with opacity below a specific limit every 100 iterations. Generate new Gaussians in unoccupied areas of 3D space to fill vacant areas [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class VDGS:
    def __init__(self, num_gaussians, hash_table_size=2**14, hash_levels=12, feature_dim=2):
        """
        Initialize VDGS model.
        :param num_gaussians: Number of 3D Gaussians.
        :param hash_table_size: Size of the hash grid.
        :param hash_levels: Number of hash levels.
        :param feature_dim: Feature dimension per hash level.
        """
        self.num_gaussians = num_gaussians
        
        # Gaussian parameters
        # Means: [N, 3]
        self.means = torch.randn(num_gaussians, 3) * 0.1
        # Covariances: [N, 6] (parameterized as scale and rotation)
        self.covariances = torch.ones(num_gaussians, 6)
        # Colors: [N, 3]
        self.colors = torch.rand(num_gaussians, 3)
        # Base Opacities: [N, 1]
        self.base_opacities = torch.ones(num_gaussians, 1)
        
        # Neural Network for View-Dependent Update
        # Hash Grid (simplified representation for pseudocode)
        self.hash_grid = nn.Embedding(hash_table_size * hash_levels, feature_dim)
        # MLP: Hash Features + View Direction -> Density Update
        # Input dim: feature_dim * hash_levels + 3 (view direction)
        # Output dim: 1 (density update)
        mlp_input_dim = feature_dim * hash_levels + 3
        self.mlp = nn.Sequential(
            nn.Linear(mlp_input_dim, 64),
            nn.LeakyReLU(),
            nn.Linear(64, 1)
        )
        
        # Optimizer
        self.optimizer = torch.optim.Adam(
            list(self.parameters()) + 
            [{'params': self.means, 'lr': 0.0001}, 
             {'params': self.covariances, 'lr': 0.0001},
             {'params': self.colors, 'lr': 0.0001},
             {'params': self.base_opacities, 'lr': 0.0001}]
        )

    def get_hash_features(self, means):
        """
        Encode 3D means using hash grid.
        :param means: [N, 3]
        :return: [N, feature_dim * hash_levels]
        """
        # In a real implementation, this involves hashing the coordinates to indices
        # and gathering features from the hash grid.
        # Here we simulate it with a placeholder function.
        # Note: The actual implementation uses a structured hash grid with 12 levels.
        # For pseudocode, we assume a function `hash_encode` exists that maps
        # continuous coordinates to discrete indices and gathers features.
        indices = hash_encode(means, levels=12, table_size=2**14)
        features = self.hash_grid(indices)
        return features.view(self.num_gaussians, -1)

    def forward(self, means, view_directions, covariances, colors, base_opacities):
        """
        Render a batch of rays.
        :param means: [N, 3]
        :param view_directions: [N, 3]
        :param covariances: [N, 6]
        :param colors: [N, 3]
        :param base_opacities: [N, 1]
        :return: Rendered colors [N, 3]
        """
        # Step 2: Hash Encoding
        hash_features = self.get_hash_features(means)
        
        # Step 3: Neural Update Calculation
        # Concatenate hash features and view direction
        input_tensor = torch.cat([hash_features, view_directions], dim=-1)
        density_updates = self.mlp(input_tensor).squeeze(-1) # [N]
        
        # Step 4: Effective Opacity Calculation
        # delta_i is the distance between adjacent samples along the ray.
        # In a full rasterization pipeline, this is computed per sample.
        # Here we assume delta_i is provided or computed by the rasterizer.
        # For the sake of the formula, we use a placeholder delta.
        delta = 0.01 # Placeholder for distance between samples
        
        # alpha_i = 1 - exp(-sigma_i * delta_sigma(d) * delta_i)
        effective_opacities = 1 - torch.exp(-base_opacities.squeeze(-1) * density_updates * delta)
        
        # Step 5: Rasterization (Alpha Compositing)
        # In a real implementation, this is done via differential Gaussian rasterization.
        # We simulate the blending for a single ray with multiple Gaussians.
        # Assume Gaussians are sorted by depth.
        # For simplicity, we assume a single Gaussian per pixel for this pseudocode snippet.
        # In reality, N Gaussians contribute to a pixel.
        
        # Simulating the final color for a single Gaussian contribution
        # C = c_i * alpha_i
        rendered_color = colors * effective_opacities.unsqueeze(-1)
        
        return rendered_color

    def step(self, gt_image, rendered_image):
        """
        Perform one optimization step.
        """
        loss = torch.nn.functional.mse_loss(rendered_image, gt_image)
        loss.backward()
        self.optimizer.step()
        self.optimizer.zero_grad()
        
        # Step 7: Pruning and Densification (simplified)
        # Eliminate Gaussians with low opacity
        # Generate new Gaussians in unoccupied areas
        pass

    def parameters(self):
        return list(self.mlp.parameters()) + list(self.hash_grid.parameters())

# Helper function for hash encoding (placeholder)
def hash_encode(means, levels, table_size):
    # This function would map 3D coordinates to hash indices
    # based on the structured hash grid implementation.
    # It is not defined in the sources, so we leave it as a comment.
    # The sources specify 12 levels, 2^14 table size, and feature dim 2.
    pass
```

**Parameters:**
*   Hash Grid Levels: 12 [S1].
*   Hash Table Size: $2^{14}$ (16,384) [S1].
*   Feature Dimension: 2 [S1].
*   Activation Function: LeakyReLU [S1].
*   MLP Layers: 2 [S1].

**Edge Cases and Pitfalls:**
*   **Joint Color and Opacity Update:** Updating both color and opacity via the neural network yields slightly worse results than updating only one. This is attributed to the dependency of opacity on color, making joint training ineffective [S1].
*   **Opacity vs. Color Update:** Changing only opacity (Opacity Multiply) gives the most consistently high-quality results on real scenes. Changing only color (Color Multiply) yields the best scores on the NeRF Synthetic dataset [S1].
*   **Precision:** The hash grid provides extremely fast queries, but the discrete nature may introduce quantization artifacts if the resolution is too low [S1].

## Operating characteristics
*   **Time Complexity:** Training time is comparable to standard Gaussian Splatting, with a slight increase due to the MLP evaluation. For VDGS-30K on RTX 3090: Mip-NeRF360 46m11s, Tanks&Temples 29m45s, Deep Blending 41m29s [S1].
*   **Memory Complexity:** Memory usage is dominated by the Gaussian parameters and the hash grid. The hash grid size is $2^{14} \times 12 \times 2$ floats, which is small compared to the Gaussian parameters [S1].
*   **Measured Performance:**
    *   Mip-NeRF360: SSIM 0.813, PSNR 27.64, LPIPS 0.220 [S1].
    *   Tanks and Temples: SSIM 0.851, PSNR 24.02, LPIPS 0.176 [S1].
    *   Deep Blending: SSIM 0.906, PSNR 29.54, LPIPS 0.243 [S1].
    *   Shiny Blender: Avg PSNR 30.62 [S1].
*   **Inference Speed:** Inference FPS on RTX 3090 for VDGS-30K*: Mip-NeRF360 41.35, Tanks&Temples 38.53, Deep Blending 44.72 [S1].
*   **Strengths:** Fast training and inference compared to NeRF; accurate modeling of light reflection, transparency, and glass; view-dependent appearance without significant training time increase [S1].
*   **Limitations:** Slightly worse performance than standard GS on synthetic datasets if not tuned correctly; joint color and opacity update is ineffective [S1].
*   **When to Use:** When rendering scenes with shiny surfaces, glass, or strong view-dependent effects where standard GS fails to capture reflections and transparency [S1].
*   **When Not to Use:** When computational resources are extremely limited and view-dependent effects are not critical; standard GS may be sufficient and faster [S1].

## Background
Viewing Direction Gaussian Splatting (VDGS) was proposed by researchers from the Faculty of Mathematics and Computer Science, Jagiellonian University, and the Department of Engineering, University of Cambridge [S1]. The method was published in December 2023 [S1]. It addresses the limitations of both NeRF (slow training/inference, difficulty in precise geometry location) and GS (hard to condition, static appearance) by combining the shape representation of GS with the appearance modeling of NeRF [S1]. The implementation uses PyTorch and TinyCUDA frameworks on NVIDIA RTX 3090 GPU [S1].

## Sources
[S1] Gaussian Splatting with NeRF-based Color and Opacity, https://doi.org/10.48550/arxiv.2312.13729, fetched 2026-10-29
