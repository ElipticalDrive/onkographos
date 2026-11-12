# Neural Light Field (NeLF)

Also called: R2L, Neural Light Field Network, Deep Residual NeLF

## Overview
Neural Light Field (NeLF) represents a 3D scene as a direct mapping from camera ray parameters to final pixel color, bypassing the volumetric ray-marching and alpha-composition steps required by Neural Radiance Fields (NeRF) [S2]. Unlike NeRF, which predicts density and radiance at sampled points along a ray requiring multiple forward passes per pixel, NeLF performs a single forward pass per ray to output the integrated RGB value directly [S2]. This approach enables orders-of-magnitude faster rendering speeds and reduced computational complexity (FLOPs) while maintaining or exceeding the visual quality of NeRF [S2]. The method is distinguished by its use of a deep residual Multi-Layer Perceptron (MLP) architecture and knowledge distillation from a pre-trained NeRF teacher model to handle the data scarcity inherent in training such deep networks [S2].

## How it works

### Inputs and Outputs
*   **Input:** A 4D oriented ray, parameterized by its origin $\mathbf{o} = (x_o, y_o, z_o)$ and normalized direction $\mathbf{d} = (x_d, y_d, z_d)$ [S2]. In the specific implementation (R2L), the input vector is formed by concatenating the spatial coordinates of $K$ sampled points along the ray, resulting in a $3K$-dimensional vector [S2].
*   **Output:** A 3D RGB color vector $\mathbf{c} = (r, g, b)$ representing the final pixel color for that ray [S2].

### Data Structures and Layout
The core data structure is the Deep Residual MLP network $G_\phi$ [S2].
*   **Network Topology:** The network consists of $D$ layers with width $W$. The standard configuration uses extensive residual blocks in internal layers [S2].
    *   **Depth ($D$):** 88 layers [S2].
    *   **Width ($W$):** 256 (for 12M FLOPs budget) or 181 (for 6M FLOPs budget) [S2].
    *   **Activation:** Standard residual block activations (typically ReLU) are used, with the final layer outputting RGB values [S2].
*   **Input Representation:**
    *   **Point Sampling:** $K=16$ points are sampled along the ray [S2].
    *   **Vector Construction:** The input vector $\mathbf{x}_{in}$ is the concatenation of the 3D coordinates of these $K$ points: $\mathbf{x}_{in} = [x_1, y_1, z_1, \dots, x_K, y_K, z_K]$ [S2].
    *   **Positional Encoding:** Input coordinates are enriched using positional encoding $\gamma(\cdot)$ as defined in the parent article [S1][S3].

### Numbered Steps

1.  **Ray Sampling:**
    *   **Training:** Randomly sample $K=16$ points along the ray using stratified sampling to prevent overfitting to specific locations [S2].
    *   **Testing:** Evenly space $K=16$ points along the ray [S2].
2.  **Input Construction:** Concatenate the spatial coordinates of the $K$ sampled points to form the $3K$-dimensional input vector [S2]. Apply positional encoding to the coordinates [S2].
3.  **Forward Pass:** Feed the encoded input vector into the deep residual MLP $G_\phi$. The network processes the input through 88 layers with extensive residual connections [S2].
4.  **Output:** The network outputs the predicted RGB color $\mathbf{c}_{pred}$ directly [S2].

### Formulas

**NeLF Mapping:**
The NeLF network $G_\phi$ maps the ray parameters to color:
$$
\mathbf{c}_{pred} = G_\phi(\mathbf{o}, \mathbf{d})
$$
where $\mathbf{o} \in \mathbb{R}^3$ and $\mathbf{d} \in \mathbb{R}^3$ (normalized) [S2].

**Loss Function (Distillation):**
The network is trained by minimizing the Mean Squared Error (MSE) between the NeLF output and the target RGB value $\hat{\mathbf{c}}$ provided by the pre-trained NeRF teacher model $F_{\Theta^*}$ [S2]:
$$
\mathcal{L} = \text{MSE}(G_\phi(\mathbf{o}, \mathbf{d}), \hat{\mathbf{c}})
$$
where $\hat{\mathbf{c}} = F_{\Theta^*}(\mathbf{o}, \mathbf{d})$ [S2].

**Pseudo Data Generation:**
Ray origins and directions are sampled from a uniform distribution $U$ within a viewing bounding box inferred from training camera poses [S2]:
$$
\begin{aligned}
x_o, y_o, z_o &\sim U(x_{min}, x_{max}) \\
x_d, y_d, z_d &\sim U(x_{d,min}, x_{d,max})
\end{aligned}
$$
[ S2 ]

### Parameters and Typical Values
*   **$K$ (Sample Points):** 16 [S2].
*   **Network Width ($W$):** 256 or 181 [S2].
*   **Network Depth ($D$):** 88 [S2].
*   **Batch Size:** 98,304 rays per batch [S2].
*   **Learning Rate:** Initial $5 \times 10^{-4}$, decayed during training [S2].
*   **Optimizer:** Adam [S2].
*   **Hard Example Ratio ($r$):** 0.2 (fraction of hard examples selected per batch) [S2].
*   **Pseudo Data Size ($S$):** 10,000 images (performance saturates around 5,000) [S2].
*   **Iterations:** 1,000k (1,000,000) [S2].

### Edge Cases and Pitfalls
*   **Trainability of Deep Networks:** Without extensive residual connections, deep NeLF networks (e.g., 88 layers) are barely trainable [S2].
*   **Data Scarcity:** Training a deep NeLF on original captured images alone leads to severe overfitting and poor generalization (e.g., PSNR of 19.81 on Lego scene with 0.1k images) [S2]. Pseudo data from a teacher model is critical [S2].
*   **Reflection Surfaces:** Unlike methods relying on ground-truth depth (e.g., DONeRF), NeLF does not suffer from failures on reflective surfaces (water, mirrors) where depth is ambiguous [S2].
*   **Input Representation:** Concatenating spatial coordinates of sampled points yields better quality (PSNR 29.50) than using Plücker coordinates (PSNR 29.08) on the Lego scene [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

class ResidualBlock(nn.Module):
    def __init__(self, width):
        super(ResidualBlock, self).__init__()
        self.fc1 = nn.Linear(width, width)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(width, width)
        
    def forward(self, x):
        identity = x
        out = self.fc1(x)
        out = self.relu(out)
        out = self.fc2(out)
        out += identity
        out = self.relu(out)
        return out

class NeLF(nn.Module):
    def __init__(self, width=256, depth=88):
        super(NeLF, self).__init__()
        # Input dimension: 3 * K (K=16) = 48
        self.input_dim = 48
        self.width = width
        self.depth = depth
        
        # Initial linear layer
        self.input_layer = nn.Linear(self.input_dim, width)
        self.relu = nn.ReLU()
        
        # Deep residual blocks
        self.residual_blocks = nn.ModuleList([
            ResidualBlock(width) for _ in range(depth - 1)
        ])
        
        # Output layer
        self.output_layer = nn.Linear(width, 3) # RGB

    def forward(self, x):
        # x shape: (batch_size, 3*K)
        x = self.input_layer(x)
        x = self.relu(x)
        
        for block in self.residual_blocks:
            x = block(x)
            
        rgb = self.output_layer(x)
        return rgb

def positional_encoding(x, L=10):
    """
    Standard positional encoding as defined in NeRF.
    x: tensor of shape (..., D)
    Returns: tensor of shape (..., D * (2*L + 1))
    """
    # Implementation details omitted for brevity, assumes standard NeRF PE
    # See parent article [S1][S3] for full implementation
    pass

def stratified_sampling(ray_near, ray_far, K=16):
    """
    Sample K points along a ray using stratified sampling.
    ray_near, ray_far: tensors of shape (batch_size, 3)
    Returns: points of shape (batch_size, K, 3)
    """
    # Generate random offsets
    t = torch.rand(ray_near.shape[0], K, device=ray_near.device)
    # Stratified sampling: divide interval into K bins
    t = (t + torch.arange(K, device=ray_near.device)) / K
    # Map to [0, 1]
    t = t.clamp(0, 1)
    # Interpolate points
    points = ray_near.unsqueeze(1) + t.unsqueeze(2) * (ray_far.unsqueeze(1) - ray_near.unsqueeze(1))
    return points

def generate_pseudo_data(teacher_nerf, num_samples=10000, bbox_min=None, bbox_max=None):
    """
    Generate pseudo data by sampling rays and querying the teacher NeRF.
    """
    # Sample origins and directions uniformly within bounding box
    # Implementation depends on specific bbox inference logic
    # See [S2] for distribution details
    pass

def train_nerf(teacher_nerf, student_nerf, dataloader, num_iterations=1000000, r=0.2):
    """
    Train NeLF via distillation from NeRF.
    """
    optimizer = optim.Adam(student_nerf.parameters(), lr=5e-4)
    hard_example_pool = []
    
    for iteration in range(num_iterations):
        # 1. Sample rays from training images
        rays_o, rays_d, gt_rgb = next(dataloader)
        
        # 2. Sample points along rays (Stratified)
        # Assume ray_near and ray_far are derived from rays_o, rays_d and scene bounds
        points = stratified_sampling(ray_near, ray_far, K=16)
        
        # 3. Flatten points for input
        # points shape: (batch_size, K, 3) -> (batch_size, 3*K)
        input_vecs = points.reshape(points.shape[0], -1)
        
        # 4. Apply Positional Encoding
        encoded_input = positional_encoding(input_vecs)
        
        # 5. Forward pass through Student NeLF
        pred_rgb = student_nerf(encoded_input)
        
        # 6. Query Teacher NeRF for targets
        # Teacher takes 6D input (origin + direction)
        # Note: Teacher implementation details vary, assuming standard NeRF query
        teacher_rgb = teacher_nerf(rays_o, rays_d)
        
        # 7. Compute Loss
        loss = torch.mean((pred_rgb - teacher_rgb) ** 2)
        
        # 8. Hard Example Mining
        # Calculate per-sample loss
        per_sample_loss = torch.mean((pred_rgb - teacher_rgb) ** 2, dim=1)
        
        # Sort losses
        sorted_indices = torch.argsort(per_sample_loss)
        num_hard = int(r * len(per_sample_loss))
        
        # Add top hard examples to pool
        hard_indices = sorted_indices[-num_hard:]
        hard_example_pool.extend(hard_indices.tolist())
        
        # Sample from pool to augment batch
        if len(hard_example_pool) > 0:
            # Randomly pick r% from pool
            pool_size = len(hard_example_pool)
            num_from_pool = int(r * len(per_sample_loss))
            pool_sample_indices = torch.randint(0, pool_size, (num_from_pool,))
            hard_indices_from_pool = [hard_example_pool[i] for i in pool_sample_indices]
            
            # Combine original batch and hard examples
            # This logic simplifies the augmentation process described in [S2]
            # In practice, this requires rebatching
            pass
            
        # 9. Backpropagate
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Clear pool periodically or manage size
        if iteration % 1000 == 0:
            hard_example_pool = []

```

## Operating characteristics

*   **Time Complexity:** $O(1)$ per ray in terms of sampling iterations, as it requires only a single forward pass through the MLP [S2]. This contrasts with NeRF's $O(N)$ where $N$ is the number of samples per ray (typically 256) [S2].
*   **Memory Complexity:** The memory footprint is determined by the size of the single deep MLP. For W256D88, it is approximately 23.7 MB [S2]. This is comparable to or slightly larger than standard NeRF models but significantly smaller than grid-based methods [S2].
*   **Performance:**
    *   **Speedup:** Achieves 28–31× wall-time speedup over NeRF on synthetic scenes [S2].
    *   **FLOPs:** Reduces FLOPs per ray by 26–35× compared to NeRF [S2].
    *   **Quality:** Delivers 1.4–2.8 dB average PSNR improvement over NeRF on synthetic datasets [S2].
*   **Strengths:**
    *   Real-time rendering capability due to single-pass inference [S1][S2].
    *   No reliance on ground-truth depth, making it robust for reflective surfaces [S2].
    *   High quality on complex real-world scenes [S2].
*   **Limitations:**
    *   Requires a pre-trained NeRF teacher model for distillation [S2].
    *   Training requires a large amount of pseudo data (10k images) to prevent overfitting [S2].
    *   Performance is limited by the teacher model's quality if real data is not included in training [S2].

## Background

*   **Origin:** The concept of Neural Light Fields (NeLF) was introduced to address the computational bottleneck of NeRF by mapping rays directly to color [S2]. The specific deep residual implementation (R2L) was proposed by Wang et al. in 2022 [S2].
*   **Development:** Early NeLF works used shallow MLPs (6-8 layers) and achieved limited speedups (~5×) [S2]. The R2L method demonstrated that deep networks (88 layers) with residual connections, combined with knowledge distillation, could achieve both high quality and significant speedups [S2].
*   **Usage:** NeLF is used for efficient novel view synthesis (NVS) in computer graphics and vision, particularly in applications requiring real-time rendering or reduced computational resources [S2]. It is also applied in head avatar modeling (e.g., LightAvatar) to eliminate reliance on explicit meshes [S1].

## Sources

[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis, https://doi.org/10.1007/978-3-031-19821-2_35, fetched 2026-10-26
