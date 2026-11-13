# D-NeRF

Also called: Deformable Neural Radiance Fields, Dynamic NeRF, Neural Radiance Fields for Dynamic Scenes

## Overview
D-NeRF is a method for synthesizing novel views at an arbitrary point in time of dynamic scenes with complex non-rigid geometries, optimizing an underlying deformable volumetric function from a sparse set of input monocular views without ground-truth geometry or multi-view images [S1]. It sets itself apart from naive extensions of NeRF (which map $(x, y, z, t)$ directly to density and radiance) by decomposing the learning process into two modules: a deformation network that maps dynamic coordinates to a static canonical space, and a canonical radiance network that predicts density and color from that space [S1, S3]. This decomposition effectively exploits temporal redundancy, allowing the model to handle rigid, articulated, and non-rigid motions from a single camera moving around the scene [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse set of $T$ RGB images $\{I_t\}_{t=1}^T$ captured by a monocular camera, along with their corresponding camera pose matrices $\{T_t\}_{t=1}^T \in SE(3)$ [S1, S3]. Each image $I_t \in \mathbb{R}^{H \times W \times 3}$ corresponds to a specific time instant $t$. The method assumes a single view per time instant [S1].
*   **Outputs:** A continuous 6D neural radiance field that can render novel images at any camera viewpoint $(\theta, \phi)$ and any time $t$ [S1].

**Data Structures and Network Architecture:**
The model consists of two deep fully connected networks (MLPs) without convolutional layers [S1, S3]:

1.  **Deformation Network ($\Psi_t$):**
    *   **Function:** Maps a 3D point $x$ at time $t$ to a displacement $\Delta x$ that transforms the point to its canonical configuration [S1, S3].
    *   **Input:** Encoded spatial coordinate $x$ and time $t$.
    *   **Output:** Displacement vector $\Delta x \in \mathbb{R}^3$.
    *   **Architecture:** 8-layer MLP with ReLU activations. No non-linearity is applied to the output $\Delta x$ [S1, S3].
    *   **Constraint:** The canonical configuration is defined as the scene at $t=0$. Therefore, $\Psi_t(x, 0) = 0$ [S1, S3].

2.  **Canonical Network ($\Psi_x$):**
    *   **Function:** Regresses volume density $\sigma$ and view-dependent RGB color $c$ from the canonical coordinate and viewing direction [S1, S3].
    *   **Input:** Encoded canonical coordinate $x' = x + \Delta x$ and viewing direction $d$.
    *   **Output:** Density $\sigma \in \mathbb{R}$ and color $c \in \mathbb{R}^3$.
    *   **Architecture:** 8-layer MLP with ReLU activations. A final sigmoid non-linearity is applied to the output color $c$ and density $\sigma$ [S1, S3].

**Positional Encoding:**
Raw coordinates are encoded into a higher-dimensional space using positional encoding $\gamma(p)$ [S1, S3]:
$$ \gamma(p) = \langle (\sin(2^l \pi p), \cos(2^l \pi p)) \rangle_{l=0}^{L-1} $$
The encoder is applied independently to each coordinate and camera view component [S1].
*   For spatial coordinates $x$: $L = 10$ [S1, S3].
*   For viewing direction $d$ and time $t$: $L = 4$ [S1, S3].

**Rendering Process:**
Given a point $x$ and viewing direction $d$ at time $t$, the rendering follows these steps:
1.  **Deformation:** Transform the point position to its canonical configuration using the deformation network:
    $$ x' = x + \Psi_t(x, t) $$
    where $x$ is the point on the camera ray $x(h) = o + hd$ [S1, S3].
2.  **Query:** Query the canonical network with the transformed point $x'$ and direction $d$ to obtain density and color:
    $$ [c(x', d), \sigma(x')] = \Psi_x(x', d) $$
3.  **Volume Rendering:** Approximate the volume rendering integral via numerical quadrature using stratified sampling [S1, S3]. Draw $N$ samples $\{h_n\}_{n=1}^N$ from evenly-spaced ray bins in $[h_n, h_f]$ [S1]. The expected color $C(p, t)$ for a pixel $p$ at time $t$ is:
    $$ C(p, t) = \int_{h_n}^{h_f} T(h, t) \sigma(p(h, t)) c(p(h, t), d) dh $$
    where $p(h, t) = x(h) + \Psi_t(x(h), t)$ is the deformed point [S3].
    The transmittance $T(h, t)$ is:
    $$ T(h, t) = \exp \left( - \int_{h_n}^{h} \sigma(p(s, t)) ds \right) $$
    The discrete approximation for the rendered pixel color $C'(p, t)$ is:
    $$ C'(p, t) = \sum_{n=1}^{N} T'(h_n, t) \alpha(h_n, t, \delta_n) c(p(h_n, t), d) $$
    where $\alpha(h, t, \delta) = 1 - \exp(-\sigma(p(h, t))\delta)$ is the opacity, $T'(h_n, t) = \exp \left( - \sum_{m=1}^{n-1} \sigma(p(h_m, t))\delta_m \right)$ is the accumulated transmittance, and $\delta_n = h_{n+1} - h_n$ is the distance between quadrature points [S1].

**Training:**
The parameters of $\Psi_x$ and $\Psi_t$ are learned simultaneously by minimizing the mean squared error (MSE) between rendered pixels $C'$ and ground truth pixels $\hat{C}$ [S1, S3]:
$$ \text{Loss} = \frac{1}{N_s} \sum_{i=1}^{N_s} \| \hat{C}(p_i, t) - C'(p_i, t) \|^2 $$
where $N_s$ is the number of rays in the batch [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def positional_encoding(p, L):
    """
    Applies positional encoding to input p.
    p: numpy array of shape (..., D)
    L: number of frequency bands
    Returns: numpy array of shape (..., D * 2 * L)
    """
    # p is expected to be a float array
    scales = np.array([2**l for l in range(L)])
    # Reshape for broadcasting: p shape (..., D), scales shape (L,)
    # We want to compute sin(2^l * pi * p) and cos(2^l * pi * p) for each l
    # Result shape: (..., D, 2, L) -> flatten last dims to (..., 2*D*L)
    
    # Efficient implementation using broadcasting
    # p[..., None, :] shape (..., 1, D)
    # scales[:, None] shape (L, 1)
    # product shape (..., L, D)
    freqs = p[..., None, :] * (2 * np.pi * scales[None, :, None])
    
    sin_vals = np.sin(freqs)
    cos_vals = np.cos(freqs)
    
    # Concatenate sin and cos along the last dimension
    # sin_vals shape (..., L, D), cos_vals shape (..., L, D)
    # We need to interleave or stack. The formula implies interleaving per dimension?
    # "<(sin..., cos...)>_0^L" usually means for each dimension, append sin/cos for each freq.
    # Standard NeRF implementation:
    # p_embed = np.concatenate([np.sin(p * freqs), np.cos(p * freqs)], axis=-1)
    # But here freqs are per dimension.
    
    # Let's follow the standard NeRF embedding logic which matches the formula:
    # For each dimension d, and each freq l, add sin and cos.
    # Reshape p to (..., D, 1)
    p_expanded = p[..., :, None] # (..., D, 1)
    freqs_expanded = scales[None, None, :] # (1, 1, L)
    
    freqs_all = p_expanded * (2 * np.pi * freqs_expanded) # (..., D, L)
    
    sin_all = np.sin(freqs_all) # (..., D, L)
    cos_all = np.cos(freqs_all) # (..., D, L)
    
    # Interleave D and L? No, usually it's (..., D*2*L) where for each dim, we have L sin/cos pairs.
    # Reshape sin_all to (..., D*L) and cos_all to (..., D*L) then concat?
    # Or reshape to (..., D, 2, L) and permute?
    # Standard: concat along last axis after reshaping D and L together.
    
    sin_reshaped = sin_all.reshape(*p.shape[:-1], -1) # (..., D*L)
    cos_reshaped = cos_all.reshape(*p.shape[:-1], -1) # (..., D*L)
    
    return np.concatenate([sin_reshaped, cos_reshaped], axis=-1)

class DeformationNetwork:
    def __init__(self, hidden_dim=256):
        # 8-layer MLP
        self.layers = []
        # Input dim: encoded x (3 * 2 * 10 = 60) + encoded t (1 * 2 * 4 = 8) = 68
        # Note: t is scalar, so encoded t is 8-dim. x is 3-dim, encoded x is 60-dim.
        self.input_dim_x = 3 * 2 * 10
        self.input_dim_t = 1 * 2 * 4
        self.input_dim = self.input_dim_x + self.input_dim_t
        
        # Initialize weights and biases for 8 layers
        # Layer 0: input -> hidden
        # Layers 1-6: hidden -> hidden
        # Layer 7: hidden -> output (3)
        self.weights = []
        self.biases = []
        
        # He initialization for ReLU
        # Weights shape: (in_dim, out_dim)
        self.weights.append(np.random.randn(self.input_dim, hidden_dim) * np.sqrt(2.0 / self.input_dim))
        self.biases.append(np.zeros(hidden_dim))
        
        for _ in range(6):
            self.weights.append(np.random.randn(hidden_dim, hidden_dim) * np.sqrt(2.0 / hidden_dim))
            self.biases.append(np.zeros(hidden_dim))
            
        self.weights.append(np.random.randn(hidden_dim, 3) * np.sqrt(2.0 / hidden_dim))
        self.biases.append(np.zeros(3))

    def forward(self, x, t):
        # x: (..., 3), t: (..., 1)
        enc_x = positional_encoding(x, 10)
        enc_t = positional_encoding(t, 4)
        input_vec = np.concatenate([enc_x, enc_t], axis=-1)
        
        h = input_vec
        for i in range(7): # First 7 layers (0-6)
            h = np.maximum(0, h @ self.weights[i] + self.biases[i]) # ReLU
        
        # Last layer: no non-linearity
        delta_x = h @ self.weights[7] + self.biases[7]
        return delta_x

class CanonicalNetwork:
    def __init__(self, hidden_dim=256):
        # 8-layer MLP
        self.layers = []
        # Input dim: encoded x' (60) + encoded d (3 * 2 * 4 = 24) = 84
        self.input_dim_x = 3 * 2 * 10
        self.input_dim_d = 3 * 2 * 4
        self.input_dim = self.input_dim_x + self.input_dim_d
        
        self.weights = []
        self.biases = []
        
        self.weights.append(np.random.randn(self.input_dim, hidden_dim) * np.sqrt(2.0 / self.input_dim))
        self.biases.append(np.zeros(hidden_dim))
        
        for _ in range(6):
            self.weights.append(np.random.randn(hidden_dim, hidden_dim) * np.sqrt(2.0 / hidden_dim))
            self.biases.append(np.zeros(hidden_dim))
            
        self.weights.append(np.random.randn(hidden_dim, 4) * np.sqrt(2.0 / hidden_dim)) # 3 for color, 1 for density
        self.biases.append(np.zeros(4))

    def forward(self, x_prime, d):
        enc_x = positional_encoding(x_prime, 10)
        enc_d = positional_encoding(d, 4)
        input_vec = np.concatenate([enc_x, enc_d], axis=-1)
        
        h = input_vec
        for i in range(7):
            h = np.maximum(0, h @ self.weights[i] + self.biases[i])
            
        output = h @ self.weights[7] + self.biases[7]
        # Sigmoid on color and density
        color = 1.0 / (1.0 + np.exp(-output[..., :3]))
        density = 1.0 / (1.0 + np.exp(-output[..., 3:4]))
        return color, density

def render_ray(ray_origin, ray_direction, near, far, deformation_net, canonical_net, num_samples=64):
    """
    Renders a single ray.
    """
    # Stratified sampling
    # Draw N samples from evenly-spaced ray bins
    bins = np.linspace(near, far, num_samples + 1)
    # Add noise to each sample within its bin
    u = np.random.rand(num_samples)
    samples = bins[:-1] + u * (bins[1:] - bins[:-1])
    
    # Points on the ray
    points = ray_origin + samples[:, None] * ray_direction # (N, 3)
    
    # Deform points to canonical space
    # deformation_net expects x and t. t is scalar for the whole ray.
    # We need to pass t to deformation_net.
    # Assuming t is passed to the render function or is global.
    # For this pseudocode, let's assume t is available.
    # delta_x = deformation_net.forward(points, t)
    # Since t is constant for the ray, we can broadcast.
    # But deformation_net takes x and t.
    
    # Let's assume t is passed as a scalar
    # delta_x = deformation_net.forward(points, np.full((num_samples, 1), t))
    
    # For the sake of the pseudocode structure, we'll assume t is a variable in scope
    # delta_x = deformation_net.forward(points, t) 
    
    # Canonical points
    # x_prime = points + delta_x
    
    # Query canonical network
    # color, density = canonical_net.forward(x_prime, ray_direction)
    
    # Volume rendering
    # delta_z = samples[1:] - samples[:-1]
    # delta_z = np.append(delta_z, 0) # Last delta is 0 or handled by alpha
    
    # alpha = 1 - exp(-density * delta_z)
    # T = exp(-sum(sigma * delta_z))
    # C = sum(T * alpha * color)
    
    pass

def train_step(images, poses, deformation_net, canonical_net, optimizer, batch_size=4096, num_samples=64):
    """
    One training step.
    """
    # Sample random pixels from random images
    # Select random time index t_idx
    t_idx = np.random.randint(len(images))
    t = t_idx # Assuming integer time steps
    image = images[t_idx]
    pose = poses[t_idx]
    
    # Sample batch_size pixels
    h, w, _ = image.shape
    pixel_indices = np.random.randint(0, h * w, batch_size)
    rows = pixel_indices // w
    cols = pixel_indices % w
    
    # Get ray origins and directions from pose
    # This requires camera projection logic. 
    # For simplicity, assume we have ray_origins and ray_directions for these pixels
    ray_origins = ... 
    ray_directions = ...
    
    gt_colors = image[rows, cols] # (batch_size, 3)
    
    # Render
    rendered_colors = []
    for i in range(batch_size):
        c, s = render_ray(ray_origins[i], ray_directions[i], near, far, deformation_net, canonical_net, t, num_samples)
        rendered_colors.append(c)
    rendered_colors = np.array(rendered_colors)
    
    # Loss
    loss = np.mean((gt_colors - rendered_colors) ** 2)
    
    # Backprop and update
    optimizer.step(loss, deformation_net, canonical_net)
    
    return loss

# Typical Parameters
# Images: 400x400
# Iterations: 800,000
# Batch size: 4096 rays
# Samples per ray: 64
# Optimizer: Adam, lr=5e-4, beta1=0.9, beta2=0.999, decay to 5e-5
# Curriculum learning: Sort images by timestamp, incrementally add higher timestamps
```

**Edge Cases and Pitfalls:**
*   **Complex Motions:** Deformation fields are difficult to learn when complex scene motions exist, potentially leading to blurry results [S4].
*   **Topological Changes:** Deformation fields are continuous and cannot easily model topological changes (e.g., objects appearing/disappearing) because they require discontinuities [S5].
*   **Transparency and Specularities:** Mapping from observation space to a canonical space may not correctly reconstruct transparent objects or specular highlights, as reflections depend on the viewing angle and surface orientation which change with deformation [S5].
*   **Single View per Time:** The method assumes a single view per time instant. If multiple views are available, the method can still be applied but the single-view constraint is a key limitation tested in the original work [S1].

## Subtopics

### [Dynamic NeRF Guided by Projected Colors](8882-dynamic-nerf-guided-by-projected-colors.md)
Replaces explicit temporal coordinate $t$ with multi-view projected color statistics to improve point correspondence estimation during on-the-fly training. It maps inputs to a "space-color" coordinate system where correspondence is easier to track due to color invariance under motion. Choose this when dealing with on-the-fly training scenarios where temporal redundancy is hard to exploit via simple deformation.

## Operating characteristics
*   **Time and Memory Complexity:** The method uses two 8-layer MLPs. The complexity is linear with respect to the number of samples per ray and the number of rays. Memory usage is dominated by the network weights and the storage of the training images.
*   **Measured Performance:** On the 'Hell Warrior' scene, D-NeRF achieves MSE $31 \times 10^{-4}$, PSNR 23.19, SSIM 0.93, LPIPS 0.08, compared to NeRF's MSE $44 \times 10^{-3}$, PSNR 13.52, SSIM 0.81, LPIPS 0.25 [S1]. On the 'T-Rex' scene, D-NeRF achieves MSE $5 \times 10^{-4}$, PSNR 32.80, SSIM 0.98, LPIPS 0.03, compared to NeRF's MSE $21 \times 10^{-3}$, PSNR 18.19, SSIM 0.89, LPIPS 0.14 [S1].
*   **Training Time:** Trained with a single Nvidia GTX 1080 for 2 days (800k iterations) [S1].
*   **Strengths:** Can synthesize novel views at arbitrary times from a sparse set of monocular views. Handles rigid, articulated, and non-rigid motions. Produces complete 3D meshes as a side product [S1].
*   **Limitations:** Struggles with complex motions (blurry results). Cannot handle topological changes. May fail to reconstruct transparency and specularities correctly [S4, S5].
*   **When to Use:** When you have a sparse set of monocular videos of a dynamic scene with non-rigid motion and want to synthesize novel views at arbitrary times.
*   **When Not to Use:** When the scene has significant topological changes, or when high-fidelity reconstruction of specularities/transparency is required. Also not suitable if multi-view synchronized data is available and you want to leverage multi-view consistency directly without deformation [S1].

## Background
D-NeRF was proposed by Albert Pumarola, Enric Corona, Gerard Pons-Moll, and Francesc Moreno-Noguer in 2020 [S1]. It was published in CVPR 2021 [S1]. The method extends NeRF [26] to handle dynamic scenes by introducing time as an additional input and splitting the representation into a deformation network and a canonical radiance network [S2, S3]. It builds on the idea of canonical space mapping, similar to Neural Volumes [35] and Scene Representation Networks [SRN], but applies it to non-rigid dynamic scenes from monocular input [S3, S4]. The method was evaluated on scenes with rigid, articulated, and non-rigid motions, such as 'Hell Warrior', 'T-Rex', and 'Bouncing Balls' [S1].

## Sources
[S1] D-NeRF: Neural Radiance Fields for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.01018, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Neural Radiance Fields-Comprehensive Survey, https://zenodo.org/records/7597137, fetched 2026-10-26
[S4] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S5] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
