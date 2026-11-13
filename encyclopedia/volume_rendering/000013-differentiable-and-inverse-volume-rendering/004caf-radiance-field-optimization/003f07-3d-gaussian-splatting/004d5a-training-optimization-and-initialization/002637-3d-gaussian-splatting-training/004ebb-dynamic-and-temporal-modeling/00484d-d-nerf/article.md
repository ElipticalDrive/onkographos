# D-NeRF

Also called: Deformable Neural Radiance Fields, Dynamic NeRF, Neural Radiance Fields for Dynamic Scenes

## Overview
D-NeRF synthesizes novel views at arbitrary time instants for dynamic scenes with complex non-rigid geometries, optimizing a deformable volumetric function from a sparse set of monocular input images without requiring ground-truth geometry or multi-view stereo data [S1]. It distinguishes itself from naive time-extended NeRFs by decomposing the representation into a canonical static radiance field and a learned deformation field, effectively exploiting temporal redundancy to handle rigid, articulated, and non-rigid motions [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse set of $T$ RGB images $\{I_t\}_{t=1}^T$ and their corresponding camera pose matrices $\{T_t\}_{t=1}^T$, where $I_t \in \mathbb{R}^{H \times W \times 3}$ and $T_t \in \mathbb{R}^{4 \times 4}$ represents the SE(3) camera pose at time $t$. The method assumes a single view per time instant [S1, S3].
*   **Outputs:** A continuous 6D function mapping spatial coordinates $\mathbf{x}$, viewing direction $\mathbf{d}$, and time $t$ to volume density $\sigma$ and view-dependent RGB color $\mathbf{c}$, enabling the rendering of novel views at any $t$ [S1].

**Data Structures:**
The model consists of two deep fully connected networks (MLPs) without convolutional layers [S1]:
1.  **Deformation Network ($\Psi_t$):** Maps a point $\mathbf{x}$ at time $t$ to a displacement $\Delta \mathbf{x}$ in the canonical space.
    *   **Architecture:** 8-layer MLP with ReLU activations.
    *   **Output:** No non-linearity applied to $\Delta \mathbf{x}$ [S1, S3].
    *   **Constraint:** $\Psi_t(\mathbf{x}, 0) = \mathbf{0}$ [S1].
2.  **Canonical Network ($\Psi_x$):** Regresses density $\sigma$ and color $\mathbf{c}$ from the canonical coordinate $\mathbf{x}' = \mathbf{x} + \Delta \mathbf{x}$ and direction $\mathbf{d}$.
    *   **Architecture:** 8-layer MLP with ReLU activations.
    *   **Output:** Sigmoid non-linearity applied to $\mathbf{c}$ and $\sigma$ [S1, S3].

**Positional Encoding:**
Inputs are encoded using $\gamma(\mathbf{p}) = \langle (\sin(2^l \pi \mathbf{p}), \cos(2^l \pi \mathbf{p})) \rangle_{l=0}^{L-1}$ [S3].
*   Spatial coordinates $\mathbf{x}$: $L=10$ [S1, S3].
*   Viewing direction $\mathbf{d}$ and time $t$: $L=4$ [S1, S3].

**Numbered Steps:**
1.  **Canonical Definition:** Define the canonical scene configuration at time $t=0$. Enforce that the deformation displacement is zero at this time: $\Psi_t(\mathbf{x}, 0) = \mathbf{0}$ [S1, S3].
2.  **Ray Sampling:** For a pixel $p$ and time $t$, cast a ray $\mathbf{x}(h) = \mathbf{o} + h\mathbf{d}$ from camera center $\mathbf{o}$ with direction $\mathbf{d}$, bounded by near $h_n$ and far $h_f$ [S3].
3.  **Deformation:** Sample $N$ points $\{h_n\}_{n=1}^N$ along the ray using stratified sampling [S1]. For each sample $h_n$, compute the 3D position $\mathbf{x}(h_n)$ and transform it to canonical space using the deformation network:
    $$ \mathbf{p}(h_n, t) = \mathbf{x}(h_n) + \Psi_t(\mathbf{x}(h_n), t) $$
    [S1, S3].
4.  **Radiance Query:** Query the canonical network with the transformed point $\mathbf{p}(h_n, t)$ and viewing direction $\mathbf{d}$ to obtain density $\sigma$ and color $\mathbf{c}$:
    $$ [\mathbf{c}(\mathbf{p}(h_n, t), \mathbf{d}), \sigma(\mathbf{p}(h_n, t))] = \Psi_x(\mathbf{p}(h_n, t), \mathbf{d}) $$
    [S1, S3].
5.  **Volume Rendering:** Compute the accumulated transmittance $T'$ and opacity $\alpha$ for each sample. Let $\delta_n = h_{n+1} - h_n$ be the distance between quadrature points [S1].
    $$ \alpha(h_n, t, \delta_n) = 1 - \exp(-\sigma(\mathbf{p}(h_n, t)) \delta_n) $$
    $$ T'(h_n, t) = \exp\left(-\sum_{m=1}^{n-1} \sigma(\mathbf{p}(h_m, t)) \delta_m\right) $$
    [S1].
6.  **Pixel Color:** Approximate the pixel color $C'(p, t)$ as the weighted sum of emitted colors:
    $$ C'(p, t) = \sum_{n=1}^{N} T'(h_n, t) \alpha(h_n, t, \delta_n) \mathbf{c}(\mathbf{p}(h_n, t), \mathbf{d}) $$
    [S1].
7.  **Optimization:** Minimize the mean squared error (MSE) between the rendered pixel $C'(p, t)$ and the ground truth pixel $\hat{C}(p, t)$:
    $$ \text{Loss} = \frac{1}{N_s} \sum_{i=1}^{N_s} \|\hat{C}(p_i, t) - C'(p_i, t)\|^2 $$
    where $N_s$ is the batch size in rays [S1, S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def positional_encoding(p, L):
    """
    Applies positional encoding to input p.
    p: tensor of shape (..., D)
    L: number of frequency bands
    Returns: tensor of shape (..., D * 2 * L)
    """
    # p is assumed to be a tensor of coordinates
    # We apply the encoding independently to each coordinate component
    encoded = []
    for l in range(L):
        freq = 2**l * math.pi
        encoded.append(math.sin(freq * p))
        encoded.append(math.cos(freq * p))
    return torch.cat(encoded, dim=-1)

def deformation_network(x, t):
    """
    8-layer MLP with ReLU activations.
    Input: encoded x (L=10) and encoded t (L=4).
    Output: displacement delta_x (no non-linearity).
    """
    # Implementation details of the MLP are omitted as per standard practice,
    # but the structure is: Linear -> ReLU -> ... -> Linear (output)
    # Input dimensions: 3*2*10 + 1*2*4 = 60 + 8 = 68
    # Output dimensions: 3
    pass

def canonical_network(x_prime, d):
    """
    8-layer MLP with ReLU activations.
    Input: encoded x_prime (L=10) and encoded d (L=4).
    Output: color c and density sigma (sigmoid applied).
    """
    # Input dimensions: 3*2*10 + 3*2*4 = 60 + 24 = 84
    # Output dimensions: 4 (3 for color, 1 for sigma)
    pass

def render_ray(o, d, t, h_n, h_f, N_samples):
    """
    Renders a single ray at time t.
    o: camera origin (3,)
    d: viewing direction (3,)
    t: time scalar
    h_n, h_f: near and far bounds
    N_samples: number of samples
    Returns: color (3,)
    """
    # 1. Stratified sampling
    # Divide [h_n, h_f] into N_samples bins
    # Draw one sample uniformly from each bin
    bins = torch.linspace(h_n, h_f, N_samples + 1)
    h_samples = bins[:-1] + torch.rand(N_samples) * (bins[1] - bins[0])
    
    # 2. Get 3D points on the ray
    # x(h) = o + h * d
    x_samples = o + h_samples.unsqueeze(1) * d.unsqueeze(0) # Shape: (N_samples, 3)
    
    # 3. Deformation
    # Encode x and t
    enc_x = positional_encoding(x_samples, L=10)
    enc_t = positional_encoding(torch.full((N_samples, 1), t), L=4)
    enc_input = torch.cat([enc_x, enc_t], dim=-1)
    
    # Get displacement delta_x
    delta_x = deformation_network(enc_input) # Shape: (N_samples, 3)
    
    # Transform to canonical space
    x_canonical = x_samples + delta_x
    
    # 4. Canonical Query
    # Encode x_canonical and d
    enc_x_c = positional_encoding(x_canonical, L=10)
    enc_d = positional_encoding(d.unsqueeze(0).expand(N_samples, -1), L=4)
    enc_input_c = torch.cat([enc_x_c, enc_d], dim=-1)
    
    # Get sigma and c
    output = canonical_network(enc_input_c) # Shape: (N_samples, 4)
    sigma = output[:, 3] # Shape: (N_samples,)
    c = output[:, :3]    # Shape: (N_samples, 3)
    
    # Apply sigmoid to sigma and c if not done inside network
    # Note: Source says sigmoid is applied to output of canonical network
    sigma = torch.sigmoid(sigma)
    c = torch.sigmoid(c)
    
    # 5. Volume Rendering
    # Compute delta_n
    delta_n = h_samples[1:] - h_samples[:-1]
    delta_n = torch.cat([delta_n, torch.tensor([h_f - h_samples[-1]])]) # Last bin
    
    # Compute alpha and T'
    # alpha = 1 - exp(-sigma * delta)
    alpha = 1.0 - torch.exp(-sigma * delta_n)
    
    # T' = exp(-sum(sigma * delta))
    # Cumulative product of (1 - alpha)
    transmittance = torch.cumprod(1.0 - alpha, dim=0)
    # Shift transmittance: T'(h_n) corresponds to sum before n
    # T'(h_1) = 1.0
    # T'(h_n) = prod_{m=1}^{n-1} (1 - alpha_m)
    T_prime = torch.cat([torch.tensor([1.0]), transmittance[:-1]])
    
    # Color = sum(T' * alpha * c)
    color = torch.sum(T_prime.unsqueeze(1) * alpha.unsqueeze(1) * c, dim=0)
    
    return color

def train_step(images, poses, t_values, optimizer):
    """
    One training step.
    images: list of T images
    poses: list of T poses
    t_values: list of T time values
    """
    optimizer.zero_grad()
    
    # Select a random time t and image I_t
    idx = random.randint(0, len(images) - 1)
    t = t_values[idx]
    I_t = images[idx]
    T_t = poses[idx]
    
    # Sample Ns rays from I_t
    # Ns = 4096
    # For each ray, call render_ray
    rendered_colors = []
    for _ in range(4096):
        # Sample pixel coordinates
        u, v = random.randint(0, H-1), random.randint(0, W-1)
        # Cast ray from camera pose T_t to pixel (u, v)
        o, d = get_ray_origin_direction(T_t, u, v, H, W, focal_length)
        color = render_ray(o, d, t, h_n, h_f, 64)
        rendered_colors.append(color)
        
    rendered_colors = torch.stack(rendered_colors)
    # Get ground truth colors for sampled pixels
    gt_colors = get_pixel_colors(I_t, sampled_u, sampled_v)
    
    loss = torch.mean((gt_colors - rendered_colors) ** 2)
    loss.backward()
    optimizer.step()
```

**Parameters:**
*   **Canonical Time:** $t=0$ [S1].
*   **Positional Encoding Depth:** $L=10$ for $\mathbf{x}$, $L=4$ for $\mathbf{d}$ and $t$ [S1].
*   **Network Depth:** 8 layers for both $\Psi_t$ and $\Psi_x$ [S1].
*   **Training Images:** $400 \times 400$ resolution [S1].
*   **Iterations:** 800,000 [S1].
*   **Batch Size:** $N_s = 4096$ rays [S1].
*   **Samples per Ray:** 64 [S1].
*   **Optimizer:** Adam, learning rate $5 \times 10^{-4}$, $\beta_1=0.9$, $\beta_2=0.999$, exponential decay to $5 \times 10^{-5}$ [S1].

**Edge Cases and Pitfalls:**
*   **Complex Motions:** Deformation fields are difficult to learn when complex scene motions exist, potentially leading to blurry results similar to Neural Volumes [S4].
*   **Single View per Time:** The method is designed for sparse monocular inputs (one view per time), which limits multi-view consistency checks but allows training from limited data [S1].
*   **Curriculum Learning:** To improve convergence, input images should be sorted by timestamp and added incrementally (curriculum learning) [S1].

## Subtopics

### [Dynamic NeRF Guided by Projected Colors](8882-dynamic-nerf-guided-by-projected-colors.md)
This method is a dynamic Neural Radiance Field (NeRF) variant that replaces the explicit temporal coordinate $t$ with multi-view projected color statistics to improve point correspondence estimation during on-the-fly training. Unlike standard space-time dynamic NeRFs that optimize irregular temporal deformation fields, this approach maps inputs to a "space-color" coordinate system where correspondence is easier to track due to color invariance under motion.

## Operating characteristics
*   **Time Complexity:** Training requires 800k iterations on a single Nvidia GTX 1080 for 2 days [S1]. Rendering time is not explicitly given in milliseconds for D-NeRF in the sources, but it involves two MLP queries per sample [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   **Hell Warrior:** MSE $31 \times 10^{-4}$, PSNR 23.19, SSIM 0.93, LPIPS 0.08 (vs NeRF: PSNR 13.52) [S1].
    *   **T-Rex:** MSE $5 \times 10^{-4}$, PSNR 32.80, SSIM 0.98, LPIPS 0.03 (vs NeRF: PSNR 18.19) [S1].
*   **Strengths:** Handles rigid, articulated, and non-rigid motions; works with sparse monocular views; produces high-quality novel views [S1].
*   **Limitations:** Struggles with complex scene motions leading to blurriness; deformation fields are hard to learn [S4].
*   **When to Use:** When synthesizing novel views of dynamic scenes with non-rigid deformations from sparse monocular inputs is required.
*   **When Not to Use:** When high-frequency details are critical in scenes with very complex motions, or when multi-view stereo data is available and rigid reconstruction is sufficient.

## Background
D-NeRF was proposed by Albert Pumarola, Enric Corona, Gerard Pons-Moll, and Francesc Moreno-Noguer in 2020/2021 [S1]. It extends the Neural Radiance Fields (NeRF) method [26] to handle dynamic scenes by introducing time as an additional input and decomposing the learning process into a canonical space and a deformation field [S2, S3]. It builds on the observation that objects move and deform but typically do not appear or disappear, inspired by classical 3D scene flow [S1].

## Sources
[S1] D-NeRF: Neural Radiance Fields for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.01018, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Neural Radiance Fields-Comprehensive Survey, https://zenodo.org/records/7597137, fetched 2026-10-26
[S4] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S5] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
