# LightningNeRF

Also called: Lightning NeRF

## Overview
Lightning NeRF is an efficient hybrid scene representation for autonomous driving that uses explicit voxel grids for density and implicit hash grids for color, initialized with LiDAR point clouds to improve training speed and rendering quality [S1]. It separates foreground and background encodings, utilizing inverse cubic modeling for the background to warp 3D points into 4D coordinates, and employs separate view-independent and view-dependent MLPs for color decomposition [S2, unofficial]. It achieves a five-fold increase in training speed and ten-fold improvement in rendering speed compared to state-of-the-art methods on real-world datasets [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera poses, images, LiDAR point clouds (for initialization), scene bounding box (AABB).
*   **Outputs:** Rendered RGB images, accumulation (opacity), depth.

**Data Structures:**
*   **Foreground Density Grid:** HashGrid encoding, 3D input, 1D output.
*   **Foreground Color Grid:** Multi-resolution HashGrid encoding, 3D input, concatenated features.
*   **Background Density Grid:** HashGrid encoding, 4D input (warped 3D + inverse norm), 1D output.
*   **Background Color Grid:** Multi-resolution HashGrid encoding, 4D input, concatenated features.
*   **Occupancy Grid:** Sparse grid for efficient sampling, updated based on density.
*   **MLPs:** View-independent (Vi) and View-dependent (Vd) networks for color.

**Parameters:**
*   `density_res`: 256 (Foreground density base resolution) [S2, unofficial].
*   `density_log2_hashmap_size`: 24 (Foreground density hashmap size) [S2, unofficial].
*   `color_base_res`: 16 (Foreground color base resolution) [S2, unofficial].
*   `color_max_res`: 256 (Foreground color max resolution) [S2, unofficial].
*   `num_levels`: 8 (Foreground color levels) [S2, unofficial].
*   `features_per_level`: 2 (Foreground color features per level) [S2, unofficial].
*   `color_log2_hashmap_size`: 19 (Foreground color hashmap size) [S2, unofficial].
*   `bg_density_res`: 32 (Background density base resolution) [S2, unofficial].
*   `bg_density_log2_hashmap_size`: 18 (Background density hashmap size) [S2, unofficial].
*   `bg_color_base_res`: 32 (Background color base resolution) [S2, unofficial].
*   `bg_color_max_res`: 128 (Background color max resolution) [S2, unofficial].
*   `bg_color_log2_hashmap_size`: 16 (Background color hashmap size) [S2, unofficial].
*   `vi_mlp_num_layers`: 3 [S2, unofficial].
*   `vi_mlp_hidden_size`: 64 [S2, unofficial].
*   `vd_mlp_num_layers`: 2 [S2, unofficial].
*   `vd_mlp_hidden_size`: 32 [S2, unofficial].
*   `init_density_value`: 10.0 [S2, unofficial].
*   `alpha_thre`: 0.01 [S2, unofficial].
*   `occ_grid_base_res`: 256 [S2, unofficial].
*   `occ_grid_num_levels`: 2 [S2, unofficial].
*   `occ_num_samples_per_ray`: 1000 [S2, unofficial].
*   `pdf_num_samples_per_ray`: 8 [S2, unofficial].
*   `pdf_samples_warmup_step`: 500 [S2, unofficial].
*   `pdf_samples_fixed_step`: 2000 [S2, unofficial].
*   `pdf_samples_fixed_ratio`: 0.5 [S2, unofficial].
*   `cone_angle`: 0.004 (for real scenes) [S2, unofficial].
*   `appearance_embedding_dim`: 0 (default) [S2, unofficial].

**Steps:**

1.  **Scene Definition:** Inspect camera frustums in a trajectory sequence. Define a foreground bounding box tightly wrapping all frustums. Define a background box by enlarging the foreground box proportionally along each dimension [S1].
2.  **LiDAR Initialization:** Load LiDAR point clouds $P_{pc} = \{p_i | p_i \in \mathbb{R}^3\}_{i=1}^{N_{pc}}$. Transform points into normalized space using metadata transform matrix transpose and scale factor [S1][S2, unofficial].
3.  **Background Augmentation:** Generate augmented background points on the front, left, right, and top surfaces of the background box.
    *   Get AABB from occupancy grid.
    *   Generate grid points using `torch.linspace` and `meshgrid`.
    *   For frontal axis 'y': create `aug_neg1` ($x=x_{min}+0.05, y=y_{min}+0.05$), `aug_pos1` ($x=x_{max}-0.05, y=y_{max}-0.05$), `aug_xz` ($y=y_{max}-0.05$), `aug_xy` ($z=z_{max}-0.05$).
    *   For frontal axis 'x': create `aug_neg1` ($y=y_{min}+0.20$), `aug_pos1` ($y=y_{max}-0.20$), `aug_yz` ($x=x_{max}-0.05$), `aug_xy` ($z=z_{max}-0.05$).
    *   Concatenate all augmented points $P_{bg}$ [S2, unofficial].
4.  **Pretrain Density Grid (Foreground):**
    *   Set model to train mode.
    *   Split foreground vertices into 50 batches.
    *   For each batch, query density, compute MSE loss against `init_density_value` (10.0), backpropagate, and step optimizer (lr=0.2).
    *   Update occupancy grid every iteration using $\lambda x: \text{field.density\_fn}(x) * \text{render\_step\_size}$.
    *   Run for 20 iterations [S2, unofficial].
5.  **Pretrain Density Grid (Background):**
    *   Concatenate background ($P_{bg}$) and augmented vertices.
    *   Split into 10 batches.
    *   For each batch, query density, compute MSE loss against `init_density_value` * 0.8, backpropagate, and step optimizer (lr=0.2).
    *   Update occupancy grid every iteration.
    *   Run for 40 iterations [S2, unofficial].
6.  **Initialize Encodings:**
    *   **Foreground Density:** `tcnn.Encoding` with `n_input_dims=3`, `otype='HashGrid'`, `n_levels=1`, `n_features_per_level=1`, `log2_hashmap_size=24`, `base_resolution=256`, `interpolation='Linear'`. Initialize parameters uniformly in $[-3, -2.8]$ [S2, unofficial].
    *   **Background Density:** `tcnn.Encoding` with `n_input_dims=4`, `otype='HashGrid'`, `n_levels=1`, `n_features_per_level=1`, `log2_hashmap_size=18`, `base_resolution=32`, `interpolation='Linear'`. Initialize parameters uniformly in $[-3, -2.8]$ [S2, unofficial].
    *   **Foreground Color:** Calculate growth factor $g = \exp(\frac{\log(256) - \log(16)}{8 - 1})$. `tcnn.Encoding` with `n_input_dims=3`, `otype='HashGrid'`, `n_levels=8`, `n_features_per_level=2`, `log2_hashmap_size=19`, `base_resolution=16`, `per_level_scale=g` [S2, unofficial].
    *   **Background Color:** Calculate growth factor $g_{bg} = \exp(\frac{\log(128) - \log(32)}{8 - 1})$. `tcnn.Encoding` with `n_input_dims=4`, `otype='HashGrid'`, `n_levels=8`, `n_features_per_level=2`, `log2_hashmap_size=16`, `base_resolution=32`, `per_level_scale=g_{bg}$ [S2, unofficial].
    *   **Appearance Embedding:** If `appearance_embedding_dim > 0`, create `Embedding(num_images, appearance_embedding_dim)` [S2, unofficial].
    *   **View-Independent MLP:** Input dim $d_{color} = 8 \times 2 + \text{appearance\_embedding\_dim}$. `tcnn.Network` with `n_input_dims=d_{color}`, `n_output_dims=3`, `otype='FullyFusedMLP'`, `activation='ReLU'`, `output_activation='Sigmoid'`, `n_neurons=64`, `n_hidden_layers=2` [S2, unofficial].
    *   **Direction Encoding:** `tcnn.Encoding` with `n_input_dims=3`, `otype='Frequency'`, `n_frequencies=4` [S2, unofficial].
    *   **View-Dependent MLP:** Input dim $d_{view} = d_{color} + 4 \times 2$. `tcnn.Network` with `n_input_dims=d_{view}`, `n_output_dims=3`, `otype='FullyFusedMLP'`, `activation='ReLU'`, `output_activation='Sigmoid'`, `n_neurons=32`, `n_hidden_layers=1` [S2, unofficial].
7.  **Initialize Sampler:**
    *   Calculate `render_step_size` as $\sqrt{\sum (\text{scene\_aabb}[3:] - \text{scene\_aabb}[:3])^2} / \text{occ\_num\_samples\_per\_ray}$ [S2, unofficial].
    *   Create `nerfacc.OccGridEstimator` with `roi_aabb`, `resolution=256`, `levels=2` [S2, unofficial].
    *   Create `LightningNeRFSampler` with occupancy grid and density function [S2, unofficial].
8.  **Density Query (Foreground):**
    *   Flatten positions to $(-1, 3)$.
    *   Normalize positions to $[0, 1]$ using `SceneBox.get_normalized_positions`.
    *   Create selector for positions inside $[0, 1]^3$.
    *   If all positions inside: `density_before_act = density_encoding(positions_norm)` [S2, unofficial].
9.  **Density Query (Background):**
    *   If some positions outside: Split into foreground and background selectors.
    *   For background positions: Shift `bg_positions_norm` from $[0, 1]$ to $[-1, 1]$ by `bg_positions_norm = bg_positions_norm * 2.0 - 1.0` [S2, unofficial].
    *   Compute inf norm `bg_inf_norm = bg_positions_norm.abs().max(-1, keepdim=True).values` [S2, unofficial].
    *   Warp to $[0, 1]$ for tcnn: `bg_positions_warp = (bg_positions_norm / bg_inf_norm + 1.0) / 2.0` [S2, unofficial].
    *   Concatenate warped positions with inverse inf norm: `bg_positions_warp = torch.cat([bg_positions_warp, torch.div(1., bg_inf_norm)], dim=-1)` [S2, unofficial].
    *   Compute `bg_density_before_act = bg_density_encoding(bg_positions_warp)` [S2, unofficial].
    *   Combine foreground and background densities: Initialize `density_before_act` with zeros. Assign `fg_density_before_act` to `fg_selector` indices and `bg_density_before_act` to `bg_selector` indices [S2, unofficial].
10. **Final Density:** Apply `trunc_exp` to `density_before_act` to get final density $\sigma = \text{trunc\_exp}(\text{density\_before\_act})$. Reshape to match input positions shape [S2, unofficial].
11. **Color Query:**
    *   Get density and color embeddings for ray samples.
    *   For foreground: `fg_color_embedding = color_encoding(positions_norm)` [S2, unofficial].
    *   For background: `bg_color_embedding = bg_color_encoding(bg_positions_warp)` (using same warping as density) [S2, unofficial].
    *   Concatenate color embeddings and appearance embedding (if used) [S2, unofficial].
12. **Color Decomposition:**
    *   **View-Independent Color:** $c_{vi} = \text{ViMLP}(\text{color\_embedding})$ [S1][S2, unofficial].
    *   **View-Dependent Color:** Encode viewing direction $d$ using frequency encoding. $c_{vd} = \text{VdMLP}(\text{color\_embedding}, \text{direction\_encoding}(d))$ [S1][S2, unofficial].
    *   Final color $c = c_{vi} + c_{vd}$ (or combined as per implementation, typically additive or concatenated before sigmoid, sources imply separate MLPs for decomposition) [S1][S2, unofficial].
13. **Rendering:**
    *   Query volume density $\sigma$ and color embedding feature $f$ from voxel grids for sample points along a ray.
    *   Combine with viewing direction $d$ in separate MLPs for view-dependent and view-independent colors.
    *   Combine components to achieve the final rendered image using standard volume rendering integration [S1].
    *   Use occupancy grid for sparse sampling, clustering around object surfaces [S1].
    *   Update occupancy grid based on initialized density mapping to ensure sample locations are sparse in the scene and cluster around object surfaces [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def trunc_exp(x):
    """Stable exponential truncation."""
    return torch.exp(torch.clamp(x, max=15.0))

class LightningNeRF:
    def __init__(self, aabb, num_images, config):
        self.aabb = aabb
        self.num_images = num_images
        self.config = config
        
        # Initialize encodings
        self.density_encoding = self._init_hashgrid(3, 1, 1, config.density_log2_hashmap_size, config.density_res)
        self.density_encoding.params.data.uniform_(-3., -2.8)
        
        self.bg_density_encoding = self._init_hashgrid(4, 1, 1, config.bg_density_log2_hashmap_size, config.bg_density_res)
        self.bg_density_encoding.params.data.uniform_(-3., -2.8)
        
        growth_factor = np.exp((np.log(config.color_max_res) - np.log(config.color_base_res)) / (config.num_levels - 1))
        self.color_encoding = self._init_hashgrid(3, config.num_levels, config.features_per_level, config.color_log2_hashmap_size, config.color_base_res, growth_factor)
        
        bg_growth_factor = np.exp((np.log(config.bg_color_max_res) - np.log(config.bg_color_base_res)) / (config.num_levels - 1))
        self.bg_color_encoding = self._init_hashgrid(4, config.num_levels, config.features_per_level, config.bg_color_log2_hashmap_size, config.bg_color_base_res, bg_growth_factor)
        
        if config.appearance_embedding_dim > 0:
            self.appearance_embedding = torch.nn.Embedding(num_images, config.appearance_embedding_dim)
        else:
            self.appearance_embedding = None
            
        in_dim_color = config.num_levels * config.features_per_level
        if config.appearance_embedding_dim > 0:
            in_dim_color += config.appearance_embedding_dim
            
        self.vi_mlp = self._init_mlp(in_dim_color, 3, config.vi_mlp_hidden_size, config.vi_mlp_num_layers - 1)
        
        self.direction_encoding = self._init_freq_encoding(3, 4)
        in_dim_view = in_dim_color + self.direction_encoding.n_output_dims
        self.vd_mlp = self._init_mlp(in_dim_view, 3, config.vd_mlp_hidden_size, config.vd_mlp_num_layers - 1)
        
        # Sampler initialization would go here
        # self.sampler = ...

    def _init_hashgrid(self, n_input_dims, n_levels, n_features_per_level, log2_hashmap_size, base_resolution, per_level_scale=None):
        # Placeholder for tcnn.Encoding initialization logic
        # Returns an object with .params and .forward()
        pass

    def _init_freq_encoding(self, n_input_dims, n_frequencies):
        # Placeholder for tcnn.Encoding initialization logic
        pass

    def _init_mlp(self, n_input_dims, n_output_dims, n_neurons, n_hidden_layers):
        # Placeholder for tcnn.Network initialization logic
        pass

    def get_density(self, positions):
        """
        positions: Tensor of shape (..., 3)
        """
        positions_flat = positions.view(-1, 3)
        positions_norm = self._normalize_positions(positions_flat)
        selector = ((positions_norm > 0.0) & (positions_norm < 1.0)).all(dim=-1)
        
        density_before_act = torch.zeros((positions_flat.shape[0], 1), device=positions_flat.device, dtype=positions_flat.dtype)
        
        if selector.all():
            fg_density_before_act = self.density_encoding(positions_norm)
            density_before_act = fg_density_before_act
        else:
            fg_selector = selector
            bg_selector = ~selector
            
            fg_positions_norm = positions_norm[fg_selector]
            fg_density_before_act = self.density_encoding(fg_positions_norm)
            
            bg_positions_norm = positions_norm[bg_selector]
            bg_positions_norm_shifted = bg_positions_norm * 2.0 - 1.0
            bg_inf_norm = bg_positions_norm_shifted.abs().max(-1, keepdim=True).values
            bg_positions_warp = (bg_positions_norm_shifted / bg_inf_norm + 1.0) / 2.0
            bg_positions_warp = torch.cat([bg_positions_warp, torch.div(1., bg_inf_norm)], dim=-1)
            bg_density_before_act = self.bg_density_encoding(bg_positions_warp)
            
            density_before_act[fg_selector] = fg_density_before_act
            density_before_act[bg_selector] = bg_density_before_act
            
        density = trunc_exp(density_before_act)
        return density.view(*positions.shape[:-1], 1)

    def get_color(self, positions, direction, image_idx):
        """
        positions: Tensor of shape (..., 3)
        direction: Tensor of shape (..., 3)
        image_idx: Tensor of shape (...)
        """
        positions_flat = positions.view(-1, 3)
        positions_norm = self._normalize_positions(positions_flat)
        selector = ((positions_norm > 0.0) & (positions_norm < 1.0)).all(dim=-1)
        
        color_embedding = torch.zeros((positions_flat.shape[0], self.config.num_levels * self.config.features_per_level), device=positions_flat.device, dtype=positions_flat.dtype)
        
        if selector.all():
            fg_color_embedding = self.color_encoding(positions_norm)
            color_embedding = fg_color_embedding
        else:
            fg_selector = selector
            bg_selector = ~selector
            
            fg_positions_norm = positions_norm[fg_selector]
            fg_color_embedding = self.color_encoding(fg_positions_norm)
            
            bg_positions_norm = positions_norm[bg_selector]
            bg_positions_norm_shifted = bg_positions_norm * 2.0 - 1.0
            bg_inf_norm = bg_positions_norm_shifted.abs().max(-1, keepdim=True).values
            bg_positions_warp = (bg_positions_norm_shifted / bg_inf_norm + 1.0) / 2.0
            bg_positions_warp = torch.cat([bg_positions_warp, torch.div(1., bg_inf_norm)], dim=-1)
            bg_color_embedding = self.bg_color_encoding(bg_positions_warp)
            
            color_embedding[fg_selector] = fg_color_embedding
            color_embedding[bg_selector] = bg_color_embedding
            
        # Add appearance embedding
        if self.appearance_embedding is not None:
            app_emb = self.appearance_embedding(image_idx.view(-1))
            color_embedding = torch.cat([color_embedding, app_emb], dim=-1)
            
        # View-Independent Color
        c_vi = self.vi_mlp(color_embedding)
        
        # View-Dependent Color
        direction_flat = direction.view(-1, 3)
        direction_enc = self.direction_encoding(direction_flat)
        vd_input = torch.cat([color_embedding, direction_enc], dim=-1)
        c_vd = self.vd_mlp(vd_input)
        
        # Combine colors (implementation detail: sources imply separate MLPs for decomposition)
        # Typically c = c_vi + c_vd or similar combination before final sigmoid if not already applied in MLP
        # The MLPs have Sigmoid output activation, so c_vi and c_vd are in [0,1]
        # Standard decomposition: c = c_vi + c_vd might exceed 1, so often c = c_vi + c_vd is not used directly, 
        # but rather c = c_vi + c_vd is computed and then clamped or the MLPs output raw values.
        # However, sources say output_activation='Sigmoid'. 
        # Let's assume the final color is the sum, or the ViMLP outputs diffuse and VdMLP outputs specular, 
        # and they are added. If Sigmoid is applied, they are in [0,1]. 
        # A common approach is c = c_vi + c_vd, but if both are sigmoid, this is bounded.
        # Another interpretation: ViMLP outputs color, VdMLP outputs modulation.
        # Given "Color Decomposition" section in S1: "radiance is composed of diffuse (view-independent) color and specular (view-dependent) color".
        # And "combine components to achieve the final rendered image".
        # We will assume c = c_vi + c_vd.
        c = c_vi + c_vd
        
        return c.view(*positions.shape[:-1], 3)

    def _normalize_positions(self, positions):
        # Normalize positions to [0, 1] based on self.aabb
        # positions: (N, 3)
        # aabb: (6,) [x_min, y_min, z_min, x_max, y_max, z_max]
        min_val = self.aabb[:3]
        max_val = self.aabb[3:]
        normalized = (positions - min_val) / (max_val - min_val)
        return normalized

    def render(self, ray_bundle):
        # Sample points along rays
        # Get positions and directions
        # Get density
        # Get color
        # Volume render
        pass
```

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than standard NeRF due to explicit density initialization and sparse sampling via occupancy grid. Rendering is also faster [S1].
*   **Memory Complexity:** Uses hash grids for color, which are memory efficient compared to dense voxel grids for color. Background uses 4D hash grids, which may consume more memory than 3D but avoids expanding the scene bounding box as in NGP [S1][S2, unofficial].
*   **Measured Performance:** Five-fold increase in training speed and ten-fold improvement in rendering speed on real-world datasets (KITTI-360, Argoverse2) [S1].
*   **Strengths:** Efficient for large-scale outdoor scenes (autonomous driving). LiDAR initialization improves convergence and quality. Separate foreground/background handling handles unbounded scenes well.
*   **Limitations:** Relies on LiDAR for initialization, which may not be available in all scenarios. Background modeling assumes a specific structure (inverse cubic).
*   **When to use:** Autonomous driving scenarios with LiDAR data. Large-scale outdoor scenes.
*   **When not to use:** Indoor scenes without LiDAR. Small-scale scenes where standard NeRF or NGP might be simpler.

## Background
Lightning NeRF was proposed to address the challenges of applying NeRF to outdoor environments, specifically for autonomous driving. It builds on the idea of using LiDAR point clouds to provide geometric priors, which are often overlooked in terms of efficiency in previous methods [S1]. The method was evaluated on KITTI-360, Argoverse2, and a private autonomous driving dataset [S1]. It was presented in 2024 [S1].

## Sources
[S1] Lightning NeRF: Efficient Hybrid Scene Representation for Autonomous Driving, https://doi.org/10.1109/icra57147.2024.10611130, fetched 2026-10-27
[S2] https://github.com/VISION-SJTU/Lightning-NeRF, https://github.com/VISION-SJTU/Lightning-NeRF, fetched 2026-10-29 (unofficial)
