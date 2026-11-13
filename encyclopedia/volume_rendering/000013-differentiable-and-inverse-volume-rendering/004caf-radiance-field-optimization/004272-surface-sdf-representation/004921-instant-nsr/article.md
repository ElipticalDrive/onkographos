# Instant-NSR

Also called: Neural Surface Reconstruction, Neural Surface Reconstruction with Hash

## Overview
Instant-NSR is a neural surface reconstruction method that accelerates training using Hash encoding while incorporating a Signed Distance Function (SDF) to improve geometry quality [S1]. It builds on [Radiance-field optimization](44-radiance-field-optimization.md) by adopting the SDF architecture of [NeuS](6060-neus.md) but replaces the multi-resolution hash grid density representation of [Instant-NGP](44-radiance-field-optimization.md) with a TSDF-based approach to ensure numerical stability [S1]. Unlike [VolSDF](10715-volsdf.md) which uses continuous coordinate networks, Instant-NSR leverages the fast lookup of hash grids, making it significantly faster to train while maintaining high-fidelity geometric reconstruction [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of calibrated images with known camera poses; a hash grid structure for spatial encoding; a neural network for SDF and color prediction.
*   **Outputs:** A rendered image; a reconstructed 3D mesh (extracted via Marching Cubes from the learned SDF); the trained network weights.

**Data Structures:**
1.  **Hash Grid:** A multi-resolution hash table mapping spatial coordinates to feature vectors. This is identical to the structure used in [Instant-NGP](44-radiance-field-optimization.md). It consists of $L$ levels, where each level $l$ has a hash table of size $2^{l+5}$ (or similar power-of-two sizing as per Instant-NGP conventions, though exact table sizes are implementation-specific) mapping 3D coordinates to $F$-dimensional feature vectors [S1].
2.  **Neural Network:** A Multi-Layer Perceptron (MLP) that takes the concatenated hash features as input. It outputs a scalar SDF value $f(\mathbf{x})$ and a 3D color vector $\mathbf{c}$. The architecture mirrors [NeuS](6060-neus.md) rather than the density MLP of Instant-NGP [S1].

**Numbered Steps:**
1.  **Hash Encoding:** For a query point $\mathbf{x}$, compute its hash indices across all $L$ resolution levels. Retrieve the corresponding feature vectors from the hash grid and concatenate them into a single feature vector $\mathbf{v}$ [S1].
2.  **SDF and Color Prediction:** Pass $\mathbf{v}$ through the MLP to predict the raw SDF value $f(\mathbf{x})$ and the color $\mathbf{c}(\mathbf{x})$ [S1].
3.  **TSDF Truncation:** Apply a sigmoid function $\pi(\cdot)$ to the raw SDF output to truncate it to the range $[-1, 1]$. This prevents numerical overflow during volume rendering [S1]:
    $$ \pi(f(\mathbf{x})) = \frac{1 - e^{-bf(\mathbf{x})}}{1 + e^{-bf(\mathbf{x})}} $$
    where $b$ is a scaling factor [S1].
4.  **Volume Rendering:** Compute the weight $w(t)$ for a sample point at distance $t$ along a ray using the cumulative density distribution $\phi_b$. Replace the standard NeRF/NeuS formulation with the TSDF-modified version to ensure stability [S1]:
    $$ \phi_b(f(\mathbf{x})) = \frac{b e^{-b\pi(f(\mathbf{x}))}}{(1 + e^{-b\pi(f(\mathbf{x}))})^2} $$
    The weight $w(t)$ is computed as:
    $$ w(t) = \phi_b(f(p(t))) \prod_{i=0}^{t-1} (1 - \phi_b(f(p(i)))) $$
    where $p(t)$ is the position along the ray [S1].
5.  **Rendering:** Composite the color and transmittance along the ray to produce the final pixel color [S1].
6.  **Loss Computation:** Calculate the total loss as a weighted sum of rendering loss and Eikonal loss [S1].
7.  **Optimization:** Update the hash grid features and MLP weights using gradient descent [S1].

**Formulas:**
*   **TSDF Mapping:**
    $$ \pi(f(\mathbf{x})) = \frac{1 - e^{-bf(\mathbf{x})}}{1 + e^{-bf(\mathbf{x})}} $$
    This maps the raw SDF $f(\mathbf{x})$ to the interval $(-1, 1)$ [S1].
*   **Weight Distribution:**
    $$ \phi_b(f(\mathbf{x})) = \frac{b e^{-b\pi(f(\mathbf{x}))}}{(1 + e^{-b\pi(f(\mathbf{x}))})^2} $$
    This is the derivative of the CDF used for sampling weights, modified to use the truncated SDF $\pi(f(\mathbf{x}))$ instead of the raw $f(\mathbf{x})$ [S1].
*   **Eikonal Loss:**
    $$ L_{eik} = \frac{1}{M} \sum_{j=1}^{M} (\|\nabla f(x_j)\|_2^2 - 1)^2 $$
    where $M$ is the number of sampled points and $\nabla f(x_j)$ is the gradient of the SDF at point $x_j$ [S1].
*   **Total Loss:**
    $$ L_{total} = \alpha L_{render} + \beta L_{eik} $$
    where $\alpha$ and $\beta$ are hyperparameters [S1].

**Parameters:**
*   **$\alpha$ (Alpha):** Weighting factor for the rendering loss. Chosen via cross-validation [S1].
*   **$\beta$ (Beta):** Weighting factor for the Eikonal loss. Chosen via cross-validation [S1].
*   **$b$:** Scaling factor in the TSDF and weight distribution formulas. Its value affects numerical stability; large $b$ without TSDF causes divergence [S1].
*   **TSDF Range:** The truncated SDF values are constrained between -1 and 1 [S1].

**Edge Cases and Pitfalls:**
*   **Numerical Instability:** In standard SDF-based methods, increasing $b$ causes $-bf(\mathbf{x})$ to become a large positive number, making $e^{-bf(\mathbf{x})}$ approach infinity. This leads to numerical overflow and network divergence [S1].
*   **Solution:** The TSDF truncation via the sigmoid function $\pi(\cdot)$ bounds the SDF values to $[-1, 1]$, preventing the exponent from exploding and ensuring stable training [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def instant_nsr_forward(x, hash_grid, mlp, b):
    """
    Forward pass for Instant-NSR.
    x: 3D point coordinates (N, 3)
    hash_grid: Multi-resolution hash grid structure
    mlp: Neural network (MLP) for SDF and color
    b: Scaling factor for TSDF
    """
    # 1. Hash Encoding
    # Retrieve features from hash grid at multiple resolutions
    features = []
    for level in hash_grid.levels:
        indices = compute_hash_indices(x, level.resolution)
        features.append(hash_grid.get(level, indices))
    v = concatenate(features, dim=-1)
    
    # 2. MLP Prediction
    sdf_raw, color = mlp(v)
    
    # 3. TSDF Truncation
    # pi(f(x)) = (1 - exp(-b*f(x))) / (1 + exp(-b*f(x)))
    exp_term = torch.exp(-b * sdf_raw)
    sdf_truncated = (1 - exp_term) / (1 + exp_term)
    
    # 4. Weight Distribution for Volume Rendering
    # phi_b = b * exp(-b * pi(f(x))) / (1 + exp(-b * pi(f(x))))^2
    exp_trunc = torch.exp(-b * sdf_truncated)
    phi_b = (b * exp_trunc) / ((1 + exp_trunc) ** 2)
    
    return sdf_truncated, color, phi_b, sdf_raw

def compute_eikonal_loss(sdf_raw, points):
    """
    Compute Eikonal loss for SDF gradient magnitude constraint.
    sdf_raw: Raw SDF output (N,)
    points: Query points (N, 3)
    """
    # Compute gradient of raw SDF w.r.t input points
    # Note: Gradient is taken w.r.t raw SDF to enforce ||grad f|| = 1
    grad_sdf = torch.autograd.grad(
        sdf_raw.sum(), points, create_graph=True, retain_graph=True
    )[0]
    grad_norm = torch.norm(grad_sdf, dim=1, p=2)
    loss = torch.mean((grad_norm ** 2 - 1) ** 2)
    return loss
```

## Subtopics

### NeuS
Uses a continuous coordinate network (MLP) for SDF representation without hash grids. Slower training but serves as the architectural baseline for the SDF formulation used in Instant-NSR [6060-neus.md].

### Surface NeRF
Another SDF-based method that uses a differentiable renderer. It does not employ hash encoding, resulting in slower convergence compared to Instant-NSR [7385-surface-nerf.md].

### VolSDF
Represents geometry using a learned SDF with a specific volume rendering formulation. It relies on standard MLPs for coordinate encoding, lacking the acceleration provided by Instant-NSR's hash grids [10715-volsdf.md].

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than continuous-coordinate SDF methods like [VolSDF](10715-volsdf.md) and [NeuS](6060-neus.md) due to the use of hash encoding for spatial lookups, similar to [Instant-NGP](44-radiance-field-optimization.md) [S1].
*   **Memory Complexity:** Depends on the size of the hash grid. Multi-resolution hash grids require substantial memory for large scenes, but less than dense voxel grids [S1].
*   **Strengths:** High-quality geometry reconstruction; fast training speed; numerical stability via TSDF [S1].
*   **Limitations:** Relies on the quality of the hash grid initialization; TSDF truncation may limit the effective range of the SDF if not scaled properly [S1].
*   **When to use:** When fast training and high-fidelity geometry are required, and the scene fits within the memory constraints of the hash grid [S1].
*   **When not to use:** For scenes requiring very large spatial extents that exceed the hash grid's coverage, or when continuous coordinate networks are preferred for simplicity [S1].

## Background
Instant-NSR was introduced as a method to combine the speed of [Instant-NGP](4000-instant-ngp.md) with the geometric accuracy of SDF-based rendering. It addresses the convergence issues that arise when applying SDF representations to hash coding frameworks by introducing Truncated SDF (TSDF) [S1]. The method has been explored in contexts such as agricultural plant phenotyping, demonstrating competitive performance with commercial MVS software like Reality Capture for 3D reconstruction of plants [S1].

## Sources
[S1] High-fidelity 3D Reconstruction of Plants using Neural Radiance Field, https://arxiv.org/abs/2311.04154v1, fetched 2026-10-27
