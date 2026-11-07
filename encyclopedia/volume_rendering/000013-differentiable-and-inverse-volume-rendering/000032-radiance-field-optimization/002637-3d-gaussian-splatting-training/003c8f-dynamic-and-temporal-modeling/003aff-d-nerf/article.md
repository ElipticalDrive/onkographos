# D-NeRF

Also called: Deformable Neural Radiance Fields, Dynamic-NeRF, Neural Radiance Fields for Dynamic Scenes

## Overview
D-NeRF is a method for synthesizing novel views at an arbitrary point in time of dynamic scenes with complex non-rigid geometries, optimized from a sparse set of input monocular views without ground-truth geometry or multi-view images [S1]. It decomposes the learning process into two modules: a spatial mapping that encodes the scene into a canonical space, and a mapping that transforms this canonical representation into the deformed scene at a particular time [S1]. Unlike naive extensions of NeRF that map $(x, y, z, t)$ directly to density and radiance, D-NeRF effectively exploits temporal redundancy by enforcing a common canonical anchor (typically $t=0$), ensuring that the deformation displacement is zero at the canonical time [S1]. This approach allows for the rendering of rigid, articulated, and non-rigid motions from a single camera view per time instant [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse set of $T$ RGB images $\{I_t\}_{t=1}^T$ and their corresponding camera pose matrices $\{T_t\}_{t=1}^T$, where $I_t \in \mathbb{R}^{H \times W \times 3}$ and $T_t \in \mathbb{R}^{4 \times 4}$ (SE(3)) [S1, S2].
*   **Outputs:** Rendered pixel colors $C'(p, t)$ for any pixel $p$ and time $t$, given camera parameters [S1].

**Data Structures:**
The model consists of two deep fully connected networks (MLPs) without convolutional layers [S1]:
1.  **Deformation Network $\Psi_t$:**
    *   **Architecture:** 8-layer MLP with ReLU activations.
    *   **Input:** Positional encoding of spatial coordinate $\mathbf{x}$ and time $t$.
    *   **Output:** Displacement vector $\Delta \mathbf{x} \in \mathbb{R}^3$. No non-linearity applied to the output [S1, S2].
2.  **Canonical Network $\Psi_x$:**
    *   **Architecture:** 8-layer MLP with ReLU activations.
    *   **Input:** Positional encoding of transformed spatial coordinate $\mathbf{x} + \Delta \mathbf{x}$ and viewing direction $\mathbf{d}$.
    *   **Output:** Volume density $\sigma \in \mathbb{R}$ and view-dependent RGB color $\mathbf{c} \in \mathbb{R}^3$. A final sigmoid non-linearity is applied to $\mathbf{c}$ and $\sigma$ [S1, S2].

**Positional Encoding:**
Coordinates are encoded using $\gamma(\mathbf{p}) = \langle (\sin(2^l \pi \mathbf{p}), \cos(2^l \pi \mathbf{p})) \rangle_{l=0}^{L-1}$, applied independently to each coordinate and camera view component [S1, S2].
*   For spatial coordinates $\mathbf{x}$: $L = 10$ [S1, S2].
*   For viewing direction $\mathbf{d}$ and time $t$: $L = 4$ [S1, S2].

**Numbered Steps:**
1.  **Canonical Configuration Definition:** Define the canonical scene as the scene at time $t=0$. Enforce that the deformation displacement is zero at this time: $\Psi_t(\mathbf{x}, 0) = \mathbf{0}$ [S1, S2].
2.  **Ray Sampling:** For a training batch, sample a random set of pixels $\{p_{t,i}\}$ corresponding to rays cast from camera position $T_t$ to pixels $i$ of image $I_t$ [S1, S2].
3.  **Discretization:** Along each ray, define near and far bounds $h_n$ and $h_f$. Draw $N$ samples $\{h_n\}_{n=1}^N$ from evenly-spaced ray bins in $[h_n, h_f]$ using stratified sampling [S1, S2].
4.  **Deformation (Canonical Mapping):** For each sample point $h$ on the ray, let $\mathbf{x}(h)$ be the 3D position. Transform it to the canonical space using the deformation network:
    $$ \Delta \mathbf{x} = \Psi_t(\mathbf{x}(h), t) $$
    $$ \mathbf{p}(h, t) = \mathbf{x}(h) + \Delta \mathbf{x} $$
    where $\mathbf{p}(h, t)$ is the point in canonical space [S1, S2].
5.  **Radiance Field Query:** Query the canonical network with the transformed point $\mathbf{p}(h, t)$ and viewing direction $\mathbf{d}$ to obtain density and color:
    $$ [\mathbf{c}(\mathbf{p}(h, t), \mathbf{d}), \sigma(\mathbf{p}(h, t))] = \Psi_x(\mathbf{p}(h, t), \mathbf{d}) $$
    [S1, S2].
6.  **Volume Rendering:** Compute the accumulated transmittance and opacity for each sample. Let $\delta_n = h_{n+1} - h_n$ be the distance between quadrature points [S1].
    *   Opacity: $\alpha(h, t, \delta) = 1 - \exp(-\sigma(\mathbf{p}(h, t)) \delta)$ [S1].
    *   Transmittance: $T'(h_n, t) = \exp\left(-\sum_{m=1}^{n-1} \sigma(\mathbf{p}(h_m, t)) \delta_m\right)$ [S1].
    *   Rendered Color:
        $$ C'(p, t) = \sum_{n=1}^{N} T'(h_n, t) \alpha(h_n, t, \delta_n) \mathbf{c}(\mathbf{p}(h_n, t), \mathbf{d}) $$
        [S1].
7.  **Optimization:** Minimize the mean squared error between the rendered pixels $C'(p, t)$ and ground-truth pixels $\hat{C}(p, t)$:
    $$ \text{Loss} = \frac{1}{N_s} \sum_{i=1}^{N_s} \|\hat{C}(p_i, t) - C'(p_i, t)\|^2 $$
    where $N_s$ is the batch size in rays [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def positional_encoding(p, L):
    """
    Applies positional encoding to input p.
    p: numpy array of shape (..., D)
    L: int, number of frequencies
    Returns: numpy array of shape (..., D * 2 * L)
    """
    # p is expected to be a vector or batch of vectors
    # Reshape to (N, D) for processing
    original_shape = p.shape
    p_flat = p.reshape(-1, p.shape[-1])
    
    freq_bands = 2.0 ** np.arange(L) * np.pi
    # sin(2^l * pi * p), cos(2^l * pi * p)
    sines = np.sin(np.outer(p_flat, freq_bands).reshape(p_flat.shape[0], -1, 2 * L // 2)) # Simplified logic below
    # Correct implementation:
    # For each dimension d in p:
    #   for l in 0..L-1:
    #     append sin(2^l * pi * p_d)
    #     append cos(2^l * pi * p_d)
    
    encoded = []
    for d in range(p_flat.shape[1]):
        col = p_flat[:, d]
        for l in range(L):
            scale = 2.0 ** l * np.pi
            encoded.append(np.sin(col * scale))
            encoded.append(np.cos(col * scale))
            
    encoded = np.stack(encoded, axis=1)
    return encoded.reshape(original_shape[:-1] + (encoded.shape[1],))

def deformation_network(x_encoded, t_encoded):
    """
    8-layer MLP with ReLU. Output: Delta x (3D).
    """
    # Placeholder for actual MLP weights
    # Input: x_encoded (L=10 for x) + t_encoded (L=4 for t)
    # Output: 3
    return np.zeros(3) # [G] Placeholder for actual network forward pass

def canonical_network(xp_encoded, d_encoded):
    """
    8-layer MLP with ReLU. Output: sigma (1), c (3). Sigmoid on output.
    """
    # Input: xp_encoded (L=10) + d_encoded (L=4)
    # Output: 4
    sigma_raw, c_raw = np.zeros(1), np.zeros(3) # [G] Placeholder
    sigma = 1.0 / (1.0 + np.exp(-sigma_raw))
    c = 1.0 / (1.0 + np.exp(-c_raw))
    return sigma, c

def render_ray(x_n, x_f, d, t, N_samples=64):
    """
    Renders a single ray.
    x_n, x_f: float, near and far bounds
    d: numpy array (3,), viewing direction
    t: float, time
    """
    # 1. Sample points
    # Stratified sampling in [x_n, x_f]
    bins = np.linspace(x_n, x_f, N_samples + 1)
    h = bins[:-1] + np.random.uniform(0, bins[1] - bins[0], N_samples)
    
    # 2. Get 3D positions on ray: x(h) = o + h*d
    # Note: o (origin) is not passed here, assumed handled by caller or implicit
    # For the formula p(h,t) = x(h) + Psi_t(x(h), t), we need x(h)
    # Let's assume ray_origin is available in context or passed.
    # For this self-contained function, we assume x_h is passed or computed.
    # To be strictly self-contained, we need ray_origin.
    # Let's assume the caller provides x_h (3D points).
    pass

def train_step(images, poses, N_s=4096, N_samples=64):
    """
    One training step.
    images: list of T images
    poses: list of T poses
    """
    # 1. Pick a random time t and image I_t, pose T_t
    t_idx = np.random.randint(len(images))
    I_t = images[t_idx]
    T_t = poses[t_idx]
    t = t_idx # Assuming discrete time steps 0..T-1
    
    # 2. Sample N_s rays
    H, W, _ = I_t.shape
    rays = []
    for _ in range(N_s):
        u = np.random.randint(0, W)
        v = np.random.randint(0, H)
        # Cast ray from T_t to pixel (u,v)
        # Get origin o and direction d
        o, d = cast_ray_from_pose(T_t, u, v, H, W) # [G] Helper for ray casting
        rays.append((o, d, u, v))
        
    rendered_colors = []
    gt_colors = []
    
    for o, d, u, v in rays:
        # 3. Sample along ray
        # Assume near/far bounds hn, hf are determined by scene geometry or fixed
        hn, hf = 0.0, 10.0 # [G] Typical bounds, not specified in source for general case
        
        # Stratified sampling
        bins = np.linspace(hn, hf, N_samples + 1)
        h = bins[:-1] + np.random.uniform(0, bins[1] - bins[0], N_samples)
        
        # 3D positions x(h)
        x_h = o + h[:, np.newaxis] * d # Shape (N_samples, 3)
        
        # 4. Deformation
        # Encode x and t
        x_enc = positional_encoding(x_h, L=10)
        t_enc = positional_encoding(np.full((N_samples, 1), t), L=4)
        
        # Get displacement
        delta_x = deformation_network(x_enc, t_enc) # Shape (N_samples, 3)
        
        # Canonical points
        p_h = x_h + delta_x
        
        # 5. Canonical Network
        d_enc = positional_encoding(np.tile(d, (N_samples, 1)), L=4)
        p_enc = positional_encoding(p_h, L=10)
        
        sigma, c = canonical_network(p_enc, d_enc)
        
        # 6. Rendering
        # Distances
        delta_h = np.diff(np.concatenate([np.array([hn]), bins])) # Shape (N_samples,)
        # Or delta_n = h_{n+1} - h_n
        delta_h = h[1:] - h[:-1]
        delta_h = np.append(delta_h, 0) # Last one doesn't matter for alpha calc usually, or handle boundary
        
        # Alpha and Transmittance
        # alpha = 1 - exp(-sigma * delta)
        alpha = 1.0 - np.exp(-sigma * delta_h[:, np.newaxis]) # Shape (N_samples, 1) if sigma is (N,1)
        # sigma is (N, 1), delta_h is (N,)
        alpha = 1.0 - np.exp(-sigma.flatten() * delta_h) # Shape (N_samples,)
        
        # T_n = exp(-sum_{m=1}^{n-1} sigma_m * delta_m)
        # Cumulative product of (1-alpha)
        T = np.ones(N_samples)
        T[1:] = np.cumprod(1.0 - alpha[:-1])
        
        # Color
        # C' = sum T_n * alpha_n * c_n
        # c is (N_samples, 3)
        color = np.sum(T[:, np.newaxis] * alpha[:, np.newaxis] * c, axis=0)
        
        rendered_colors.append(color)
        gt_colors.append(I_t[v, u])
        
    rendered_colors = np.array(rendered_colors)
    gt_colors = np.array(gt_colors)
    
    # Loss
    loss = np.mean((gt_colors - rendered_colors) ** 2)
    return loss
```

**Parameters:**
*   **Canonical Time:** $t=0$ [S1, S2].
*   **Positional Encoding Depth:** $L=10$ for $\mathbf{x}$, $L=4$ for $\mathbf{d}$ and $t$ [S1, S2].
*   **Network Depth:** 8 layers for both $\Psi_t$ and $\Psi_x$ [S1, S2].
*   **Training Images:** $400 \times 400$ resolution [S1].
*   **Iterations:** 800,000 [S1].
*   **Batch Size:** $N_s = 4096$ rays [S1].
*   **Samples per Ray:** 64 [S1].
*   **Optimizer:** Adam, learning rate $5 \times 10^{-4}$, $\beta_1=0.9$, $\beta_2=0.999$, exponential decay to $5 \times 10^{-5}$ [S1].

**Edge Cases and Pitfalls:**
*   **Complex Motions:** Deformation fields are difficult to learn when complex scene motions exist, potentially leading to blurry results similar to Neural Volumes [S3].
*   **Topology Changes:** Deformation fields must be continuous, making it difficult to simulate topology changes which require discontinuities [S4].
*   **Single View per Time:** The method assumes a single camera view per time instant, which limits multi-view consistency checks during training [S1].

## Subtopics

### [Dynamic NeRF Guided by Projected Colors](8882-dynamic-nerf-guided-by-projected-colors.md)
Replaces explicit temporal coordinate $t$ with multi-view projected color statistics to improve point correspondence estimation, mapping inputs to a "space-color" coordinate system.

## Operating characteristics
*   **Time Complexity:** Training requires 800k iterations on a single Nvidia GTX 1080, taking approximately 2 days [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:**
    *   **Hell Warrior:** MSE $31 \times 10^{-4}$, PSNR 23.19, SSIM 0.93, LPIPS 0.08 [S1].
    *   **T-Rex:** MSE $5 \times 10^{-4}$, PSNR 32.80, SSIM 0.98, LPIPS 0.03 [S1].
    *   Compared to NeRF (which fails on dynamic scenes, e.g., T-Rex PSNR 18.19) and T-NeRF (which lacks canonical mapping, e.g., T-Rex PSNR 32.01) [S1].
*   **Strengths:** Can synthesize novel views at arbitrary times for rigid, articulated, and non-rigid motions from sparse monocular views [S1]. Exploits temporal redundancy better than naive 6D extensions [S1].
*   **Limitations:** Struggles with complex motions resulting in blurriness; cannot handle topology changes due to continuous deformation field constraint; performance degrades on high-frequency details compared to methods using shared MLP maps or CNNs [S3, S4].

## Background
D-NeRF was proposed by Albert Pumarola, Enric Corona, Gerard Pons-Moll, and Francesc Moreno-Noguer from Institut de Robòtica i Informàtica Industrial (CSIC-UPC) and Max Planck Institute for Informatics [S1]. It was published in CVPR 2021 [S1]. The method builds on NeRF [26] and generalizes it to handle non-rigid scenes by decomposing the learning into a canonical scene and a scene flow deformation field, inspired by classical 3D scene flow [S1]. It addresses the limitation of NeRF which requires multiple views of a rigid scene, allowing D-NeRF to learn from a single view per time instant [S1].

## Sources
[S1] D-NeRF: Neural Radiance Fields for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.01018, fetched 2026-10-26
[S2] Neural Radiance Fields-Comprehensive Survey, https://zenodo.org/records/7597137, fetched 2026-10-26
[S3] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S4] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
