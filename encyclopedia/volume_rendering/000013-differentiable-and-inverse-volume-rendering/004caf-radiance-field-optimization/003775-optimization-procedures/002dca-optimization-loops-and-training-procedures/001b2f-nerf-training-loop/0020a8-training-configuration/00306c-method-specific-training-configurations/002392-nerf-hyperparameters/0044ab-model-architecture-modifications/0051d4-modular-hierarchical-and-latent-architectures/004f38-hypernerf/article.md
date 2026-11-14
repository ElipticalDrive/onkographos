# HyperNeRF

Also called: Hyper-Space Template, Hyper-space Neural Radiance Fields, HyperNeRF Template

## Overview
HyperNeRF is a method for modeling non-rigidly deforming scenes with topological variations (e.g., openings, closings, or transient effects) by extending the canonical 3D coordinate space of a Neural Radiance Field (NeRF) into a higher-dimensional ambient space [S2]. Unlike continuous deformation fields which struggle with discontinuities, HyperNeRF represents each observation frame as a non-planar slice through this hyperdimensional template, allowing smooth topological changes without requiring discontinuous deformation mappings [S2]. It sets itself apart from sibling methods like Strata-NeRF by explicitly using higher-dimensional ambient coordinates and deformable slicing manifolds to handle topology, rather than stratified sampling or static hierarchical structures [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ images $\{I_i\}_{i=1}^N$ with known camera poses and viewing directions. Each image $i$ is associated with a latent deformation code $\omega_i$ and a latent appearance code $\psi_i$.
*   **Outputs:** A rendered image for a given camera pose and time/moment, or a continuous 4D+ representation of the scene.

**Data Structures:**
1.  **Deformation Field MLP ($W$):** Maps observation-space coordinates to canonical-space coordinates.
    *   Input: $(x, \omega_i)$ where $x \in \mathbb{R}^3$ is the spatial coordinate and $\omega_i \in \mathbb{R}^k$ is the per-observation latent deformation code.
    *   Output: $(r, v) \in \mathfrak{se}(3)$, representing rotation and translation parameters for the rigid body transformation.
2.  **Slicing Surface MLP ($H$):** Defines the non-planar slicing manifold in the ambient space.
    *   Input: $(x, \omega_i)$ where $x \in \mathbb{R}^3$ is the spatial coordinate and $\omega_i$ is the latent deformation code.
    *   Output: $w \in \mathbb{R}^W$, the position along the $W$ ambient dimensions.
3.  **Template NeRF MLP ($F$):** The core radiance field defined in the hyper-space.
    *   Input: $(x', w, d, \psi_i)$ where $x'$ is the canonical spatial coordinate, $w \in \mathbb{R}^W$ is the ambient coordinate, $d \in \mathbb{R}^3$ is the view direction, and $\psi_i \in \mathbb{R}^p$ is the per-frame appearance embedding.
    *   Output: $(c, \sigma)$ where $c \in \mathbb{R}^3$ is the color and $\sigma \in \mathbb{R}$ is the volume density.

**Numbered Steps:**

1.  **Initialize Parameters:**
    *   Define the number of ambient dimensions $W$. Typical value: $W=2$ [S2].
    *   Define the positional encoding bandwidth $m$. For spatial input $x$ to $H$, $m=6$; for ambient input $w$ to $F$, $m=1$ [S2].
    *   Initialize MLPs $W$, $H$, and $F$ with random weights.
    *   Initialize latent codes $\omega_i$ and $\psi_i$ for each input image $i$.

2.  **Map Observation to Canonical Space (Deformation):**
    *   For a ray originating at camera position with direction $d$, sample points $x_{obs}$ in observation space.
    *   Compute the deformation field output: $(r, v) = W(x_{obs}, \omega_i)$.
    *   Construct the rigid transformation matrix $T_{rigid}$ from $(r, v) \in \mathfrak{se}(3)$.
    *   Transform the point to canonical space: $x_{can} = T_{rigid} \cdot x_{obs}$ [S2].

3.  **Compute Ambient Coordinate (Slicing):**
    *   Query the slicing surface MLP to determine the ambient coordinate corresponding to the spatial location: $w = H(x_{can}, \omega_i)$.
    *   Note: The slicing surface $H$ allows the slice to be curved, mapping different spatial locations to different parts of the ambient space, enabling compact representation of topological changes [S2].

4.  **Query Template NeRF:**
    *   Concatenate the canonical spatial coordinate $x_{can}$ and the ambient coordinate $w$ to form the hyper-space coordinate $(x_{can}, w) \in \mathbb{R}^{3+W}$.
    *   Apply positional encoding $\gamma$ to $x_{can}$ and $w$.
        *   For $x_{can}$: $\gamma(x_{can})$ uses $m=6$ frequencies [S2].
        *   For $w$: $\gamma(w)$ uses $m=1$ frequency [S2].
    *   Concatenate encoded spatial, ambient, view direction $d$, and appearance code $\psi_i$.
    *   Compute radiance and density: $(c, \sigma) = F(\gamma(x_{can}), \gamma(w), d, \psi_i)$ [S2].

5.  **Volume Rendering:**
    *   Perform standard volume rendering along the ray using the sampled $(c, \sigma)$ pairs to compute the final pixel color [S2].

**Formulas:**

The sinusoidal positional encoding is defined as [S2]:
$$
\gamma(x) = [\sin(x), \cos(x), \dots, \sin(2^{m-1}x), \cos(2^{m-1}x)]^T
$$
where $m$ is the number of sinusoids.

The windowed positional encoding used for coarse-to-fine optimization is [S2]:
$$
\gamma_\alpha(x) = [w_0(\alpha)\sin(x), \dots, w_{m-1}(\alpha)\cos(2^{m-1}x)]^T
$$
where the window function $w_j(\alpha)$ is:
$$
w_j(\alpha) = \frac{1 - \cos(\pi \cdot \text{clamp}(\alpha - j, 0, 1))}{2}
$$
and $\alpha \in [0, m]$ is linearly increased during training to narrow the Neural Tangent Kernel (NTK) bandwidth [S2].

The template NeRF is represented as [S2]:
$$
F: (x, w, d, \psi_i) \rightarrow (c, \sigma)
$$
where $x$ is spatial coordinate, $w$ is ambient coordinate, $d$ is view direction, and $\psi_i$ is latent appearance code.

The deformable slicing surface is defined as [S2]:
$$
H: (x, \omega_i) \rightarrow w
$$
where $x$ is spatial position, $w$ is position along the ambient axes, and $\omega_i$ is the per-input latent embedding.

**Parameters:**
*   $W$: Number of higher (ambient) dimensions. Typical value: 2 [S2].
*   $m$: Hyper-parameter for positional encoding bandwidth. $m=6$ for spatial input to $H$, $m=1$ for ambient input to $F$ [S2].
*   $\alpha$: Optimization schedule parameter for NTK bandwidth, starting at 0 and linearly increasing to $m$ over 10k iterations [S2].

**Edge Cases and Pitfalls:**
*   **Topological Changes:** Continuous deformation fields (e.g., Nerfies) fail at topological changes (mouth opening, peeling) because they require discontinuities in the deformation field, which band-limited MLPs cannot model easily [S2]. HyperNeRF avoids this by using ambient dimensions [S2].
*   **Regularization:** The elastic regularization loss from Nerfies (penalizing non-unit singular values of the deformation Jacobian) is ill-suited for scenes with topological variations as it violates the as-rigid-as-possible assumption [S2]. HyperNeRF replaces this with an optimization strategy encouraging smooth behavior in higher dimensions [S2].
*   **Ambient Dimension Count:** Increasing $W$ beyond 2 did not improve performance in the cited experiments, suggesting $W=2$ is sufficient for the tested sequences [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp
from jax import grad, jit

def positional_encoding(x, m):
    """
    Computes sinusoidal positional encoding.
    x: input coordinates (..., D)
    m: number of frequency bands
    Returns: encoded vector (..., D * 2 * m)
    """
    freq_bands = jnp.power(2.0, jnp.arange(m))
    # x shape: (..., D), freq_bands shape: (m,)
    # Output shape: (..., D, m, 2) -> reshape to (..., D * 2 * m)
    scaled_x = x[..., None, None] * freq_bands[None, None, :]
    encoding = jnp.concatenate([jnp.sin(scaled_x), jnp.cos(scaled_x)], axis=-1)
    return encoding.reshape(*x.shape[:-1], -1)

def windowed_positional_encoding(x, m, alpha):
    """
    Computes windowed positional encoding for coarse-to-fine optimization.
    alpha: current bandwidth parameter, 0 <= alpha <= m
    """
    freq_bands = jnp.power(2.0, jnp.arange(m))
    scaled_x = x[..., None, None] * freq_bands[None, None, :]
    
    # Window function w_j(alpha)
    # j is index 0..m-1
    j_indices = jnp.arange(m)
    # clamp(alpha - j, 0, 1)
    clamp_val = jnp.clip(alpha - j_indices, 0.0, 1.0)
    weights = (1.0 - jnp.cos(jnp.pi * clamp_val)) / 2.0
    
    # Apply weights to sin/cos components
    sin_part = jnp.sin(scaled_x) * weights[None, None, :]
    cos_part = jnp.cos(scaled_x) * weights[None, None, :]
    encoding = jnp.concatenate([sin_part, cos_part], axis=-1)
    return encoding.reshape(*x.shape[:-1], -1)

def hypernerf_forward(x_obs, d, omega_i, psi_i, params):
    """
    Single forward pass for a point on a ray.
    x_obs: observation space coordinate (3,)
    d: view direction (3,)
    omega_i: latent deformation code for frame i
    psi_i: latent appearance code for frame i
    params: dictionary containing weights for W, H, F
    """
    # 1. Deformation Field W: (x, omega) -> (r, v) in se(3)
    # Assume W is a simple MLP for brevity; in practice, use JAX nn.Dense
    se3_params = params['W'](jnp.concatenate([x_obs, omega_i]))
    r, v = se3_params[:3], se3_params[3:]
    # Construct rigid transform T_rigid from r, v
    # [G] Placeholder for se(3) to rigid transform matrix construction
    T_rigid = se3_to_rigid(r, v) 
    
    # Transform to canonical space
    x_can = T_rigid @ x_obs
    
    # 2. Slicing Surface H: (x, omega) -> w (ambient coord)
    # Input to H is canonical spatial coord x_can
    w = params['H'](jnp.concatenate([x_can, omega_i]))
    
    # 3. Template NeRF F: (x, w, d, psi) -> (c, sigma)
    # Positional encoding for spatial x_can (m=6)
    pe_x = windowed_positional_encoding(x_can, m=6, alpha=params['alpha'])
    # Positional encoding for ambient w (m=1)
    pe_w = windowed_positional_encoding(w, m=1, alpha=params['alpha'])
    # View direction encoding (standard NeRF, m=10 typically, not specified in notes for d)
    # [G] Assuming standard NeRF PE for d with m=10 as per original NeRF, 
    # though notes only specify m for x and w.
    pe_d = positional_encoding(d, m=10) 
    
    # Concatenate inputs to F
    f_input = jnp.concatenate([pe_x, pe_w, pe_d, psi_i])
    
    # Query F
    c, sigma = params['F'](f_input)
    
    return c, sigma

def se3_to_rigid(r, v):
    """
    Converts skew-symmetric matrix parameters (r) and translation (v) to 4x4 matrix.
    [G] Standard conversion from Lie algebra se(3) to Lie group SE(3).
    """
    # r is (3,), representing skew-symmetric matrix elements
    # Construct skew-symmetric matrix R_skew
    R_skew = jnp.array([[0, -r[2], r[1]],
                        [r[2], 0, -r[0]],
                        [-r[1], r[0], 0]])
    # Exponential map for rotation (Rodrigues' formula)
    # [G] Placeholder for jax implementation of exp_map_so3
    R = exp_map_so3(R_skew)
    # Translation is just v
    T = jnp.eye(4)
    T[:3, :3] = R
    T[:3, 3] = v
    return T
```

## Subtopics

### Deformation-based NeRFs (e.g., Nerfies)
Use a continuous deformation field to map observation space to canonical space. They struggle with topological changes because MLPs with positional encoding are band-limited and cannot model the discontinuities required for topology changes [S2].

### Latent-conditioned NeRFs (e.g., NeRF-W, Instant-NGP with latents)
Condition the MLP on a latent code per frame to handle appearance or shape variations. They often require substantial regularization or additional supervision to disambiguate shape from appearance [S2].

### HyperNeRF
Combines deformation and latent conditioning. It uses deformation for well-behaved motion modeling and extends the input space to higher dimensions with a deformable slicing surface to handle topology. It requires less regularization than pure latent methods and handles topology better than pure deformation methods [S2].

## Operating characteristics
*   **Time Complexity:** Training is computationally intensive. Training at half 1080p resolution (960x540) takes roughly 8 hours on 4 TPU v4s for 250k iterations [S2]. Full HD training takes roughly 64 hours for 1M iterations [S2].
*   **Memory Complexity:** Depends on the size of the MLPs ($W, H, F$) and the number of latent codes. The use of higher dimensions ($W=2$) adds minimal overhead compared to the base NeRF MLPs [S2].
*   **Performance:** HyperNeRF reduces average error rates by 4.1% for interpolation and 8.6% for novel-view synthesis compared to Nerfies, as measured by LPIPS [S2].
*   **Strengths:** Capable of synthesizing views in scenes with significant topological changes (e.g., human mouth opening, banana peeling) [S1]. Produces sharper, higher-quality results with fewer artifacts than strictly deformation-based methods [S2].
*   **Limitations:** Requires careful tuning of the optimization schedule (NTK bandwidth) to ensure smooth behavior in higher dimensions [S2]. Increasing ambient dimensions beyond $W=2$ did not improve performance in tested sequences [S2].

## Background
HyperNeRF was introduced by Kevin Park et al. in June 2021 [S1]. It builds upon Nerfies (Park et al., 2020) by extending the canonical space to a higher dimension and using a slicing MLP to describe how to return to the 3D representation using ambient space coordinates [S1]. The method is inspired by level set methods pioneered in the late 1980s by Osher and Sethian, which model moving scenes as static objects in a higher-dimensional ambient space [S2]. HyperNeRF generalizes classical level sets by allowing any number of ambient dimensions and using MLPs to represent general, curved slicing manifolds rather than hyperplanes [S2]. It is used for photorealistic free-viewpoint reconstruction of challenging deforming scenes from monocular video, with applications in virtual/augmented reality and digital entertainment [S1, S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] HyperNeRF, https://doi.org/10.1145/3478513.3480487, fetched 2026-10-28
