# DVGO (Deep Voxel Grids)
Also called: Deep Voxel Grids, Direct Voxel Grid Optimization

## Overview
DVGO is a super fast radiance field method using a hybrid representation: implicit for color inference and explicit for density inference, chosen for its potential for real-time applications [S1, unofficial][S2]. It speeds up NeRF by replacing the density MLP with voxel grids, using implicit representation for color inference and explicit for density inference [S1, unofficial]. Unlike completely implicit NeRF representations, which are not fast enough for real-time applications due to the querying cost of volume rendering on a significant number of points, DVGO uses voxel grids to store density and color features, allowing for efficient trilinear interpolation and volume rendering [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input images with known camera poses; near and far planes defining the sampling interval for rays.
*   **Outputs:** A rendered color image $\hat{C}(r)$ for a given ray $r$.

**Data Structures:**
DVGO uses a dual-stage design: a coarse stage and a fine stage.
1.  **Coarse Stage:** Uses two explicit volumes (voxel grids):
    *   $V^{(\sigma,c)}$: Stores raw density $\ddot{\sigma}^{(c)}$. Layout: $\mathbb{R}^{N_x \times N_y \times N_z}$.
    *   $V^{(rgb,c)}$: Stores color $c^{(c)}$. Layout: $\mathbb{R}^{C \times N_x \times N_y \times N_z}$ (where $C$ is the color channel dimension, typically 3).
2.  **Fine Stage:** Uses refined grids and a shallow MLP:
    *   $V^{(\sigma,f)}$: Stores raw density $\ddot{\sigma}^{(f)}$. Layout: $\mathbb{R}^{N_x \times N_y \times N_z}$ (higher resolution than coarse).
    *   $V^{(feat,f)}$: Stores color features. Layout: $\mathbb{R}^{C \times N_x \times N_y \times N_z}$.
    *   $MLP_{\Theta_{rgb}}$: A shallow multilayer neural network parameterized by $\Theta$ for view-dependent color inference.

**Numbered Steps:**

1.  **Ray Sampling:** For a pixel-rendering ray $r$, sample $n$ points $x_i$ between pre-defined near and far planes [S2]. Let $\delta_i$ be the distance between adjacent sampled points [S2].

2.  **Coarse Stage Query (Optional/Initial):**
    *   Interpolate raw density: $\ddot{\sigma}^{(c)} = \text{interp}(x, V^{(\sigma,c)})$ [S1, unofficial].
    *   Interpolate color: $c^{(c)} = \text{interp}(x, V^{(rgb,c)})$ [S1, unofficial].
    *   *Note:* The coarse stage is primarily for fast convergence and geometry initialization [S1, unofficial].

3.  **Fine Stage Density:**
    *   Interpolate raw density from the refined grid: $\ddot{\sigma}^{(f)} = \text{interp}(x, V^{(\sigma,f)})$ [S1, unofficial].
    *   The density volume $V^{(\sigma,c)}$ is initialized with zero raw density to prevent suboptimal geometry [S1, unofficial]. The fine stage density grid $V^{(\sigma,f)}$ is initialized with the optimized coarse geometry $V^{(\sigma,c)}$ [S1, unofficial].

4.  **Fine Stage Color:**
    *   Interpolate features from the refined feature grid: $f_i = \text{interp}(x_i, V^{(feat,f)})$ [S1, unofficial][S2].
    *   Map the color feature to a view-dependent color emission $c_i^{(rgb)}$ using the shallow MLP:
        $$c_i^{(rgb)} = MLP_{\Theta_{rgb}}(f_i, x_i, d)$$
        where $d$ is the ray direction and positional encoding is applied to $x_i$ and $d$ [S2].

5.  **Density Activation:**
    *   Apply shifted softplus to the raw density to obtain final density $\sigma_i$:
        $$\sigma_i = \text{softplus}(\ddot{\sigma}_i) = \log(1 + \exp(\ddot{\sigma}_i + b))$$
        where $\ddot{\sigma}_i$ is the value stored in the density grid ($V^{(\sigma,f)}$ for fine stage) and $b$ is a shift hyperparameter [S1, unofficial][S2].

6.  **Volume Rendering Accumulation:**
    *   Calculate the light termination probability (opacity) $\alpha_i$:
        $$\alpha_i = 1 - \exp(-\sigma_i \delta_i)$$
    *   Calculate the accumulated transmittance $T_i$ from the near plane to point $i$:
        $$T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$$
    *   Accumulate results into a single color $\hat{C}(r)$ using volume rendering quadrature:
        $$\hat{C}(r) = \sum_{i=1}^{n} T_i \alpha_i c_i + T_{n+1} c_{bg}$$
        where $c_{bg}$ is a pre-defined background color [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def softplus(x, b):
    """Shifted softplus activation."""
    return np.log(1 + np.exp(x + b))

def trilinear_interpolate(grid, points):
    """
    Perform trilinear interpolation.
    grid: numpy array of shape (Nx, Ny, Nz) or (C, Nx, Ny, Nz)
    points: numpy array of shape (N, 3) containing 3D coordinates
    Returns: interpolated values. If grid is 3D, returns (N,). If 4D, returns (N, C).
    """
    # Implementation of trilinear interpolation is assumed standard.
    # This function maps (R^3, R^{C x Nx x Ny x Nz}) -> R^C.
    pass

def render_ray_dvgo(ray_origin, ray_direction, near_plane, far_plane, 
                    V_sigma_f, V_feat_f, mlp_rgb, b, bg_color, 
                    num_samples=128):
    """
    Render a single ray using the DVGO fine stage.
    
    Args:
        ray_origin: (3,) array
        ray_direction: (3,) array, normalized
        near_plane: float
        far_plane: float
        V_sigma_f: (Nx, Ny, Nz) array, raw density grid
        V_feat_f: (C, Nx, Ny, Nz) array, feature grid
        mlp_rgb: function taking (features, position, direction) -> color
        b: float, shift hyperparameter for softplus
        bg_color: (3,) array
        num_samples: int, number of sample points
        
    Returns:
        color: (3,) array
    """
    # 1. Sample points along the ray
    t = np.linspace(near_plane, far_plane, num_samples)
    points = ray_origin + t[:, np.newaxis] * ray_direction # Shape (num_samples, 3)
    
    # 2. Interpolate raw density
    raw_density = trilinear_interpolate(V_sigma_f, points) # Shape (num_samples,)
    
    # 3. Activate density
    density = softplus(raw_density, b) # Shape (num_samples,)
    
    # 4. Interpolate features
    features = trilinear_interpolate(V_feat_f, points) # Shape (num_samples, C)
    
    # 5. Compute view-dependent color
    # Positional encoding is applied to points and direction before MLP
    # Assuming mlp_rgb handles positional encoding internally or expects encoded inputs
    colors = np.array([mlp_rgb(features[i], points[i], ray_direction) for i in range(num_samples)]) # Shape (num_samples, 3)
    
    # 6. Volume Rendering
    # Calculate delta distances
    deltas = np.diff(t, append=far_plane) # Shape (num_samples,)
    
    # Calculate alpha
    alpha = 1.0 - np.exp(-density * deltas) # Shape (num_samples,)
    
    # Calculate transmittance T_i
    # T_i = product_{j=1}^{i-1} (1 - alpha_j)
    # T_1 = 1.0
    transmittance = np.cumprod(1.0 - alpha)
    transmittance = np.roll(transmittance, 1)
    transmittance[0] = 1.0
    
    # Accumulate color
    # C(r) = sum_{i=1}^{n} T_i * alpha_i * c_i + T_{n+1} * c_bg
    contribution = transmittance * alpha[:, np.newaxis] * colors
    final_color = np.sum(contribution, axis=0) + transmittance[-1] * bg_color
    
    return final_color
```

**Parameters:**
*   $b$: A shift hyperparameter in the softplus activation function for density [S1, unofficial][S2].
*   $c_{bg}$: Pre-defined background color [S2].

**Edge Cases and Pitfalls:**
*   **Initialization:** $V^{(\sigma,c)}$ is initialized with zero raw density to prevent suboptimal geometry [S1, unofficial].
*   **Precision:** The softplus activation ensures density is non-negative. Numerical stability in $\exp(-\sigma_i \delta_i)$ should be considered for very large or small values [S2].
*   **Boundaries:** Sampling is restricted to pre-defined near and far planes [S2]. Points outside the voxel grid bounds require clamping or padding during interpolation.

## Subtopics

### Hybrid Representation
Uses both explicit voxel grids and implicit MLPs. Explicit for density, implicit for color. Faster than fully implicit NeRF. [Hybrid Representation](7846-hybrid-representation.md)

### NeRF
Neural Radiance Fields, the foundational implicit representation method. [NeRF](52-name.md)

## Operating characteristics
*   **Time and Memory Complexity:** DVGO is significantly faster than completely implicit NeRF representations because it replaces the density MLP with voxel grids, avoiding the querying cost of volume rendering on a significant number of points via MLPs [S1, unofficial]. Memory usage is determined by the resolution of the voxel grids ($N_x \times N_y \times N_z$) and the size of the shallow MLP [S2].
*   **Measured Performance:** DVGO demonstrates "super fast" radiance fields with the potential for real-time applications [S1, unofficial]. It offers competitive results and speed when compared to other hybrid representations [S1, unofficial].
*   **Strengths:** Efficient volumetric information gain computation; dual-stage design allows fast convergence in the coarse stage and high-quality reconstruction in the fine stage; flexibility in level of detail suits different task requirements [S1, unofficial].
*   **Limitations:** Resolution is limited by the voxel grid size; may suffer from aliasing or blurring at high resolutions if not properly regularized (not explicitly stated in sources, but implied by grid-based nature).
*   **When to Use:** When real-time or near real-time novel view synthesis is required, and a hybrid explicit-implicit representation is acceptable [S1, unofficial][S2].
*   **When Not to Use:** When extremely high fidelity without resolution constraints is needed and training/inference time is not a concern (use fully implicit NeRF) [S1, unofficial].

## Background
*   **Origin:** DVGO (Direct Voxel Grid Optimization) was proposed as a method to speed up NeRF [S1, unofficial][S2]. It is cited as reference [59] in [S1, unofficial] and [35] in [S2].
*   **History:** It is part of a recent trend of "super fast radiance fields" using hybrid representations (implicit for light field, explicit for density field) [S1, unofficial].
*   **Usage:** Used in ActiveRMAP as the hybrid representation for the radiance field [S1, unofficial].

## Sources
[S1] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S2] NeRF Synthesis with Shading Guidance, https://link.springer.com/chapter/10.1007/978-981-99-9666-7_16, fetched 2026-10-26
