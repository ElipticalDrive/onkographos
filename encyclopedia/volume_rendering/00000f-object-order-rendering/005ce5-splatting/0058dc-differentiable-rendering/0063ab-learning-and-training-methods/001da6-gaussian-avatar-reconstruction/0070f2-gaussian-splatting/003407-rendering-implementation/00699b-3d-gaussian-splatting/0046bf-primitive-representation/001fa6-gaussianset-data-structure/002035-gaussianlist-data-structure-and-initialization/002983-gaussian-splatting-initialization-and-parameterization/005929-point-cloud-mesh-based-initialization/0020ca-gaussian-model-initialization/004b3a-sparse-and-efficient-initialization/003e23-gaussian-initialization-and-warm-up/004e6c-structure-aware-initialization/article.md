# Structure-Aware Initialization

Also called: Variance-based Initialization

## Overview
Structure-Aware Initialization is a strategy that adapts primitive placement and scale to local image complexity using local variance [S1, unofficial]. It solves the problem of inefficient coverage in radiance-field optimization by ensuring dense, fine-grained primitives in complex regions and sparse, coarse coverage in smooth areas [S1, unofficial]. This method accelerates convergence and improves final fidelity, particularly when using high primitive counts [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Target image $I$ (RGB), canvas dimensions, parameters $s_{max}, s_{min}, \sigma_c$ [S1, unofficial].
- Output: Set of initial Gaussian parameters $\Theta = \{(x_i, y_i, s_i, c_i, \nu_i, \theta_i)\}$ [S1, unofficial].

**Data Structures:**
- Lattice points: A regular grid of candidate positions $(x, y)$ covering the image canvas [S1, unofficial].
- Normalized Local Variance map: $NLV: \text{canvas} \rightarrow [0, 1]$ [S1, unofficial].

**Steps:**
1. **Compute Local Variance:** Calculate the local variance of the target image $I$ across RGB channels using a $7 \times 7$ sliding window [S1, unofficial]. Let $V(x, y)$ be the variance at pixel $(x, y)$ [S1, unofficial].
2. **Normalize Variance:** Normalize the variance map to $[0, 1]$ to obtain $NLV(x, y)$ [S1, unofficial].
   $$ NLV(x, y) = \frac{V(x, y) - \min(V)}{\max(V) - \min(V)} $$
   [G: Handle case where $\max(V) = \min(V)$ by setting $NLV=0$ or $0.5$ depending on implementation preference, though sources do not specify.]
3. **Sample Positions:** Sample initial positions $(x_i, y_i)$ from the lattice points with probability proportional to:
   $$ P(x, y) \propto 0.1 + 0.9 \cdot NLV(x, y) $$
   [S1, unofficial]
4. **Initialize Scale:** Set the scale $s_i$ for each sampled point based on its normalized local variance:
   $$ s_i = s_{max} - (s_{max} - s_{min}) \cdot NLV(x_i, y_i) $$
   [S1, unofficial]
5. **Initialize Color:** Sample color $c_i$ from a normal distribution centered on the target image color at that position:
   $$ c_i \sim \mathcal{N}(I(x_i, y_i), \sigma_c^2) $$
   [S1, unofficial]
6. **Initialize Opacity:** Fix the opacity parameter $\nu_i$ to a constant value to ensure gradient flow:
   $$ \nu_i = -2.0 \quad (\approx 12\% \text{ opacity}) $$
   [S1, unofficial]
7. **Initialize Rotation:** Sample rotation angle $\theta_i$ uniformly:
   $$ \theta_i \sim \mathcal{U}[0, 2\pi) $$
   [S1, unofficial]

**Parameters:**
- Sliding window size: $7 \times 7$ [S1, unofficial].
- Opacity prior $\nu_i$: $-2.0$ [S1, unofficial].
- Color noise $\sigma_c$: Not given in the sources [S1, unofficial].
- Scale bounds $s_{max}, s_{min}$: Not given in the sources [S1, unofficial].

**Pitfalls:**
- Disabling both soft rasterization and structure-aware initialization often leads to destabilization of the fit and missing fine details [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def structure_aware_initialization(target_image, lattice_points, s_max, s_min, sigma_c):
    """
    Initialize Gaussian splats based on local image variance.
    
    Args:
        target_image: np.array of shape (H, W, 3), RGB image.
        lattice_points: list of (x, y) tuples covering the canvas.
        s_max: float, maximum initial scale.
        s_min: float, minimum initial scale.
        sigma_c: float, standard deviation for color initialization.
        
    Returns:
        positions: np.array of shape (N, 2)
        scales: np.array of shape (N,)
        colors: np.array of shape (N, 3)
        opacities: np.array of shape (N,)
        rotations: np.array of shape (N,)
    """
    h, w, _ = target_image.shape
    window_size = 7
    half_window = window_size // 2
    
    # Pad image for boundary handling
    padded_image = np.pad(target_image, ((half_window, half_window), (half_window, half_window), (0, 0)), mode='reflect')
    
    # Compute variance for each channel in the sliding window
    # Variance = E[X^2] - (E[X])^2
    # Use a uniform kernel for the sliding window mean
    kernel = np.ones((window_size, window_size)) / (window_size ** 2)
    
    def sliding_mean(img_2d):
        return np.convolve(img_2d.flatten(), kernel.flatten(), mode='valid').reshape(h, w)
    
    var_r = sliding_mean(padded_image[:, :, 0][half_window:-half_window, half_window:-half_window]**2) - sliding_mean(padded_image[:, :, 0][half_window:-half_window, half_window:-half_window])**2
    var_g = sliding_mean(padded_image[:, :, 1][half_window:-half_window, half_window:-half_window]**2) - sliding_mean(padded_image[:, :, 1][half_window:-half_window, half_window:-half_window])**2
    var_b = sliding_mean(padded_image[:, :, 2][half_window:-half_window, half_window:-half_window]**2) - sliding_mean(padded_image[:, :, 2][half_window:-half_window, half_window:-half_window])**2
    
    local_variance = var_r + var_g + var_b
    
    # Normalize variance to [0, 1]
    min_var = np.min(local_variance)
    max_var = np.max(local_variance)
    
    if max_var == min_var:
        nlv_map = np.zeros_like(local_variance)
    else:
        nlv_map = (local_variance - min_var) / (max_var - min_var)
        
    # Map lattice points to grid indices
    # Assuming lattice_points are integer coordinates within [0, H-1] x [0, W-1]
    lattice_indices = [(y * w + x) for x, y in lattice_points]
    probs = 0.1 + 0.9 * nlv_map.flatten()
    probs = probs / np.sum(probs) # Normalize to sum to 1
    
    # Sample N points from lattice_points based on probs
    num_samples = len(lattice_points)
    sampled_flat_indices = np.random.choice(len(probs), size=num_samples, replace=False, p=probs)
    
    # Convert flat indices back to (x, y)
    sampled_y = sampled_flat_indices // w
    sampled_x = sampled_flat_indices % w
    positions = np.column_stack((sampled_x, sampled_y))
    
    # Initialize scale
    nlv_values = nlv_map[sampled_y, sampled_x]
    scales = s_max - (s_max - s_min) * nlv_values
    
    # Initialize color
    target_colors = target_image[sampled_y, sampled_x]
    colors = np.random.normal(loc=target_colors, scale=sigma_c, size=(num_samples, 3))
    
    # Initialize opacity
    opacities = np.full(num_samples, -2.0)
    
    # Initialize rotation
    rotations = np.random.uniform(0, 2 * np.pi, num_samples)
    
    return positions, scales, colors, opacities, rotations
```

## Subtopics

### Farthest Point Sampling with Prefix Nearest-Neighbor Scale Initialization
This initialization creates an implicit hierarchy where points sampled early are larger and points sampled later are smaller [S2]. It uses farthest point sampling to sample positions [S2]. For the scale of point $i$ with position $x_{Ii}$, it runs nearest-neighbors across prefixes of the farthest point sampling order so that the initial scale is approximately $\min_{0 \le j < i} d(x_{Ii}, x_{Ij})$ [S2]. Unlike standard 3DGS initialization, this method allows larger primitives to potentially replace smaller ones via texture alone during optimization [S2]. A pitfall is that using nearest-neighbors initialization from Kerbl et al. [22] leads to small primitives, which is not ideal for this representation [S2].

## Operating characteristics
**Time and Memory Complexity:**
- The runtime overhead is almost the same as standard initialization [S1, unofficial].
- Not given in the sources.

**Measured Performance:**
- On 512×512 targets, PSNR improves from 24.4 dB (disabled) to 25.5 dB (enabled) [S1, unofficial].
- On 1024×1024 targets, PSNR improves from 25.7 dB to 25.9 dB [S1, unofficial].
- Using both structure-aware initialization and soft rasterization produces the best results across all targets, with sharper edges, fewer spurious fragments, and reduced seed-to-seed variance [S1, unofficial].

**Strengths and Limitations:**
- Accelerates convergence and improves final fidelity, particularly when using high primitive counts [S1, unofficial].
- Ensures dense and fine coverage in complex regions while maintaining sparse and coarse coverage in flat areas [S1, unofficial].
- Disabling both soft rasterization and structure-aware initialization often leads to destabilization of the fit and missing fine details [S1, unofficial].

**When to Use:**
- When using high primitive counts and requiring faster convergence and higher fidelity [S1, unofficial].
- When fine detail preservation is critical [S1, unofficial].

## Background
Originated in DiffBMP [S1, unofficial]. It is used in differentiable rendering with bitmap primitives to improve the quality of dynamic and spatially constrained rendering [S1, unofficial].

## Sources
[S1] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
[S2] Nexels: Neurally-Textured Surfels for Real-Time Novel View Synthesis with Sparse Primitives, https://doi.org/10.1007/978-3-032-37531-5_9, fetched 2026-10-28
