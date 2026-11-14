# Directed Distance Field (DDF) for Gaussian Splatting

Also called: Directed Distance Function, ray-query oracle

## Overview
The Directed Distance Field (DDF) is a neural implicit representation that serves as a constant-time, constant-memory ray oracle for 3D Gaussian Splatting (3DGS) scenes [S1, unofficial]. It maps a ray (defined by origin $\mathbf{x}$ and direction $\mathbf{d}$) to the distance $t$ to the first surface and a visibility probability $\xi$, enabling secondary ray queries such as shadows, ambient occlusion, and global illumination without iterative sphere tracing or bounding volume hierarchy (BVH) traversal [S1, unofficial]. Unlike sibling methods that scale with the number of Gaussians, the DDF’s query cost and memory footprint remain flat regardless of scene size, making it suitable for large-scale captures where traditional ray-tracing primitives become prohibitive [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A ray defined by origin $\mathbf{x} \in \mathbb{R}^3$ and unit direction $\mathbf{d} \in S^2$.
*   **Output:** A tuple $(t, \xi)$, where $t \in \mathbb{R}^+$ is the distance to the first surface along $\mathbf{d}$, and $\xi \in [0, 1]$ is the probability that the ray intersects the scene geometry [S1, unofficial]. The hit point is computed as $\mathbf{x} + t\mathbf{d}$ when $\xi$ is high [S1, unofficial].

### Data Structures
The DDF is implemented as a neural field with a fixed memory footprint of approximately 52 MB, independent of the number of Gaussians in the scene [S1, unofficial].

1.  **Multiresolution Hash Grid (Point Encoding):**
    *   **Structure:** A hash table mapping coordinate features to a latent vector.
    *   **Levels:** 16 levels.
    *   **Entries per Level:** $2^{19}$ (524,288).
    *   **Base Resolution:** 16.
    *   **Growth Factor:** 1.5.
    *   **Features per Entry:** 2 (concatenated into a 2D vector per level).
    *   **Bounding Box:** A normalized box bounding the object/scene.
    *   **Parameter Count:** ~13 million parameters.
2.  **Directional Encoding:**
    *   **Method:** Sinusoidal positional encoding.
    *   **Bands:** 4 sinusoidal bands.
3.  **Multilayer Perceptron (MLP):**
    *   **Architecture:** 2 layers.
    *   **Width:** 64 neurons per layer.
    *   **Input:** Concatenated hash grid features (from point $\mathbf{x}$) and directional encoding (from $\mathbf{d}$).
    *   **Output:** Two scalars: distance $t$ and visibility $\xi$.
    *   **Parameter Count:** ~8,000 parameters.

### Training Procedure

#### Supervision Data
The DDF is trained using samples of the form $(\mathbf{x}, \mathbf{d}, t_{target}, \xi_{target})$. Three sources of supervision are used [S1, unofficial]:
1.  **GS-Rendered Depth:** Primary rays from the camera provide coarse depth targets.
2.  **Neural Surface Sphere Tracing:** A neural surface (e.g., NeuS) fit to the same images is sphere-traced to provide clean distance targets, which recover thin features lost in Gaussian depth.
3.  **Ground-Truth Mesh Ray-Casting:** For evaluation or upper-bound training, rays are cast against ground-truth meshes.

Additionally, **Self-Consistency Loss** is applied to sharpen thin features. For a ray hitting at distance $t$, a point shifted forward by distance $s$ along the ray is sampled. The DDF evaluated at this shifted point must predict distance $t-s$ [S1, unofficial].

#### Loss Function
The total loss $L$ is a combination of distance and visibility terms [S1, unofficial]:

$$ L = \text{ClampedL1}(t_{pred}, t_{target}) + \lambda \cdot \text{BCE}(\xi_{pred}, \xi_{target}) $$

*   $\text{ClampedL1}$: L1 loss on distance, clamped to a maximum value (box scale) to prevent far-away surfaces from dominating.
*   $\text{BCE}$: Binary Cross-Entropy loss on the hit flag $\xi$.
*   $\lambda$: Weight for the visibility loss, set to 0.3.

#### Optimization Parameters
*   **Optimizer:** Adam.
*   **Batch Size:** 16,384 rays.
*   **Learning Rate:** $3 \times 10^{-3}$, decayed by cosine schedule to 5% of initial value.
*   **Training Steps:** 30,000.
*   **Data Augmentation:** 30% of each batch is replaced with ray-march shifted points for self-consistency supervision.
*   **Hardware:** Single NVIDIA A100 GPU.
*   **Inference:** Reduced precision with a compiled forward pass (e.g., via `tinycudann`) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class DirectedDistanceField:
    """
    Implementation of the Directed Distance Field (DDF) for Gaussian Splatting.
    Maps ray origin x and direction d to distance t and visibility xi.
    """
    def __init__(self, num_levels=16, level_size=2**19, base_resolution=16, 
                 growth_factor=1.5, features_per_level=2, num_directional_bands=4, 
                 mlp_width=64, mlp_layers=2):
        """
        Initialize the DDF network.
        """
        super().__init__()
        self.num_levels = num_levels
        self.level_size = level_size
        self.features_per_level = features_per_level
        self.num_directional_bands = num_directional_bands
        self.mlp_width = mlp_width
        self.mlp_layers = mlp_layers
        
        # Multiresolution Hash Grid Encoding
        # Note: In practice, this uses a CUDA kernel (e.g., tinycudann) for 
        # efficient hashing and interpolation. Here we simulate the structure.
        self.hash_grid = nn.EmbeddingBag(
            num_embeddings=num_levels * level_size, 
            embedding_dim=features_per_level, 
            mode='sum'
        )
        
        # Directional Encoding (Sinusoidal)
        # 4 bands means 8 frequencies (sin/cos) per dimension? 
        # Source says "4 sinusoidal bands". Typically implies 4 frequencies.
        # Assuming standard positional encoding: 2 * num_bands * 3 dims = 24 features.
        # However, source says "4 sinusoidal bands" for direction. 
        # We will assume a standard sinusoidal encoder for 3D direction.
        self.directional_encoder = SinusoidalEncoder(bands=num_directional_bands)
        
        # MLP
        layers = []
        input_dim = (num_levels * features_per_level) + self.directional_encoder.output_dim
        for i in range(mlp_layers):
            layers.append(nn.Linear(input_dim, mlp_width))
            layers.append(nn.ReLU())
            input_dim = mlp_width
        layers.append(nn.Linear(mlp_width, 2)) # Output: t, xi
        self.mlp = nn.Sequential(*layers)

    def forward(self, x, d):
        """
        Query the DDF.
        Args:
            x: Tensor of shape (N, 3), ray origins.
            d: Tensor of shape (N, 3), ray directions (unit vectors).
        Returns:
            t: Tensor of shape (N,), distance to surface.
            xi: Tensor of shape (N,), visibility probability.
        """
        # 1. Encode Point x using Multiresolution Hash Grid
        # The hash grid maps coordinates to features across multiple resolutions.
        # For implementation, this typically involves:
        # - Normalizing x to [0, 1] or within the bounding box.
        # - For each level l, computing grid coordinates and hashing to table index.
        # - Interpolating features from the 8 corners of the grid cell.
        # - Summing/concatenating features from all levels.
        # 
        # Since standard PyTorch does not have a built-in multiresolution hash grid,
        # we assume a helper function `hash_grid_encode` exists or is implemented 
        # via CUDA. Here we denote the output feature vector.
        point_features = hash_grid_encode(x, self.num_levels, self.level_size, 
                                          self.features_per_level, self.base_resolution, 
                                          self.growth_factor)
        
        # 2. Encode Direction d using Sinusoidal Bands
        direction_features = self.directional_encoder(d)
        
        # 3. Concatenate Features
        combined_features = torch.cat([point_features, direction_features], dim=-1)
        
        # 4. Pass through MLP
        output = self.mlp(combined_features)
        
        # 5. Extract t and xi
        t = output[:, 0]
        xi = torch.sigmoid(output[:, 1]) # Ensure xi is in [0, 1]
        
        return t, xi

class SinusoidalEncoder(nn.Module):
    def __init__(self, bands=4):
        super().__init__()
        self.bands = bands
        # 4 bands -> 4 frequencies. Each frequency has sin and cos.
        # 3 dimensions * 4 frequencies * 2 (sin/cos) = 24 features.
        self.output_dim = 3 * bands * 2

    def forward(self, x):
        # x shape: (N, 3)
        # Frequencies: 2^0, 2^1, 2^2, 2^3
        frequencies = torch.tensor([2**i for i in range(self.bands)], dtype=x.dtype, device=x.device)
        # Expand x to (N, 3, 1) and frequencies to (1, 1, bands)
        # Result: (N, 3, bands)
        scaled_x = x.unsqueeze(-1) * frequencies.unsqueeze(0).unsqueeze(0)
        # Compute sin and cos
        sin_vals = torch.sin(scaled_x)
        cos_vals = torch.cos(scaled_x)
        # Concatenate along last dim: (N, 3, 2*bands) -> (N, 6*bands)
        encoded = torch.cat([sin_vals, cos_vals], dim=-1).view(x.shape[0], -1)
        return encoded

def hash_grid_encode(x, num_levels, level_size, features_per_level, base_resolution, growth_factor):
    """
    Placeholder for the actual multiresolution hash grid encoding.
    In the source, this is implemented via tinycudann or similar CUDA kernel.
    This function simulates the output dimensionality.
    """
    # The output of the hash grid encoding is a vector of size:
    # num_levels * features_per_level
    # For the DDF: 16 levels * 2 features = 32 features.
    # Note: The actual implementation involves hashing coordinates to indices 
    # and interpolating between grid vertices.
    N = x.shape[0]
    # Return dummy features of the correct size for the MLP input
    return torch.zeros(N, num_levels * features_per_level, device=x.device)

def compute_loss(t_pred, xi_pred, t_target, xi_target, clamp_scale=1.0, lambda_bce=0.3):
    """
    Compute the DDF training loss.
    """
    # Clamped L1 loss on distance
    l1_error = torch.abs(t_pred - t_target)
    clamped_l1 = torch.clamp(l1_error, max=clamp_scale)
    loss_l1 = torch.mean(clamped_l1)
    
    # BCE loss on visibility
    loss_bce = torch.nn.functional.binary_cross_entropy(xi_pred, xi_target)
    
    # Total loss
    loss = loss_l1 + lambda_bce * loss_bce
    return loss
```

### Edge Cases and Pitfalls
*   **Thin Feature Loss:** Supervising solely on 3DGS-rendered depth blurs thin parts (e.g., mug handles, animal legs) because soft Gaussians average narrow structures. Clean distance supervision (from neural surfaces or meshes) is required to recover these features [S1, unofficial].
*   **Compact Objects:** For compact objects where Gaussian depth already captures geometry, clean distance supervision may not help and can hurt if the sphere-traced point cloud is noisy, causing "fattening" [S1, unofficial].
*   **Direct Image Supervision:** Learning directly from volume-rendered color loss does not recover thin parts because most sampled rays hit the background, providing a weak gradient signal for distance output [S1, unofficial].
*   **Ray Origin Dependency:** The DDF is accurate primarily for ray origins on or near the surface. Rays starting far from the surface (e.g., ground plane below an object) may report false occlusion [S1, unofficial].
*   **Ambient Occlusion Accuracy:** Ambient occlusion (21.3 dB) is weaker than shadows (30.3 dB) because averaging many rays accumulates small per-ray errors [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** $O(1)$ per ray query. The DDF requires a single forward pass through the network, regardless of the number of Gaussians $N$ [S1, unofficial].
*   **Memory Complexity:** $O(1)$. The field size is fixed at ~52 MB, independent of scene size [S1, unofficial].

### Measured Performance
*   **Speed vs. SDF:** The DDF is 26 to 72 times faster than sphere tracing an equivalent neural Signed Distance Field (SDF) [S1, unofficial].
*   **Speed vs. BVH:** 
    *   On small scenes (few thousand Gaussians), an RT-core BVH is faster per ray due to hardware optimization.
    *   On large scenes (hundreds of thousands to millions of Gaussians), the DDF becomes competitive or faster. At 1M Gaussians, the DDF (with `tinycudann`) achieves 0.005 µs/ray vs. 0.009 µs/ray for RT-core BVH [S1, unofficial].
    *   BVH latency rises by ~9x from 5K to 1M Gaussians, while DDF latency remains flat [S1, unofficial].
*   **Speed vs. Rasterization:** For sparse queries (1K-10K rays), the DDF is 600+ times faster than re-rasterizing with `gsplat`. At a full frame (1M rays), it ties rasterization (4.81 ms vs. 4.45 ms) because rasterization is bound by frame buffer writing [S1, unofficial].
*   **Quality:** 
    *   Shadows: 30.3 dB PSNR against reference ray tracer.
    *   Ambient Occlusion: 21.3 dB PSNR.
    *   Geometry: Chamfer distance improves from 0.082 to 0.065 (21% drop) with clean distance supervision [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** Constant time and memory; no RT-core hardware required; mesh-free pipeline; recovers thin features with clean supervision; scales to millions of Gaussians.
*   **Limitations:** Ambient occlusion quality is lower than shadows; accuracy degrades for ray origins far from the training surface; mesh-free version is limited by the smoothness of the underlying neural surface [S1, unofficial].

## Background

### Origin and History
The Directed Distance Field for Gaussian Splatting was introduced in the paper "Directed Distance Fields for Constant-Time Ray Queries on Gaussian Splatting" [S1, unofficial]. It addresses the limitation of 3DGS rasterization, which only supports camera rays, by distilling the Gaussian scene into a neural field that acts as a secondary-ray oracle. The method utilizes multiresolution hash grid encoding from `tinycudann` [S1, unofficial] to achieve high performance and compact memory usage.

### Usage
The DDF is used as a secondary-ray oracle for global illumination tasks, including shadows and ambient occlusion, in 3D Gaussian Splatting renderers. It enables mesh-free pipelines where geometry is inferred directly from images via Gaussian splatting and neural surface fitting, avoiding the need for explicit mesh reconstruction [S1, unofficial].

## Sources

[S1] Directed Distance Fields for Constant-Time Ray Queries on Gaussian Splatting, https://arxiv.org/abs/2606.00817v1, fetched 2026-10-26 (unofficial)
