# LoD-NeuS

Also called: Anti-Aliased Neural Implicit Surfaces with Encoding Level of Detail

## Overview
LoD-NeuS is a neural implicit surface reconstruction method that encodes features with a continuous level of detail (LoD) from a tri-plane-based representation to adaptively reconstruct high-fidelity geometry and address aliasing concerns [S1]. It builds on [Tri-MipRF Training Configuration](11504-tri-miprf-training-configuration.md) by introducing multi-scale tri-plane position encoding and multi-convolved featurization to approximate ray integrals within a cone, distinguishing itself from standard NeuS by reducing MLP queries while maintaining high-frequency detail [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images, camera parameters, and an initial set of sparse points or volume (for initialization).
*   **Outputs:** A signed distance function (SDF) field and a color field, defining the 3D geometry and appearance.

**Data Structures:**
*   **Tri-Planes:** Three 2D feature planes ($xy$, $yz$, $zx$) representing the 3D space. Each plane has a specific resolution and number of channels. The features are initialized using a Gaussian distribution derived from the spatial coordinates [S1].
*   **Geometry Network:** An MLP with 8 hidden layers, hidden size 256, and weight normalization [S1].
*   **Color Network:** An MLP taking features, position, view direction, and normal to predict color [S1].

**Numbered Steps:**

1.  **Multi-Scale Tri-Plane Position Encoding:**
    For a 3D point $\mathbf{x} = (x, y, z)$, encode it into multiple scales. The tri-planes are initialized such that the variance $\sigma^2$ for each plane is proportional to the squared distance of the coordinates in that plane's subspace [S1]. Specifically, for $n$ scales, the initialization is:
    $$ \sigma_{xy}^2 = \frac{1}{2n}(x^2 + y^2), \quad \sigma_{yz}^2 = \frac{1}{2n}(y^2 + z^2), \quad \sigma_{zx}^2 = \frac{1}{2n}(z^2 + x^2) $$
    This ensures that higher frequencies are captured at finer scales [S1].

2.  **Multi-Convolved Featurization (Cone Discrete Sampling):**
    To approximate the ray integral within a cone and reduce aliasing, the method aggregates LoD features from multi-convolved multi-scale features of vertices within a conical frustum along a ray [S1]. This replaces standard point sampling with a cone-tracing approximation. Compared to 4x super-sampling, this requires only 1/4 of the MLP queries [S1].

3.  **SDF and Color Prediction:**
    *   The aggregated features are passed to the geometry network to predict the SDF value $\hat{d}$ and feature vector $\mathbf{f}$ [S1].
    *   The geometry network uses a skip connection: the input is concatenated with the output of the 4th hidden layer [S1].
    *   The color network takes $\mathbf{f}$, the 3D position, view direction, and surface normal (derived from the SDF gradient) to predict the color $\hat{c}$ [S1].

4.  **SDF Growth Refinement (Optional/For Thin Surfaces):**
    For thin surfaces, an SDF growth refinement is applied based on SDF sphere tracing to improve reconstruction [S1]. This involves rendering rays around error maps and refining the SDF values [S1].

5.  **Loss Computation and Optimization:**
    *   Compute photometric loss between rendered and input images.
    *   Compute SDF loss (e.g., Eikonal loss, consistency loss) [S1].
    *   Update network weights using the ADAM optimizer [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class LoDNeuS:
    def __init__(self, tri_plane_res, num_scales, hidden_size=256, num_layers=8):
        """
        Initialize LoD-NeuS model.
        :param tri_plane_res: Resolution of the tri-planes (H, W)
        :param num_scales: Number of LoD scales
        :param hidden_size: Hidden layer size for MLPs
        :param num_layers: Number of hidden layers in geometry network
        """
        self.num_scales = num_scales
        self.tri_plane_res = tri_plane_res
        
        # Initialize Tri-Planes with Gaussian distribution based on spatial coordinates
        # See Eq. 11 in [S1] for initialization logic
        self.tri_planes = nn.ParameterList([
            nn.Parameter(torch.randn(3, num_scales, hidden_size, tri_plane_res[0], tri_plane_res[1]))
        ])
        
        # Geometry Network: 8 hidden layers, size 256, skip connection at layer 4
        self.geometry_net = nn.Sequential(
            nn.Linear(hidden_size + 3, hidden_size), # Input: features + xyz
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size), # Layer 4 output
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, 1 + hidden_size) # Output: SDF + features
        )
        
        # Apply Weight Normalization to stabilize training [S1]
        for m in self.geometry_net.modules():
            if isinstance(m, nn.Linear):
                nn.utils.weight_norm(m)
                
        # Color Network (simplified)
        self.color_net = nn.Sequential(
            nn.Linear(hidden_size + 3 + 3, 128), # Features + xyz + view_dir
            nn.ReLU(),
            nn.Linear(128, 3) # Output: RGB
        )
        
        self.optimizer = torch.optim.Adam(self.parameters(), lr=5e-4)
        self.scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(self.optimizer, T_max=300000)

    def encode_position(self, xyz, tri_planes):
        """
        Multi-scale tri-plane position encoding.
        :param xyz: (N, 3) coordinates
        :param tri_planes: List of 3 tensors (xy, yz, zx) for each scale
        :return: Aggregated features (N, hidden_size)
        """
        # Extract xy, yz, zx coordinates
        x, y, z = xyz[:, 0:1], xyz[:, 1:2], xyz[:, 2:3]
        
        features = []
        for scale in range(self.num_scales):
            # Get tri-planes for this scale
            # tri_planes[scale] is a list of 3 tensors: [xy, yz, zx]
            xy_plane, yz_plane, zx_plane = tri_planes[scale]
            
            # Sample features from tri-planes using bilinear interpolation
            # Normalize coordinates to [-1, 1] for sampling
            xy_coords = torch.cat([x, y], dim=1)
            yz_coords = torch.cat([y, z], dim=1)
            zx_coords = torch.cat([z, x], dim=1)
            
            # Sample from planes (pseudo-code for grid_sample)
            xy_feat = F.grid_sample(xy_plane.unsqueeze(0), xy_coords.unsqueeze(0).unsqueeze(0), mode='bilinear', padding_mode='border').squeeze()
            yz_feat = F.grid_sample(yz_plane.unsqueeze(0), yz_coords.unsqueeze(0).unsqueeze(0), mode='bilinear', padding_mode='border').squeeze()
            zx_feat = F.grid_sample(zx_plane.unsqueeze(0), zx_coords.unsqueeze(0).unsqueeze(0), mode='bilinear', padding_mode='border').squeeze()
            
            # Aggregate features from the three planes
            scale_feat = (xy_feat + yz_feat + zx_feat) / 3.0
            features.append(scale_feat)
            
        # Concatenate features from all scales
        aggregated_feat = torch.cat(features, dim=1)
        return aggregated_feat

    def forward(self, xyz, view_dir, tri_planes):
        """
        Forward pass.
        :param xyz: (N, 3) 3D points
        :param view_dir: (N, 3) view directions
        :param tri_planes: List of tri-planes for each scale
        :return: SDF, color, normal
        """
        # Step 1: Encode position with multi-scale tri-planes
        features = self.encode_position(xyz, tri_planes)
        
        # Step 2: Predict SDF and features using geometry network
        # Concatenate features with xyz for the first layer
        geo_input = torch.cat([features, xyz], dim=1)
        geo_output = self.geometry_net(geo_input)
        sdf = geo_output[:, 0:1]
        geo_features = geo_output[:, 1:]
        
        # Step 3: Compute normal from SDF gradient (numerical or analytical)
        # [G] Normal is typically computed via finite differences or automatic differentiation
        normal = self.compute_normal(xyz, sdf)
        
        # Step 4: Predict color using color network
        color_input = torch.cat([geo_features, xyz, view_dir], dim=1)
        color = self.color_net(color_input)
        
        return sdf, color, normal

    def compute_normal(self, xyz, sdf):
        """
        Compute surface normal from SDF.
        """
        # [G] Standard finite difference or autograd method
        eps = 1e-4
        dx = torch.zeros_like(xyz)
        dx[:, 0] = eps
        sdf_dx = self.forward(xyz + dx, None, None)[0]
        dy = torch.zeros_like(xyz)
        dy[:, 1] = eps
        sdf_dy = self.forward(xyz + dy, None, None)[0]
        dz = torch.zeros_like(xyz)
        dz[:, 2] = eps
        sdf_dz = self.forward(xyz + dz, None, None)[0]
        
        normal = torch.cat([sdf_dx - sdf, sdf_dy - sdf, sdf_dz - sdf], dim=1)
        normal = F.normalize(normal, dim=1)
        return normal

    def train_step(self, rays, tri_planes):
        """
        Single training step.
        :param rays: Batch of rays (origin, direction, color)
        :param tri_planes: Current tri-planes
        :return: Loss
        """
        self.optimizer.zero_grad()
        
        # Sample points along rays
        xyz, view_dir, target_color = self.sample_rays(rays)
        
        # Forward pass
        sdf, color, normal = self.forward(xyz, view_dir, tri_planes)
        
        # Compute losses
        # [G] Photometric loss, SDF loss, Eikonal loss, etc.
        loss = self.compute_loss(sdf, color, target_color, normal)
        
        loss.backward()
        self.optimizer.step()
        self.scheduler.step()
        
        return loss.item()

    def sample_rays(self, rays):
        """
        Sample points along rays using hierarchical sampling and cone discrete sampling.
        """
        # [G] Implementation of hierarchical sampling from [Wang et al. 2021]
        # and cone discrete sampling as described in [S1]
        pass

    def compute_loss(self, sdf, color, target_color, normal):
        """
        Compute total loss.
        """
        # [G] Photometric loss
        photometric_loss = F.mse_loss(color, target_color)
        
        # [G] SDF loss (e.g., Eikonal loss)
        eikonal_loss = torch.mean((torch.norm(normal, dim=1) - 1.0) ** 2)
        
        # [G] Other losses (consistency, etc.)
        total_loss = photometric_loss + eikonal_loss
        return total_loss
```

**Parameters:**
*   **Geometry Network:** 8 hidden layers, hidden size 256 [S1].
*   **Skip Connection:** Concatenates input with output of the 4th layer [S1].
*   **Optimizer:** ADAM [S1].
*   **Learning Rate:** Linearly warmed up from 0 to $5 \times 10^{-4}$ in the first 5k iterations, then cosine decay to a minimum of $2.5 \times 10^{-5}$ [S1].
*   **Ray Batch Size:** 512 [S1].
*   **Total Training Iterations:** 300k [S1].
*   **Hierarchical Sampling:** Follows [Wang et al. 2021] [S1].

**Edge Cases and Pitfalls:**
*   **NGP Encoding Artifacts:** Using Neural Graphics Primitives (NGP) encoding can cause performance degradation due to hash collisions, leading to artifacts in areas with scarce textures or observations, as spatial points record surface distance [S1].
*   **Thin Surfaces:** Standard sampling may fail for thin surfaces; the SDF growth refinement module is recommended for such cases [S1].

## Operating characteristics
*   **Time Complexity:** Training takes around 9 hours for 300k iterations on a single Nvidia A100 GPU [S1]. Inference time is approximately 160 seconds for a 1600x1200 image [S1].
*   **Memory Complexity:** GPU memory usage is 13G [S1].
*   **Performance:** Compared to 4x super-sampling, the method requires 1/4 of MLP queries [S1]. Training time is nearly half that of HF-NeuS [S1].
*   **Accuracy:** Achieves an average Chamfer distance of 0.72 on the DTU dataset [S1]. On a specific comparison table (Table 4), it achieves a Chamfer Distance of 0.114 [S1].
*   **Strengths:** High-fidelity geometry reconstruction, reduced computational burden compared to super-sampling, faster training than HF-NeuS [S1].
*   **Limitations:** May require additional regularization or refinement for thin surfaces or areas with scarce textures if using hash-based encodings like NGP [S1].

## Background
LoD-NeuS was introduced in the paper "Anti-Aliased Neural Implicit Surfaces with Encoding Level of Detail" [S1]. It addresses the issue of aliasing in neural surface reconstruction by introducing a continuous level of detail in the encoding process. The method builds upon the NeuS framework [Wang et al. 2021] and incorporates ideas from tri-plane representations [S1]. It was evaluated on the DTU dataset without mask supervision [S1].

## Sources
[S1] Anti-Aliased Neural Implicit Surfaces with Encoding Level of Detail, https://doi.org/10.1145/3610548.3618197, fetched 2026-10-27
