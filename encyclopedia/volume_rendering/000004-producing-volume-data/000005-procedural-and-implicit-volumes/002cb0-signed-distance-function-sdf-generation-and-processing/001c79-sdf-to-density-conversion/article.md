# SDF to Density Conversion

Also called: Glass Ball IOR Distribution, Implicit Surface Density Construction, SDF to Density/Feature Volume, SDF-based Density, SDFVolume, signed distance to density conversion

## Overview
SDF to Density Conversion transforms a Signed Distance Function (SDF) value $d(\mathbf{x})$ into a volumetric density $\sigma(\mathbf{x})$ or intermediate opacity $\rho(\mathbf{x})$ suitable for differentiable volume rendering [S4, S5, unofficial][S9]. It solves the problem of rendering continuous 3D implicit surfaces without explicit mesh extraction by treating the zero-level set of the SDF as a volumetric boundary [G]. Unlike standard volume rendering which takes density directly, this method derives density from the distance to the implicit surface, enabling the representation of open surfaces and varying thickness through exponential or sigmoidal falloffs [S1][S4, S5, unofficial]. It sets itself apart from its siblings by providing the core mathematical mapping function, whereas [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md) provides the complete rendering pipeline and [Volume SDF Rendering](1344-volume-sdf-rendering.md) focuses on neural integration with analytic backends [S3].

## How it works
**Inputs:**
- Signed distance value $d$ (or $s = -d$) at a point $\mathbf{x}$ along a ray [S4, S5, unofficial].
- Parameters $\alpha$ (scaling factor) and $\beta$ (falloff/sharpness control) [S4, S5, unofficial].
- Ray segment length $\Delta t$ (distance between consecutive sample points) [S4, S5, unofficial].
- Optional: `rainbow` flag for color generation [S4, S5, unofficial].

**Outputs:**
- Volumetric density $\sigma$ (or extinction coefficient) [S4, S5, unofficial].
- Intermediate density $\rho$ (opacity probability) [S4, S5, unofficial].
- Feature/color vector $c$ [S4, S5, unofficial].

**Data Structures:**
- Scalar values for $d, \alpha, \beta, \Delta t$.
- Vectors for sample points $\mathbf{x}$ and features $c$.

**Steps:**

1. **Compute Signed Distance:** Evaluate the SDF at the sample point $\mathbf{x}$ to get $d(\mathbf{x})$ [S5, unofficial].
2. **Normalize Distance:** Compute $s = -d(\mathbf{x})$. This ensures $s > 0$ inside the object and $s < 0$ outside, aligning with the exponential falloff definition [S5, unofficial].
3. **Compute Intermediate Density ($\psi$ or $\rho$):** Apply the piecewise exponential function [S4, S5, unofficial]:
   $$
   \psi(s) = \begin{cases} 
   0.5 \cdot \exp(s / \beta) & \text{if } s \le 0 \\
   1 - 0.5 \cdot \exp(-s / \beta) & \text{if } s > 0 
   \end{cases}
   $$
   Scale by $\alpha$ to get the intermediate density $\rho$:
   $$
   \rho = \alpha \cdot \psi(s)
   $$
   Here, $\beta$ controls the sharpness of the transition; smaller $\beta$ yields sharper surfaces [S5, unofficial]. $\alpha$ controls overall opacity [S4, unofficial].
4. **Compute Volumetric Density ($\sigma$):** Convert intermediate density to volumetric density using the ray segment length $\Delta t$ [S4, S5, unofficial]:
   $$
   \sigma = -\frac{\ln(1 - \rho)}{\Delta t}
   $$
   This step ensures that the probability of absorption over the segment $\Delta t$ matches $\rho$ [S5, unofficial].
5. **Compute Color/Feature (Optional):**
   - If `rainbow` is true, compute base color from position:
     $$
     \text{base\_color} = \text{clamp}(|\mathbf{x} - \mathbf{c}|, 0.02, 0.98)
     $$
     where $\mathbf{c}$ is the center [S4, S5, unofficial].
   - Else, $\text{base\_color} = 1.0$ [S4, S5, unofficial].
   - Final feature:
     $$
     c = \text{base\_color} \cdot \text{feature\_param} \cdot \rho
     $$
     (Note: Some implementations multiply by $\rho$ or use $\rho$ for weighting [S4, S5, unofficial]).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def sdf_to_density(signed_distance, alpha, beta):
    """
    Converts signed distance to intermediate density.
    signed_distance: torch.Tensor of shape (N,)
    alpha: float, scaling factor
    beta: float, falloff sharpness
    """
    # s = -d, so s > 0 inside, s < 0 outside
    s = -signed_distance
    
    # Piecewise exponential
    # Inside (s <= 0): 0.5 * exp(s / beta)
    # Outside (s > 0): 1 - 0.5 * exp(-s / beta)
    psi = torch.where(
        s <= 0.0,
        0.5 * torch.exp(s / beta),
        1.0 - 0.5 * torch.exp(-s / beta)
    )
    
    # Scale by alpha
    density_intermediate = alpha * psi
    return density_intermediate

def compute_volumetric_density(density_intermediate, delta_t):
    """
    Converts intermediate density to volumetric density sigma.
    density_intermediate: torch.Tensor of shape (N,)
    delta_t: torch.Tensor of shape (N,), segment lengths
    """
    # Avoid log(0) or log(negative) by clamping density_intermediate to [0, 1)
    # In practice, density_intermediate should be < 1 if alpha is tuned correctly.
    # If density_intermediate >= 1, sigma becomes infinite or NaN.
    clamped_density = torch.clamp(density_intermediate, min=0.0, max=0.9999)
    
    sigma = -torch.log(1.0 - clamped_density) / delta_t
    return sigma

def render_step(ray_bundle, sdf_fn, alpha, beta, rainbow=False, center=None, feature_param=None):
    """
    Full rendering step for SDF to density conversion.
    ray_bundle: dict with 'sample_points' (N, 3) and 'deltas' (N,)
    sdf_fn: callable returning signed distance
    """
    # 1. Extract sample points and depth values
    points = ray_bundle['sample_points']
    deltas = ray_bundle['deltas']
    
    # 2. Handle last segment padding if necessary (e.g., ray termination)
    # If the last delta is 0 or undefined, pad with a large value to prevent division by zero
    if deltas[-1] == 0 or torch.isnan(deltas[-1]):
        deltas[-1] = 1e10
        
    # 3. Evaluate SDF
    signed_distances = sdf_fn(points)
    
    # 4. Convert to intermediate density
    density_inter = sdf_to_density(signed_distances, alpha, beta)
    
    # 5. Compute volumetric density
    sigma = compute_volumetric_density(density_inter, deltas)
    
    # 6. Compute base color
    if rainbow:
        if center is None:
            center = torch.zeros(3, device=points.device)
        base_color = torch.abs(points - center)
        base_color = torch.clamp(base_color, min=0.02, max=0.98)
    else:
        base_color = torch.ones_like(points)
        
    # 7. Compute final feature
    if feature_param is None:
        feature_param = torch.ones(1, device=points.device)
    feature = base_color * feature_param * density_inter # Or sigma, depending on specific impl [S4, S5, unofficial]
    
    return {
        'density': sigma,
        'feature': feature,
        'depth': ray_bundle['depth'] # Assuming depth is passed or computed
    }
```

**Parameters:**
- $\alpha$: Scalar multiplier for intermediate density, controls overall opacity. Typical values are learned or tuned constants [S4, S5, unofficial].
- $\beta$: Scalar controlling exponential falloff rate. Smaller values yield sharper surfaces [S5, unofficial]. Typical values are learned or tuned constants [S5, unofficial].
- `rainbow`: Boolean flag. If true, computes base color from position; else 1.0 [S4, S5, unofficial].
- `sharpness` (in Glass Ball IOR): Controls gradient of IOR transition, value 7 in example [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** The last segment's delta is padded with $10^{10}$ to prevent division by zero when the ray exits the sampling volume [S4, S5, unofficial].
- **Density > 1:** If $\rho \ge 1$, $\ln(1-\rho)$ is undefined. Ensure $\alpha$ is tuned such that $\rho < 1$ [S5, unofficial].
- **Precision:** Floating-point precision issues may arise for very small $\beta$ or large distances [G].

## Subtopics

### [Volume SDF Rendering](1344-volume-sdf-rendering.md)
Volume SDF Rendering is a neural rendering technique that integrates color and density along rays sampled from an implicit SDF. It solves the problem of rendering complex 3D geometry from multi-view images by converting the continuous SDF into a volumetric density field, allowing standard volume rendering to capture both surface appearance and thin structures. Unlike standard volume rendering which takes density directly, this method derives density from the distance to the implicit surface, enabling the representation of open surfaces and varying thickness through the Laplace distribution CDF. It differs from [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md) primarily in its specific parameterization of the density transition and its integration with analytic or neural SDF backends.

### [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md)
VolSDF Volume Rendering synthesizes images from a neural implicit surface representation by converting Signed Distance Function (SDF) values into volumetric density [G]. It solves the problem of rendering continuous 3D surfaces without explicit mesh extraction by treating the zero-level set of the SDF as a volumetric boundary [G]. It sets itself apart from its parent [SDF to Density Conversion](1282-sdf-to-density-conversion.md) by providing the complete rendering pipeline (sampling, transmittance, aggregation) rather than just the conversion function, and from its sibling [Volume SDF Rendering](1344-volume-sdf-rendering.md) by using a differentiable neural SDF with stratified sampling and alpha-compositing rather than discrete voxel rasterization.

### [Box with Smoothed/Eroded Edges](3417-box-with-smoothed-eroded-edges.md)
This method converts a signed distance field (SDF) value into a volumetric density for a box primitive, applying a sigmoid-based erosion at the boundaries to create smooth, anti-aliased edges and adding Perlin noise to roughen the surface. It differs from standard SDF-to-density conversions by explicitly expanding the box geometry to define an erosion zone and combining axis-aligned sigmoid falloffs with spatial noise, rather than relying solely on a global distance threshold or simple alpha blending. It builds on [SDF to Density Conversion](1282-sdf-to-density-conversion.md).

### [SDF Alpha Computation](3643-sdf-alpha-computation.md)
Computes the output alpha for a Signed Distance Field (SDF) given the Screen Space Ratio (SSR), Signed Distance (SD), Signed Distance Ratio (SDR), isoPerimeter (dilation/contraction), and softness. It converts a signed distance value into a density value using alpha and beta parameters, building on [SDF to Density Conversion](1282-sdf-to-density-conversion.md).

### [Gray Transform (G_o, G_s)](10492-gray-transform-g-o-g-s.md)
The Gray Transform converts a discrete object description or a continuous surface description into a volumetric scene (density field) suitable for physical modeling, manufacturing, or volume rendering. It distinguishes itself from sibling methods by explicitly defining density values based on interior/exterior status or signed distance to a surface, rather than relying on implicit volume generation or alpha-compositing parameters directly.

### Procedural Index of Refraction Distribution
Simulates mirages by creating a non-uniform distribution of the index of refraction (IOR) in air, where IOR decreases with increasing temperature [S2, unofficial]. It creates a normal lapse layer ($index = 0.95 \cdot 2^z$) and an inversion layer ($index = 0.8^z$), combines them via minimum, and sets IOR to 1.0 for $z < 0$ [S2, unofficial]. It clips IOR to $[1, 1.5]$ and uses a sharpness parameter (7) to control the gradient of the IOR transition from a glass ball surface [S2, unofficial].

### FBM-based Implicit Density Definition
Defines an implicit volumetric object where density $\rho$ is derived from a Fractional Brownian Motion (FBM) field $f$ [S6]. Density is zero when $f(\mathbf{p}) < v$, and remapped via a positive function $h$ when $f(\mathbf{p}) \ge v$ [S6]. The 0-isosurface of $\rho$ corresponds to the $v$-isosurface of $f$ [S6]. Pitfall: If $h$ is not positive, the correspondence breaks [S6].

### Density-based Volume Region
Extends VolumeRegion by scaling absorption, scattering, extinction, and emission coefficients by a spatially varying density function $density(\mathbf{pt})$ [S7, unofficial]. Formulas: $\sigma_a(\mathbf{pt}) = \sigma_a \cdot density(\mathbf{pt})$, $\sigma_s(\mathbf{pt}) = \sigma_s \cdot density(\mathbf{pt})$, $\sigma_t(\mathbf{pt}) = (\sigma_a + \sigma_s) \cdot density(\mathbf{pt})$, $L_{ve}(\mathbf{pt}) = L_e \cdot density(\mathbf{pt})$ [S7, unofficial]. Uses Henyey-Greenstein phase function with parameter $g_{scatter}$ [S7, unofficial].

### Smeared Heaviside Function for Mean Curvature
A smeared-out Heaviside function $H$ used to compute mean curvature for regularization [S8]. Defined piecewise based on threshold $\epsilon_H$: $H(\hat{\phi}) = 0$ if $\hat{\phi} < -\epsilon_H$; $1/2 + \hat{\phi}/(2\epsilon_H) + (1/(2\pi))\sin(\pi\hat{\phi}/\epsilon_H)$ if $-\epsilon_H \le \hat{\phi} \le \epsilon_H$; $1$ if $\hat{\phi} > \epsilon_H$ [S8]. $\epsilon_H$ is the mean edge distance between sites, excluding the 5% longer edges [S8].

### NeuS Volume Rendering Transformation
Transforms SDF values into volumetric densities for alpha compositing, ensuring the CDF of volume rendering matches a scaled, flipped sigmoid shape [S9]. Transmittance $T(t)$ is related to SDF $S(t)$ by $1 - T(t) = \text{sigmoid}(s \cdot -SDF(t))$, where $s$ is a global scaling coefficient [S9]. The weighting function $w(t)$ has local maxima at SDF value 0 [S9]. Pitfall: Ties magnitude of volume density globally to sharpness via $s$, which is restrictive [S9].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of sample points along rays [S4, S5, unofficial].
- **Memory Complexity:** $O(N)$ for storing intermediate densities and features [S4, S5, unofficial].
- **Performance:** Efficient on GPU due to vectorized operations (e.g., `torch.where`) [S5, unofficial].
- **Strengths:** Differentiable, allows for gradient-based optimization of SDF parameters [S4, S5, unofficial]. Handles thin structures and open surfaces [S1].
- **Limitations:** Requires careful tuning of $\alpha$ and $\beta$ to avoid artifacts [S5, unofficial]. Sensitive to ray sampling density [G].
- **When to Use:** Neural implicit surface rendering, differentiable rendering, volume visualization of implicit surfaces [S4, S5, unofficial].
- **When Not to Use:** When explicit meshes are available and preferred, or when non-differentiable rendering is sufficient [G].

## Background
Originated in the context of neural implicit surface representations like VolSDF [Wang et al. 2021] and NeuS [Yariv et al. 2021] [S1]. VolSDF learns scaling and shrinking coefficients on distance-transformed volume densities [S9]. NeuS demands the CDF of volume rendering to match a sigmoid shape [S9]. The method is widely used in differentiable rendering and inverse rendering applications [S1].

## Sources
[S1] Moment Bounds are Differentiable: Efficiently Approximating Measures in Inverse Rendering, https://doi.org/10.1145/3730899, fetched 2026-10-27
[S2] ShengzhiWu/taichi-volume-renderer, https://github.com/ShengzhiWu/taichi-volume-renderer, fetched 2026-10-26 (unofficial)
[S3] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S4] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S6] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S7] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S8] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S9] Alpha Invariance: On Inverse Scaling Between Distance and Volume Density in Neural Radiance Fields, https://www.youtube.com/watch?v=YQhBEXC464w, fetched 2026-10-31
