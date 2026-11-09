# GIRAFFE

Also called: Compositional Generative Neural Feature Fields, GIRAFFE Scene Composition

## Overview
GIRAFFE represents scenes as a composition of $N$ distinct entities (objects and a background), where each entity possesses its own neural feature field and learnable affine transformation. This architecture disentangles pose, shape, and appearance, allowing for explicit control over individual objects within a generated scene [S1]. It differs from monolithic NeRF variants by decomposing the scene representation into shared-weight object fields combined via a density-weighted composition operator, rendered through a two-stage process involving low-resolution feature maps and 2D neural upsampling [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera pose $\xi$, number of entities $N$, latent shape codes $\mathbf{z}_s^i$, latent appearance codes $\mathbf{z}_a^i$, and transformation parameters $\mathbf{T}_i = \{\mathbf{s}_i, \mathbf{t}_i, \mathbf{R}_i\}$ for each entity $i \in \{1, \dots, N\}$ [S1].
*   **Outputs:** A synthesized RGB image $\hat{\mathbf{I}} \in \mathbb{R}^{H \times W \times 3}$ [S1].

**Data Structures:**
1.  **Feature Field $h_\theta$:** An MLP with 8 layers, hidden dimension 128, and ReLU activations. It takes positional encoding of transformed coordinates and directions, plus latent codes, and outputs density $\sigma \in \mathbb{R}$ and feature vector $\mathbf{f} \in \mathbb{R}^{M_f}$ [S1].
    *   Input dimension: $L_x + L_d + 2 \cdot \dim(\mathbf{z})$, where $L_x = 60$ (position), $L_d = 24$ (direction) [S1].
    *   Output: Density head (dim 1), Feature head (dim $M_f$). $M_f=128$ for $64\times64$ output, $M_f=256$ for $256\times256$ output [S1].
2.  **Background Field $h_\theta^N$:** Same architecture but with half the layers and hidden dimension [S1].
3.  **Transformation Parameters:**
    *   $\mathbf{s}_i \in \mathbb{R}^3$: Scale.
    *   $\mathbf{t}_i \in \mathbb{R}^3$: Translation.
    *   $\mathbf{R}_i \in SO(3)$: Rotation matrix [S1].
    *   For the background ($i=N$), $\mathbf{s}_N$ and $\mathbf{t}_N$ are fixed to span the scene and center at the origin; $\mathbf{R}_N$ is identity [S1].
4.  **Feature Image $I_V$:** Low-resolution feature map of size $H_V \times W_V \times M_f$ (typically $16 \times 16$) [S1].
5.  **Neural Renderer $\pi_\theta^{\text{neural}}$:** A 2D CNN mapping $I_V$ to final RGB image. Uses leaky ReLU, nearest neighbor upsampling, and $3\times3$ convolutions with skip connections [S1].

**Numbered Steps:**

1.  **Sample Entities and Parameters:**
    Sample the number of entities $N \sim p_N$. For each entity $i \in \{1, \dots, N\}$, sample latent codes $\mathbf{z}_s^i, \mathbf{z}_a^i \sim \mathcal{N}(0, I)$ and transformation $\mathbf{T}_i \sim p_T$. Sample camera pose $\xi \sim p_\xi$ [S1].

2.  **Define Ray Sampling:**
    For each pixel in the target image, define a ray $\mathbf{d}_k$ based on $\xi$. Sample $N_s = 64$ points $\{\mathbf{x}_j\}_{j=1}^{N_s}$ along the ray [S1].

3.  **Transform Coordinates to Object Space:**
    For each sample point $\mathbf{x}_j$ and direction $\mathbf{d}_j$ (parallel to ray), compute the inverse transformation for each entity $i$:
    $$
    k_i^{-1}(\mathbf{x}_j) = \mathbf{R}_i^{-1} \left( \begin{bmatrix} 1/s_{i,1} & 0 & 0 & 0 \\ 0 & 1/s_{i,2} & 0 & 0 \\ 0 & 0 & 1/s_{i,3} & 0 \\ 0 & 0 & 0 & 1/s_{i,3} \end{bmatrix} (\mathbf{x}_j - \mathbf{t}_i) \right)
    $$
    $$
    k_i^{-1}(\mathbf{d}_j) = \mathbf{R}_i^{-1} \left( \begin{bmatrix} 1/s_{i,1} & 0 & 0 & 0 \\ 0 & 1/s_{i,2} & 0 & 0 \\ 0 & 0 & 1/s_{i,3} & 0 \\ 0 & 0 & 0 & 1/s_{i,3} \end{bmatrix} \mathbf{d}_j \right)
    $$
    Note: The transformation matrix in the source uses $s_3$ for the last diagonal element in the homogeneous coordinate context, effectively scaling the translation component similarly or treating it as a uniform scale for the homogeneous part [S1].

4.  **Evaluate Feature Fields:**
    Apply positional encoding $\gamma(\cdot)$ to the transformed coordinates and directions. Input to $h_\theta$ for entity $i$:
    $$
    (\sigma_{i,j}, \mathbf{f}_{i,j}) = h_\theta(\gamma(k_i^{-1}(\mathbf{x}_j)), \gamma(k_i^{-1}(\mathbf{d}_j)), \mathbf{z}_s^i, \mathbf{z}_a^i)
    $$
    For the background entity $N$, use $h_\theta^N$ with fixed $\mathbf{T}_N$ [S1].

5.  **Compose Entities (Composition Operator):**
    Calculate the total density and weighted feature vector at sample point $\mathbf{x}_j$:
    $$
    \sigma_j = \sum_{i=1}^N \sigma_{i,j}
    $$
    $$
    \mathbf{f}_j = \frac{1}{\sigma_j} \sum_{i=1}^N \sigma_{i,j} \mathbf{f}_{i,j} \quad \text{if } \sigma_j > 0, \text{ else } \mathbf{0}
    $$
    This ensures gradient flow to all entities with non-zero density [S1].

6.  **Volume Render Feature Vector:**
    Compute the final feature vector $\mathbf{f}$ for the pixel by integrating along the ray:
    $$
    \mathbf{f} = \sum_{j=1}^{N_s} \tau_j \alpha_j \mathbf{f}_j
    $$
    where:
    $$
    \alpha_j = 1 - \exp(-\sigma_j \delta_j)
    $$
    $$
    \tau_j = \prod_{k=1}^{j-1} (1 - \alpha_k)
    $$
    $$
    \delta_j = \|\mathbf{x}_{j+1} - \mathbf{x}_j\|_2
    $$
    This produces the low-resolution feature image $I_V \in \mathbb{R}^{H_V \times W_V \times M_f}$ [S1].

7.  **2D Neural Rendering:**
    Upsample $I_V$ to the final resolution $H \times W$ using $\pi_\theta^{\text{neural}}$. The network combines nearest neighbor upsampling with $3\times3$ convolutions. Skip connections add bilinearly upsampled previous outputs to the current convolutional output. The final layer applies a sigmoid activation to produce $\hat{\mathbf{I}} \in \mathbb{R}^{H \times W \times 3}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GIRAFFE:
    def __init__(self, Mf=128, Ns=64, H_V=16, W_V=16, H=64, W=64):
        # Feature Field MLP
        self.feature_field = FeatureMLP(hidden_dim=128, n_layers=8, Mf=Mf)
        # Background Field MLP (half depth/dim)
        self.bg_field = FeatureMLP(hidden_dim=64, n_layers=4, Mf=Mf)
        # 2D Neural Renderer
        self.neural_renderer = NeuralRenderer2D(Mf=Mf, H_V=H_V, W_V=W_V, H=H, W=W)
        
        self.Ns = Ns
        self.Mf = Mf

    def positional_encoding(self, x, L):
        """Standard sinusoidal positional encoding."""
        # x shape: (..., D)
        # Returns shape: (..., L*D)
        freq_bands = 2.0 ** torch.linspace(0, L-1, L).to(x.device) * torch.pi
        # Expand dims for broadcasting
        freq_bands = freq_bands.view(1, -1)
        x_expanded = x.unsqueeze(-1) * freq_bands.unsqueeze(0)
        pe = torch.cat([torch.sin(x_expanded), torch.cos(x_expanded)], dim=-1)
        return pe.view(x.shape[0], -1)

    def transform_point(self, x, s, t, R):
        """Transform point from scene to object space."""
        # x: (..., 3)
        # s: (3,), t: (3,), R: (3,3)
        # Inverse transform: R^-1 * (S^-1 * (x - t))
        x_shifted = x - t
        x_scaled = x_shifted / s.unsqueeze(0) # Broadcast division
        x_rotated = torch.matmul(x_scaled, R.T) # (..., 3) @ (3, 3).T
        return x_rotated

    def transform_direction(self, d, s, R):
        """Transform direction from scene to object space."""
        # Directions are not translated, only scaled and rotated
        d_scaled = d / s.unsqueeze(0)
        d_rotated = torch.matmul(d_scaled, R.T)
        return d_rotated

    def forward(self, camera_poses, entity_transforms, latent_codes):
        """
        camera_poses: (B, 4, 4) or extrinsics
        entity_transforms: List of N dicts with 's', 't', 'R'
        latent_codes: List of N tuples (z_s, z_a)
        """
        B = camera_poses.shape[0]
        N = len(entity_transforms)
        
        # Initialize rays based on camera poses (simplified)
        # In practice, generate rays for each pixel
        # For this pseudocode, assume we have ray origins O and directions D
        # Shape: (B, H, W, 3)
        O, D = generate_rays(camera_poses) 
        
        # Sample points along rays
        # Assume uniform sampling or stratified sampling
        # t values: (B, H, W, Ns)
        t = torch.linspace(0.1, 1.0, self.Ns).to(O.device).view(1, 1, 1, -1)
        # Expand to (B, H, W, Ns, 3)
        # X = O[..., None, :] + D[..., None, :] * t[..., None]
        # Note: Exact ray generation depends on camera model. 
        # Assuming O and D are already prepared for all pixels.
        
        # We need to aggregate features per pixel.
        # Feature Image I_V shape: (B, H_V, W_V, Mf)
        # But we render at 16x16 first.
        
        # Let's assume we render to 16x16 grid first.
        # Map pixel coordinates to 16x16
        # This part is complex to pseudocode without specific ray casting logic.
        # We will outline the core loop per sample point.
        
        # To keep it implementable:
        # 1. Cast rays for 16x16 grid.
        # 2. Sample Ns points.
        # 3. Compute sigma, f for each entity.
        # 4. Compose.
        # 5. Volume render.
        # 6. Upsample.
        
        # Placeholder for ray casting logic specific to the camera model
        # rays_O, rays_D = cast_rays_16x16(camera_poses)
        
        # For the sake of the algorithm description, we assume
        # we have batched inputs for all sample points of the 16x16 grid.
        # Shape: (B * 16 * 16 * Ns, 3)
        
        # ... (Ray casting implementation omitted for brevity, see Parent) ...
        
        # Example of processing one batch of points
        # points_scene: (Batch, 3)
        # dirs_scene: (Batch, 3)
        
        # Initialize accumulated features
        # We need to sum sigmas and weighted features
        
        # This loop runs over N entities
        total_sigma = 0
        weighted_feature_sum = 0
        
        for i in range(N):
            s_i = entity_transforms[i]['s']
            t_i = entity_transforms[i]['t']
            R_i = entity_transforms[i]['R']
            z_s, z_a = latent_codes[i]
            
            # Transform
            points_obj = self.transform_point(points_scene, s_i, t_i, R_i)
            dirs_obj = self.transform_direction(dirs_scene, s_i, R_i)
            
            # PE
            pe_x = self.positional_encoding(points_obj, L=10) # L_x = 2*3*10 -> 10 bands
            pe_d = self.positional_encoding(dirs_obj, L=4)   # L_d = 2*3*4 -> 4 bands
            
            # Concatenate latent codes
            # Assuming z_s and z_a are concatenated or passed separately
            # The paper says z_s, z_a ~ N(0,I). 
            # Input to MLP: [pe_x, pe_d, z_s, z_a]
            input_vec = torch.cat([pe_x, pe_d, z_s, z_a], dim=-1)
            
            # Predict
            if i == N - 1: # Background
                sigma_i, f_i = self.bg_field(input_vec)
            else:
                sigma_i, f_i = self.feature_field(input_vec)
            
            # Accumulate
            # sigma_i: (Batch, 1)
            # f_i: (Batch, Mf)
            total_sigma += sigma_i
            weighted_feature_sum += sigma_i * f_i
            
        # Compose
        # Avoid division by zero
        mask = total_sigma > 1e-5
        feature_composed = torch.where(mask, weighted_feature_sum / total_sigma, torch.zeros_like(weighted_feature_sum))
        
        # Volume Render
        # Need delta_j. Assuming uniform spacing for simplicity in this snippet
        # delta = (t_max - t_min) / Ns
        delta = 0.9 / self.Ns
        
        alpha = 1.0 - torch.exp(-total_sigma * delta)
        transmittance = torch.ones_like(alpha)
        for j in range(1, self.Ns):
            transmittance[:, j:] *= (1.0 - alpha[:, :j])
            
        # Final feature vector per pixel (aggregated over Ns samples)
        # This requires reshaping to pixel level
        # feature_image_raw: (B, 16, 16, Mf)
        feature_image_raw = self.volume_render_feature(feature_composed, alpha, transmittance, points_scene)
        
        # 2D Neural Rendering
        output_image = self.neural_renderer(feature_image_raw)
        
        return output_image

class FeatureMLP(nn.Module):
    def __init__(self, hidden_dim, n_layers, Mf):
        super().__init__()
        layers = []
        in_dim = 84 # 60 (pos) + 24 (dir) + latent dims (assumed small or fixed)
        # Note: Exact latent dim not specified, assuming it's part of input
        for _ in range(n_layers):
            layers.append(nn.Linear(in_dim, hidden_dim))
            layers.append(nn.ReLU())
            in_dim = hidden_dim
        self.mlp = nn.Sequential(*layers)
        self.density_head = nn.Linear(hidden_dim, 1)
        self.feature_head = nn.Linear(hidden_dim, Mf)
        
    def forward(self, x):
        h = self.mlp(x)
        sigma = self.density_head(h)
        f = self.feature_head(h)
        return sigma, f

class NeuralRenderer2D(nn.Module):
    def __init__(self, Mf, H_V, W_V, H, W):
        super().__init__()
        # Simple upsampling CNN
        self.conv1 = nn.Conv2d(Mf, 64, 3, padding=1)
        self.conv2 = nn.Conv2d(64, 3, 3, padding=1)
        self.leaky_relu = nn.LeakyReLU(0.2)
        self.H_V = H_V
        self.W_V = W_V
        self.H = H
        self.W = W
        
    def forward(self, x):
        # x: (B, Mf, H_V, W_V)
        # Upsample to H, W
        x = F.interpolate(x, size=(self.H, self.W), mode='nearest')
        x = self.conv1(x)
        x = self.leaky_relu(x)
        # Skip connection logic would involve bilinear upsample of original
        # For simplicity, just final conv and sigmoid
        out = self.conv2(x)
        out = torch.sigmoid(out)
        return out
```

**Parameters:**
*   $N$: Number of entities. Fixed or varied per dataset [S1].
*   $M_f$: Feature dimension. 128 for $64\times64$, 256 for $256\times256$ [S1].
*   $N_s$: Number of samples per ray. 64 [S1].
*   $H_V, W_V$: Feature image resolution. $16\times16$ [S1].
*   Generator MLP: 8 layers, hidden dim 128, ReLU. Background: half layers/dim [S1].
*   Positional Encoding: $L_x=10$ bands (60 dims), $L_d=4$ bands (24 dims) [S1].
*   Optimizer: RMSprop. Generator LR: $1\times10^{-4}$ ($2.5\times10^{-4}$ for 256px). Discriminator LR: $5\times10^{-4}$. Batch size: 32. EMA decay: 0.999 [S1].
*   GAN Penalty: $\lambda = 10$ [S1].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** When $\sigma = 0$ in the composition operator, the feature vector is set to 0 [S1].
*   **Background Transformation:** The background's scale and translation are fixed to span the scene and center at the origin. Incorrectly learning these can break scene alignment [S1].
*   **Resolution Mismatch:** Rendering feature maps at $16\times16$ and upsampling requires the 2D neural renderer to be capable of high-quality upsampling without entangling global properties [S1].

## Operating characteristics
*   **Time Complexity:** Rendering is dominated by the MLP evaluations for $N_s \times N$ entities per pixel. The 2D upsampling is $O(H \cdot W)$.
*   **Memory Complexity:** Stores $N$ feature fields (shared weights) and $N$ sets of transformation parameters and latent codes.
*   **Strengths:** Explicit control over object pose, shape, and appearance. Disentangled representation allows for scene composition and editing.
*   **Limitations:** Requires defining the number of entities $N$. Performance depends on the quality of the 2D neural renderer for high-resolution output.
*   **When to use:** When scene disentanglement and generative control over individual objects are required.
*   **When not to use:** For scenes with a fixed, known number of objects where simple monolithic NeRF suffices, or when real-time performance is critical without the overhead of GAN training.

## Background
GIRAFFE was introduced by Kong et al. in 2021. It extends the NeRF framework by introducing compositional generative neural feature fields. The method is designed to disentangle entities in a scene, allowing for the generation of novel scenes with controlled object properties. It uses a GAN objective for training the generative model [S1].

## Sources
[S1] GIRAFFE: Representing Scenes as Compositional Generative Neural Feature Fields, https://github.com/chaerinkong/paper-reviews/issues/63, fetched 2026-10-28
