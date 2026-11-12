# Neural 3D Volume Decomposition

Also called: Neural-PIL, Neural Pre-Integrated Lighting for Reflectance Decomposition

## Overview
Neural 3D Volume Decomposition is a method for jointly estimating 3D shape, spatially-varying BRDF (SVBRDF), and per-image illumination from a collection of images of an object captured under varying viewpoints and lighting conditions [S1, unofficial]. It extends the Neural Radiance Field (NeRF) framework by replacing the implicit view-dependent color output with an explicit physical material representation (Cook-Torrance BRDF parameters) and a latent illumination vector [S1, unofficial]. Unlike standard NeRF which optimizes for novel view synthesis directly, this method solves the inverse problem of reflectance decomposition, enabling high-quality relighting and material-aware view synthesis [S1, unofficial].

## How it works
The method builds on [Radiance-field optimization](44-radiance-field-optimization.md) by introducing a physically-based rendering formulation within the neural volume. It uses a two-stage sampling strategy similar to NeRF but optimizes for material and lighting parameters rather than just color [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   A set of $q$ images $C_j \in \mathbb{R}^{s \times 3}$ for $j \in \{1, \dots, q\}$, where $s$ is the number of pixels per image [S1, unofficial].
*   Per-pixel binary masks $M_j \in \{0, 1\}^{s \times 1}$ indicating which pixels belong to the object [S1, unofficial].
*   Camera poses (viewpoint and orientation) for each image [S1, unofficial].

**Outputs:**
*   A neural 3D volume $V$ that, for any 3D point $x \in \mathbb{R}^3$, outputs:
    *   Cook-Torrance BRDF parameters $b \in \mathbb{R}^7$: diffuse color $b_d \in \mathbb{R}^3$, specular color $b_s \in \mathbb{R}^3$, and roughness $b_r \in \mathbb{R}$ [S1, unofficial].
    *   Unit-length surface normal $n \in \mathbb{R}^3$ [S1, unofficial].
    *   Optical density $\sigma \in \mathbb{R}$ [S1, unofficial].
*   Per-image latent illumination vectors $z_l \in \mathbb{R}^{128}$ for each of the $q$ images [S1, unofficial].

### Data Structures
The neural volume $V$ is implemented as a Multi-Layer Perceptron (MLP) [S1, unofficial].
*   **Input to MLP:** 3D coordinate $x \in \mathbb{R}^3$ and view direction $d \in \mathbb{R}^3$ [S1, unofficial].
*   **Output of MLP:** Concatenated vector $[b_d, b_s, b_r, n, \sigma] \in \mathbb{R}^{3+3+1+3+1} = \mathbb{R}^{11}$ [S1, unofficial].
*   **Illumination Storage:** A learnable tensor of shape $q \times 128$ storing $z_l$ for each image [S1, unofficial].

### Rendering Formulation
The method replaces NeRF's direct color prediction with a rendering equation that integrates BRDF and illumination [S1, unofficial].

1.  **Ray Sampling:** For a camera ray $r(t) = o + td$, sample points $t_i$ along the ray between near bound $t_n$ and far bound $t_f$ [S1, unofficial].
2.  **Neural Query:** For each sample point $t_i$, query the neural volume $V$ to get $b(t_i)$, $n(t_i)$, and $\sigma(t_i)$ [S1, unofficial].
3.  **Illumination Retrieval:** Retrieve the latent illumination vector $z_l$ corresponding to the image index $j$ of the camera ray [S1, unofficial].
4.  **BRDF Evaluation:** Compute the Cook-Torrance BRDF $f_r$ using $b(t_i)$, the surface normal $n(t_i)$, the view direction $d$, and the light direction derived from $z_l$ [S1, unofficial].
5.  **Radiance Integration:** Compute the outgoing radiance $L_o$ at each sample point by integrating the incident illumination (derived from $z_l$) against the BRDF [S1, unofficial].
6.  **Volume Rendering:** Composite the final color $\hat{c}(r)$ using numerical quadrature [S1, unofficial]:

$$
\hat{c}(r) = \sum_{i=1}^{N} T_i \alpha_i L_i
$$

Where:
*   $N$ is the number of samples [S1, unofficial].
*   $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity at sample $i$, with $\delta_i$ being the distance between samples [S1, unofficial].
*   $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance [S1, unofficial].
*   $L_i$ is the radiance at sample $i$, computed from the BRDF and illumination [S1, unofficial].

The transmittance $T(t)$ is defined as [S1, unofficial]:

$$
T(t) = \exp\left(-\int_{t_n}^{t} \sigma(t') dt'\right)
$$

### Optimization
The network is optimized by minimizing the difference between the rendered images and the input images $C_j$ [S1, unofficial]. The loss function typically includes a photometric term comparing $\hat{c}(r)$ to the pixel color in $C_j$ for pixels where $M_j=1$ [S1, unofficial]. The method may also use pre-trained networks (neural-PIL, light-SMAE, BRDF-SMAE) to regularize or initialize the optimization of BRDF and light parameters [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class NeuralVolumeDecomposition:
    def __init__(self, num_images, num_samples_per_ray=64, near=0.1, far=10.0):
        """
        Initialize the decomposition framework.
        
        Args:
            num_images (int): Number of input images q.
            num_samples_per_ray (int): Number of samples N per ray.
            near (float): Near plane distance t_n.
            far (float): Far plane distance t_f.
        """
        self.num_images = num_images
        self.num_samples = num_samples_per_ray
        self.near = near
        self.far = far
        
        # Neural Volume MLP: Input (x, d) -> Output (b_d, b_s, b_r, n, sigma)
        # b_d: 3, b_s: 3, b_r: 1, n: 3, sigma: 1 -> Total 11
        self.mlp = nn.Sequential(
            nn.Linear(6, 256), # 3 for x, 3 for d
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 11)
        )
        
        # Learnable per-image illumination latent vectors z_l in R^128
        self.illumination_latents = nn.Parameter(torch.randn(num_images, 128))
        
    def sample_rays(self, origins, directions):
        """
        Sample points along rays.
        
        Args:
            origins: (q, 3) ray origins o.
            directions: (q, 3) ray directions d.
            
        Returns:
            points: (q, N, 3) sampled 3D points.
            deltas: (q, N) distances between samples.
        """
        # Generate linearly spaced t values between near and far
        t_vals = torch.linspace(self.near, self.far, self.num_samples, device=origins.device)
        # Add noise for stratified sampling (optional, standard in NeRF)
        # t_vals = t_vals + torch.rand_like(t_vals) * (self.far - self.near) / self.num_samples
        
        # Interpolate between t_vals to get points
        # points = origins.unsqueeze(1) + directions.unsqueeze(1) * t_vals.unsqueeze(2)
        # However, standard NeRF sampling often uses uniform or hierarchical.
        # For simplicity, we assume uniform sampling along the ray direction.
        # Note: The source mentions "fixed sampling pattern" for coarse, "high-density" for fine.
        # Here we implement a single pass for clarity, or two passes if hierarchical.
        # The source notes describe the two-MLP structure for sampling, but the decomposition
        # focuses on the output parameters. We will use uniform sampling for the core definition.
        
        # Expand dimensions for broadcasting
        origins_exp = origins.unsqueeze(1) # (q, 1, 3)
        dirs_exp = directions.unsqueeze(1) # (q, 1, 3)
        t_vals_exp = t_vals.unsqueeze(0).unsqueeze(2) # (1, N, 1)
        
        points = origins_exp + dirs_exp * t_vals_exp # (q, N, 3)
        
        # Calculate deltas (distance between samples)
        # delta = t[i+1] - t[i]
        # For the last sample, delta is far - t[N-1]
        deltas = t_vals[1:] - t_vals[:-1] # (N-1,)
        delta_inf = torch.tensor([self.far - t_vals[-1]], device=origins.device)
        deltas = torch.cat([deltas, delta_inf], dim=0) # (N,)
        deltas = deltas.unsqueeze(0).unsqueeze(0).expand(origins.shape[0], self.num_samples, 1) # (q, N, 1)
        
        return points, deltas

    def query_volume(self, points, directions):
        """
        Query the neural volume for material and density parameters.
        
        Args:
            points: (q, N, 3) 3D coordinates.
            directions: (q, 1, 3) view directions (broadcasted to q, N, 3).
            
        Returns:
            b_d: (q, N, 3)
            b_s: (q, N, 3)
            b_r: (q, N, 1)
            n: (q, N, 3)
            sigma: (q, N, 1)
        """
        # Concatenate points and directions
        # points: (q, N, 3), directions: (q, N, 3) -> (q, N, 6)
        inputs = torch.cat([points, directions], dim=-1)
        
        # MLP forward pass
        outputs = self.mlp(inputs) # (q, N, 11)
        
        # Parse outputs
        b_d = outputs[:, :, 0:3]
        b_s = outputs[:, :, 3:6]
        b_r = outputs[:, :, 6:7]
        n = outputs[:, :, 7:10]
        sigma = outputs[:, :, 10:11]
        
        # Normalize normals to unit length
        n = n / (torch.norm(n, dim=-1, keepdim=True) + 1e-6)
        
        return b_d, b_s, b_r, n, sigma

    def compute_radiance(self, b_d, b_s, b_r, n, directions, image_idx):
        """
        Compute radiance L using Cook-Torrance BRDF and latent illumination.
        
        Args:
            b_d: (q, N, 3)
            b_s: (q, N, 3)
            b_r: (q, N, 1)
            n: (q, N, 3)
            directions: (q, N, 3) view directions
            image_idx: (q,) index of the image for each ray batch
            
        Returns:
            L: (q, N, 3) radiance
        """
        # Retrieve illumination latent vector for each ray
        # image_idx is (q,), we need to broadcast to (q, N, 128)
        # Assuming image_idx is repeated for each sample along the ray
        z_l = self.illumination_latents[image_idx] # (q, 128)
        z_l_exp = z_l.unsqueeze(1).expand(-1, self.num_samples, -1) # (q, N, 128)
        
        # Note: The exact mapping from z_l to light direction/intensity is not detailed in the source notes.
        # The source mentions "neural-PIL" and "light-SMAE" networks are pre-trained.
        # For the purpose of this implementation, we assume a placeholder function
        # that maps z_l and view direction to incident radiance.
        # In a real implementation, this would involve the pre-trained neural-PIL network.
        
        # Placeholder for BRDF and Lighting integration
        # L = BRDF(b_d, b_s, b_r, n, view_dir, light_dir_from_z_l)
        # Since the source does not provide the exact formula for z_l -> light,
        # we mark this as [G] (implementation gap) based on standard practices.
        # Typically, z_l might represent spherical harmonics coefficients or environment map indices.
        
        # Simplified placeholder: Assume z_l directly influences the light color/intensity
        # This is a guess [G] to make the code runnable, as the source abstracts this.
        light_color = torch.sigmoid(z_l_exp) # (q, N, 128) -> not 3D, so this is wrong.
        
        # Correct approach for placeholder:
        # The source says "optimize per-image illuminations".
        # We will assume a simple diffuse lighting model for the placeholder,
        # as the full Cook-Torrance integration with neural lighting is complex.
        # L = b_d * light_intensity + b_s * specular_term
        # Let's assume z_l contains a global light direction and intensity.
        
        # For the sake of the article's requirement for a complete implementation,
        # we must define a concrete step. The source does not give the formula.
        # We will use a standard diffuse approximation with z_l as a color multiplier.
        # This is a significant simplification [G].
        
        # Reshape z_l to match spatial dimensions
        # z_l is (q, 128). We need to map 128 to 3D light color.
        # Let's assume a linear projection layer is part of the illumination model.
        if not hasattr(self, 'light_proj'):
            self.light_proj = nn.Linear(128, 3)
            
        light_color = self.light_proj(z_l_exp) # (q, N, 3)
        
        # Cook-Torrance BRDF approximation (simplified for placeholder)
        # F = b_s, D = function(b_r, n, h), G = function(n, v, h)
        # We will use a simple diffuse + specular blend.
        
        # View direction for each sample
        v = directions # (q, N, 3)
        
        # Half vector h = normalize(v + l), where l is light direction
        # Assume light direction is derived from z_l or fixed.
        # Let's assume a fixed light direction for simplicity, modulated by z_l intensity.
        light_dir = torch.tensor([0.0, 1.0, 0.0], device=directions.device).unsqueeze(0).unsqueeze(0) # (1, 1, 3)
        light_dir = light_dir.expand(q, self.num_samples, 3)
        
        h = torch.normalize(v + light_dir, dim=-1)
        
        # Diffuse term
        diffuse = b_d * torch.max(torch.sum(n * light_dir, dim=-1, keepdim=True), torch.zeros_like(n))
        
        # Specular term (simplified Blinn-Phong)
        specular = b_s * torch.pow(torch.max(torch.sum(n * h, dim=-1, keepdim=True), torch.zeros_like(n)), b_r * 100)
        
        L = (diffuse + specular) * light_color
        
        return L

    def render(self, origins, directions, image_idx):
        """
        Render an image from a batch of rays.
        
        Args:
            origins: (q, 3)
            directions: (q, 3)
            image_idx: (q,) integer indices of the images
            
        Returns:
            rendered_color: (q, 3)
        """
        q = origins.shape[0]
        
        # 1. Sample points
        points, deltas = self.sample_rays(origins, directions)
        
        # 2. Query volume
        b_d, b_s, b_r, n, sigma = self.query_volume(points, directions)
        
        # 3. Compute radiance
        L = self.compute_radiance(b_d, b_s, b_r, n, directions, image_idx)
        
        # 4. Volume Rendering
        # Alpha: alpha = 1 - exp(-sigma * delta)
        alpha = 1.0 - torch.exp(-sigma * deltas.squeeze(-1)) # (q, N)
        
        # Transmittance: T = exp(-sum(sigma * delta))
        # Cumulative sum of sigma * delta
        sigma_delta = sigma.squeeze(-1) * deltas.squeeze(-1) # (q, N)
        T = torch.exp(-torch.cumsum(sigma_delta, dim=1)) # (q, N)
        # Shift T for the final term: T_i = exp(-sum_{j<i} sigma_j delta_j)
        # T[:, 0] should be 1.0
        T_shifted = torch.cat([torch.ones(q, 1, device=origins.device), T[:, :-1]], dim=1)
        
        # Composite color: c = sum(T_i * alpha_i * L_i)
        weights = T_shifted * alpha # (q, N)
        weights = weights.unsqueeze(-1) # (q, N, 1)
        
        rendered_color = torch.sum(weights * L, dim=1) # (q, 3)
        
        return rendered_color

    def loss_function(self, rendered, target, mask):
        """
        Compute loss between rendered and target images.
        
        Args:
            rendered: (q, 3)
            target: (q, 3)
            mask: (q, 1) binary mask
            
        Returns:
            loss: scalar
        """
        # Apply mask
        masked_rendered = rendered * mask
        masked_target = target * mask
        
        # L1 or L2 loss
        loss = torch.mean((masked_rendered - masked_target) ** 2)
        
        return loss
```

### Edge Cases and Pitfalls
*   **Degenerate Input:** If the object is not present in an image (mask is all zeros), the loss contribution is zero, but the illumination vector for that image is still optimized. This may lead to unstable illumination estimates if the object is rarely visible [S1, unofficial].
*   **Precision:** The optimization of BRDF parameters (especially roughness $b_r$) can be sensitive to initialization. Pre-training with neural-PIL helps stabilize this [S1, unofficial].
*   **Boundaries:** The near and far bounds $t_n$ and $t_f$ must be chosen to encompass the object. If the object is outside these bounds, it will not be rendered [S1, unofficial].

## Subtopics

### Hierarchical Grid Optimization
Uses a hierarchical grid structure to accelerate the query of the neural volume, reducing memory and computation time compared to a single dense MLP [1414-hierarchical-grid-optimization.md].

### 3D Gaussian Splatting
Represents the scene as a set of 3D Gaussians instead of a neural volume, offering faster rendering but potentially less accurate material decomposition [1795-3d-gaussian-splatting.md].

### Neural Radiance Fields (NeRF)
The foundational method that Neural 3D Volume Decomposition extends. NeRF optimizes for view-dependent color, while this method optimizes for physical material properties [4972-neural-radiance-fields.md].

## Operating characteristics
*   **Time Complexity:** The rendering time is dominated by the MLP queries for each sample point along each ray. With $N$ samples per ray and $R$ rays, the complexity is $O(R \cdot N)$. The optimization time is significantly higher than NeRF due to the additional parameters (BRDF, normals, illumination) and the need for stable gradient-based optimization [S1, unofficial].
*   **Memory Complexity:** The memory usage is dominated by the MLP weights and the learnable illumination vectors. The MLP size is typically small (a few hundred thousand parameters), and the illumination vectors are $q \times 128$. This is comparable to NeRF [S1, unofficial].
*   **Strengths:** Enables high-quality relighting and material-aware view synthesis. Can recover accurate shape and material properties from real-world datasets [S1, unofficial].
*   **Limitations:** Requires images captured under varying illumination conditions. Optimization is complex and may require pre-trained networks for stability. Not suitable for dynamic scenes or single-image input [S1, unofficial].
*   **When to use:** When the goal is to recover physical material properties and illumination from a set of images, rather than just novel view synthesis.
*   **When not to use:** When only novel view synthesis is needed, or when the input images do not have varying illumination.

## Background
Neural 3D Volume Decomposition, also known as Neural-PIL, was introduced in the paper "Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition" [S1, unofficial]. The method addresses the challenge of recovering 3D shape and SVBRDF from images captured under varying lighting conditions, a problem relevant to digitizing real-world objects like statues or landmarks [S1, unofficial]. It builds on the success of NeRF in novel view synthesis and extends it to physical material decomposition [S1, unofficial].

## Sources
[S1] Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition, https://doi.org/10.48550/arxiv.2110.14373, fetched 2026-10-26 (unofficial)
