# Grid-NeRF

Also called: Grid-guided NeRF, Grid-guided Neural Radiance Fields (GridNeRF), Two-branch model

## Overview
Grid-NeRF is a two-branch architecture that integrates grid-based and Neural Radiance Field (NeRF) approaches under a joint learning scheme to render large-scale urban scenes [S1]. It solves the underfitting and blurring of purely MLP-based NeRFs and the noisy artifacts of pure feature grid methods on massive scenes [S1]. It sets itself apart by using a coarse factorized feature grid to guide the NeRF branch's sampling and inject geometric/appearance features, rather than relying solely on hierarchical hash encoding or surface-relative constraints [S1]. It builds on [Standard Multi-resolution Hash Encoding](13152-standard-multi-resolution-hash-encoding.md).

## How it works
Grid-NeRF consists of two parallel branches: a **Grid Branch** and a **NeRF Branch**, which are optimized jointly [S1]. The method relies on a factorized 3D feature grid to reduce memory complexity from $O(N^3)$ to $O(N^2)$ [S1].

### Inputs and Outputs
*   **Inputs:** 3D spatial coordinates $\mathbf{x} = (x, y, z)$ and viewing direction $\mathbf{d}$ [S1].
*   **Outputs:** Volume density $\sigma$ and color $\mathbf{c}$ for each branch, combined via volume rendering to produce the final rendered pixel color [S1].

### Data Structures
**Factorized Feature Grid:**
Unlike standard 3D voxel grids, Grid-NeRF factorizes the 3D feature grid into:
1.  **2D Ground Feature Planes:** A set of feature planes $\mathbf{P}$ spanning the $xy$-plane (ground level) [S1].
2.  **Vertical Shared Vector:** A feature vector $\mathbf{v}_z$ shared along the $z$-axis [S1].
This factorization exploits the observation that urban semantics are primarily distributed on the ground [S1]. The retrieved feature is computed as:
$$ \mathbf{f}_{\text{grid}}(\mathbf{x}) = \text{Bilinear}_{xy}(\mathbf{x}_{xy}, \mathbf{P}) + \mathbf{v}_z(x_z) $$
where $\text{Bilinear}_{xy}$ performs standard bilinear interpolation on the ground planes [G].

**Neural Networks:**
1.  **Grid Encoder:** Retrieves features from the factorized grid based on $\mathbf{x}$ [S1].
2.  **Lightweight MLP:** A small Multi-Layer Perceptron that maps the concatenated features and positional encodings to density $\sigma$ and color $\mathbf{c}$ [S1][S2, unofficial].

### Steps
**1. Pre-training Stage (Grid Branch Initialization):**
The target scene is modeled using the factorized feature grid to coarsely capture scene geometry and appearance [S1]. This provides a compact representation of the large scene [S1].

**2. Guidance Step 1 (Sampling Guidance):**
The coarse feature grid guides the NeRF branch's point sampling to concentrate around the scene surface, as inferred from the grid's geometry cues [S1]. This drastically compresses the sampling space [S1].

**3. Guidance Step 2 (Feature Injection):**
At each sampled position $\mathbf{x}_i$ along the ray:
*   Retrieve features from the factorized grid: $\mathbf{f}_{\text{grid}} = \text{GridRetrieve}(\mathbf{x}_i)$ [S1].
*   Compute standard NeRF positional encoding: $\mathbf{PE}(\mathbf{x}_i)$ [S1].
*   Concatenate: $\mathbf{F}_{\text{input}} = [\mathbf{PE}(\mathbf{x}_i); \mathbf{f}_{\text{grid}}]$ [S1].
This supplies NeRF with extra features about scene geometry and appearance at sampled positions [S1].

**4. Joint Learning Stage:**
Both branches are optimized simultaneously using gradients from the final rendered image loss [S1].
*   The **NeRF Branch** uses the lightweight MLP to predict $\sigma$ and $\mathbf{c}$ from $\mathbf{F}_{\text{input}}$ and $\mathbf{d}$ [S1].
*   The **Grid Branch** is updated via backpropagation from the NeRF branch, regularizing the grid to produce more accurate and natural rendering results [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def positional_encoding(x, num_freq_bands=10):
    """Standard NeRF positional encoding using sine/cosine functions [G]."""
    pe = []
    for i in range(num_freq_bands):
        freq = 2.0 ** i
        pe.extend([math.sin(x[0]*freq), math.cos(x[0]*freq),
                   math.sin(x[1]*freq), math.cos(x[1]*freq),
                   math.sin(x[2]*freq), math.cos(x[2]*freq)])
    return pe

def bilinear_interp_2d(grid, x, y):
    """Bilinear interpolation on a 2D feature grid [G]."""
    ix, iy = int(x), int(y)
    fx, fy = x - ix, y - iy
    ix, iy = max(0, min(ix, grid.shape[0]-2)), max(0, min(iy, grid.shape[1]-2))
    c00 = grid[ix, iy]; c10 = grid[ix+1, iy]
    c01 = grid[ix, iy+1]; c11 = grid[ix+1, iy+1]
    w00 = (1-fx)*(1-fy); w10 = fx*(1-fy); w01 = (1-fx)*fy; w11 = fx*fy
    return w00*c00 + w10*c10 + w01*c01 + w11*c11

def retrieve_factorized_features(x, grid_xy, vec_z):
    """Retrieve features from factorized grid: bilinear interpolation on xy-plane + z-axis vector [G]."""
    f_xy = bilinear_interp_2d(grid_xy, x[0], x[1])
    iz = int(x[2]); fz = x[2] - iz
    iz = max(0, min(iz, len(vec_z)-2))
    f_z = (1-fz)*vec_z[iz] + fz*vec_z[iz+1]
    return f_xy + f_z

def mlp_forward(features, direction, weights):
    """Forward pass through lightweight MLP with ReLU and exp for density [G]."""
    h = features
    for W in weights[:-2]:
        h = [max(0.0, sum(h[j]*W[j][k] for j in range(len(h)))) for k in range(len(W[0]))]
    sigma = math.exp(sum(h[j]*weights[-2][j] for j in range(len(h))))
    color = [sum(h[j]*weights[-1][j] for j in range(len(h))) for _ in range(3)]
    return sigma, color

def volume_render(sigmas, colors, near, far, num_samples):
    """Standard alpha compositing volume rendering [G]."""
    t_vals = [near + i*(far-near)/num_samples for i in range(num_samples)]
    deltas = [t_vals[i+1]-t_vals[i] if i < num_samples-1 else (far-near)/num_samples for i in range(num_samples)]
    rendered_color = [0.0, 0.0, 0.0]
    T = 1.0
    for i in range(num_samples):
        alpha = 1.0 - math.exp(-sigmas[i]*deltas[i])
        weight = alpha * T
        rendered_color = [rendered_color[c] + weight*colors[i][c] for c in range(3)]
        T *= (1.0 - alpha + 1e-10)
    return rendered_color

def grid_nerf_forward(x, d, grid_xy, vec_z, mlp_weights):
    """Grid-NeRF forward pass."""
    f_grid = retrieve_factorized_features(x, grid_xy, vec_z)
    pe = positional_encoding(x)
    f_input = pe + f_grid
    sigma, color = mlp_forward(f_input, d, mlp_weights)
    return sigma, color

def joint_training_step(ray, grid_xy, vec_z, mlp_weights, loss_fn):
    """Single training step."""
    # Sampling concentrated near surface inferred from grid cues [G]
    points = [ray.origin + ray.direction * t for t in [ray.near + i*(ray.far-ray.near)/128 for i in range(128)]]
    sigmas, colors = [], []
    for p in points:
        sigma, color = grid_nerf_forward(p, ray.direction, grid_xy, vec_z, mlp_weights)
        sigmas.append(sigma)
        colors.append(color)
    rendered_color = volume_render(sigmas, colors, ray.near, ray.far, len(points))
    loss = loss_fn(rendered_color, ray.gt_color)
    return loss
```

### Parameters
*   **Grid Resolution:** Not given in the sources [S1].
*   **MLP Depth/Width:** Lightweight / small [S1][S2, unofficial]. Exact dimensions not given in the sources [S1].
*   **Memory Complexity:** Reduced from $O(N^3)$ to $O(N^2)$ due to factorization [S1].

### Edge Cases and Pitfalls
*   **Large Scale Scenes:** Pure NeRF methods underfit and blur; pure grid methods show noisy artifacts [S1]. Grid-NeRF mitigates both.
*   **Factorization Assumption:** The method assumes urban semantics are primarily on the ground ($xy$-plane) [S1].

## Subtopics

### HashNeRF
Uses hierarchical hash tables for multi-resolution encoding without explicit grid factorization [G]. Choose when memory efficiency is critical and scene is not strictly urban/ground-dominant [G].

### Surface-Relative Hash Encoding
Encodes features relative to a surface mesh rather than absolute coordinates [G]. Choose when explicit geometry is available and surface detail is paramount [G].

### HyRF (Hybrid Radiance Fields)
Combines explicit and implicit representations to balance rendering speed and geometric accuracy [G]. Choose when hybrid scene modeling is required [G].

### Neural Pre-integrated Emitters (NEF)
Models volumetric light transport using neural pre-integrated emitters [G]. Choose when simulating participating media or volumetric lighting is required [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** $O(N^2)$ due to factorization of 3D grid into 2D planes and shared vector [S1].
*   **Performance:** On a 2.7km² urban scene with 5k drone images, Grid-NeRF achieves PSNR 24.893 (Grid & NeRF branches) and 24.970, compared to NeRF-based PSNR 20.508 and Feature Grid-based PSNR 21.511 [S1].
*   **Strengths:** Photorealistic novel view synthesis for large-scale scenes; handles high-frequency details better than pure grids; avoids blurring of pure NeRFs [S1].
*   **Limitations:** Requires joint optimization of two branches; factorization may limit applicability to non-urban scenes [S1].
*   **When to use:** Large urban scene reconstruction from aerial/drone imagery [S1].
*   **When not to use:** Small-scale scenes where overhead is unjustified; non-urban scenes where ground-dominant assumption fails [S1].

## Background
Grid-NeRF was proposed by researchers from CUHK, MPI Informatics, Zhejiang University, and Adobe Research [S1]. It was published in CVPR 2023 [S1]. The work addresses the limitations of existing NeRF-based and feature grid-based methods for large-scale urban scene rendering [S1]. It builds on the idea of combining explicit grid representations with implicit neural fields [S1].

## Sources
[S1] Grid-guided Neural Radiance Fields for Large Urban Scenes, https://doi.org/10.1109/cvpr52729.2023.00802, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
