# SDF to Density Conversion

Also called: signed distance to density conversion

## Overview
SDF to Density Conversion transforms a Signed Distance Function (SDF) value $d(\mathbf{x})$ into a volumetric density $\sigma$ or opacity $\rho$ suitable for volume rendering or mesh generation. It solves the problem of representing continuous implicit surfaces as volumetric fields without explicit mesh extraction, allowing standard volume rendering techniques to capture surface appearance and thin structures [S1][S2, S4, S6, unofficial]. Unlike methods that take density directly, this approach derives density from the distance to the implicit surface, enabling the representation of open surfaces and varying thickness through specific parameterizations like the Laplace distribution CDF or sigmoid functions [S1][S6, unofficial]. It differs from [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md) primarily in its specific parameterization of the density transition and its integration with analytic or neural SDF backends [S3].

## How it works
**Inputs and Outputs:**
- Input: Signed distance value $d$ (or $s = -d$) at a point $\mathbf{x}$, and parameters $\alpha$ (scale) and $\beta$ (falloff/sharpness).
- Output: Volumetric density $\sigma$ or intermediate density $\rho$.

**Data Structures:**
- No specific complex data structures are required for the conversion function itself; it operates on scalar values or tensors of signed distances.

**Steps:**
1.  **Evaluate SDF:** Compute the signed distance $d$ from the point to the surface. By convention in some implementations, $s = -d$ is used, where $s > 0$ is inside the object and $s < 0$ is outside [S6, unofficial].
2.  **Compute Intermediate Density:** Apply a piecewise exponential or sigmoid function to map $d$ (or $s$) to an intermediate density $\rho$ or $\psi$.
    -   **Exponential Laplace/CDF approach:**
        $$ \psi = \begin{cases} 0.5 e^{s/\beta} & \text{if } s \le 0 \\ 1 - 0.5 e^{-s/\beta} & \text{if } s > 0 \end{cases} $$
        where $s = -d$ [S6, unofficial].
        Alternatively, using $d$ directly:
        $$ \rho = \begin{cases} 0.5 \cdot \exp(-d / \beta) & d > 0 \\ 1 - 0.5 \cdot \exp(d / \beta) & d \le 0 \end{cases} $$
        [S4, unofficial].
    -   **Scaling:** Multiply by $\alpha$ to get the final intermediate density: $\rho_{final} = \alpha \cdot \psi$ (or $\alpha \cdot \rho$) [S4, S6, unofficial].
3.  **Compute Rendering Density (Optional):** If ray marching with step size $\Delta t$ (or $\Delta$), convert the intermediate density to volumetric density $\sigma$ for alpha compositing:
    $$ \sigma = -\frac{\ln(1 - \rho_{final})}{\Delta t} $$
    [S4, S6, unofficial]. This ensures the transmittance matches the desired opacity over the segment.
4.  **Edge Cases:**
    -   If the ray exits the sampling volume, pad the last segment's delta with a large value (e.g., $10^{10}$) to prevent division by zero [S4, unofficial].
    -   For binary density (mesh generation), set density to 1.0 if $d < 0$ and 0.0 otherwise [S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def sdf_to_density(signed_distance, alpha, beta):
    """
    Converts signed distance to volumetric density using exponential falloff.
    
    Args:
        signed_distance: Tensor of signed distance values d.
        alpha: Scalar multiplier for overall opacity.
        beta: Scalar controlling the exponential falloff rate.
        
    Returns:
        density: Tensor of intermediate density values.
    """
    # s is negative signed distance: s > 0 inside, s < 0 outside
    s = -signed_distance
    
    # Piecewise exponential mapping
    # Inside (s <= 0): 0.5 * exp(s / beta)
    # Outside (s > 0): 1 - 0.5 * exp(-s / beta)
    psi = torch.where(
        s <= 0.0,
        0.5 * torch.exp(s / beta),
        1 - 0.5 * torch.exp(-s / beta),
    )
    
    # Scale by alpha
    density = alpha * psi
    return density

def compute_rendering_density(intermediate_density, delta_t):
    """
    Converts intermediate density to volumetric density for ray marching.
    
    Args:
        intermediate_density: Tensor of density values rho.
        delta_t: Tensor of segment lengths along the ray.
        
    Returns:
        sigma: Volumetric density.
    """
    # Avoid log(0) by clamping intermediate_density to [0, 1-eps]
    # Note: In practice, intermediate_density should be < 1.0
    rho = torch.clamp(intermediate_density, max=0.9999)
    sigma = -torch.log(1.0 - rho) / delta_t
    return sigma
```

**Parameters:**
-   $\alpha$ (alpha): Scalar multiplier for the intermediate density, controls overall opacity. Typically a learned or tuned constant [S4, S6, unofficial].
-   $\beta$ (beta): Scalar controlling the exponential falloff rate of density from the surface. Smaller values yield sharper surfaces [S4, S6, unofficial].
-   $s$: Global scaling coefficient in sigmoid-based formulations (e.g., NeuS), acting as the sharpness parameter [S10].

## Subtopics

### [Volume SDF Rendering](1344-volume-sdf-rendering.md)
Volume SDF Rendering is a neural rendering technique that integrates color and density along rays sampled from an implicit Signed Distance Function (SDF). It solves the problem of rendering complex 3D geometry from multi-view images by converting the continuous SDF into a volumetric density field, allowing standard volume rendering to capture both surface appearance and thin structures. Unlike standard volume rendering which takes density directly, this method derives density from the distance to the implicit surface, enabling the representation of open surfaces and varying thickness through the Laplace distribution CDF. It differs from [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md) primarily in its specific parameterization of the density transition and its integration with analytic or neural SDF backends.

### [Procedural Index of Refraction Distribution](1810-procedural-index-of-refraction-distribution.md)
Simulating mirages by creating a non-uniform distribution of the index of refraction in air, where IOR decreases with increasing temperature.

### [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md)
VolSDF Volume Rendering synthesizes images from a neural implicit surface representation by converting Signed Distance Function (SDF) values into volumetric density [G]. It solves the problem of rendering continuous 3D surfaces without explicit mesh extraction by treating the zero-level set of the SDF as a volumetric boundary [G]. It sets itself apart from its parent [SDF to Density Conversion](1282-sdf-to-density-conversion.md) by providing the complete rendering pipeline (sampling, transmittance, aggregation) rather than just the conversion function, and from its sibling [Volume SDF Rendering](1344-volume-sdf-rendering.md) by using a differentiable neural SDF with stratified sampling and alpha-compositing rather than discrete voxel rasterization.

### [Box with Smoothed/Eroded Edges](3417-box-with-smoothed-eroded-edges.md)
This method converts a signed distance field (SDF) value into a volumetric density for a box primitive, applying a sigmoid-based erosion at the boundaries to create smooth, anti-aliased edges and adding Perlin noise to roughen the surface. It differs from standard SDF-to-density conversions by explicitly expanding the box geometry to define an erosion zone and combining axis-aligned sigmoid falloffs with spatial noise, rather than relying solely on a global distance threshold or simple alpha blending. It builds on [SDF to Density Conversion](1282-sdf-to-density-conversion.md).

### [SDF Alpha Computation](3643-sdf-alpha-computation.md)
Computes the output alpha for a Signed Distance Field (SDF) given the Screen Space Ratio (SSR), Signed Distance (SD), Signed Distance Ratio (SDR), isoPerimeter (dilation/contraction), and softness. It converts a signed distance value into a density value using alpha and beta parameters, building on [SDF to Density Conversion](1282-sdf-to-density-conversion.md).

### [Gray Transform (G_o, G_s)](10492-gray-transform-g-o-g-s.md)
The Gray Transform converts a discrete object description or a continuous surface description into a volumetric scene (density field) suitable for physical modeling, manufacturing, or volume rendering. It distinguishes itself from sibling methods by explicitly defining density values based on interior/exterior status or signed distance to a surface, rather than relying on implicit volume generation or alpha-compositing parameters directly.

### FBM-based Implicit Density Definition
Defines an implicit volumetric object where density $\rho$ is derived from an FBM field $f$. Density is zero when $f(\mathbf{p}) < v$, and remapped via function $h$ when $f(\mathbf{p}) \ge v$. The 0-isosurface of $\rho$ corresponds to the v-isosurface of $f$ [S7].
Formula:
$$ \rho(\mathbf{p}) = \begin{cases} h(f(\mathbf{p})) & \text{when } f(\mathbf{p}) \geq v \\ 0 & \text{otherwise} \end{cases} $$
where $f(\mathbf{p}) = B_n(\mathbf{p})$ is the FBM field, $v$ is the threshold, and $h$ is a positive remapping function [S7].
Pitfall: If $h$ is not a positive function, the 0-isosurface of $\rho$ will not correspond to the v-isosurface of $f$ [S7].

### Density-based Volume Region
DensityRegion extends VolumeRegion by scaling absorption, scattering, extinction, and emission coefficients by a spatially varying density function density(pt) [S8, unofficial].
Formulas:
- $\sigma_a(pt) = \text{sig\_a} \cdot \text{density}(pt)$
- $\sigma_s(pt) = \text{sig\_s} \cdot \text{density}(pt)$
- $\sigma_t(pt) = (\text{sig\_a} + \text{sig\_s}) \cdot \text{density}(pt)$
- $L_{ve}(pt) = \text{le} \cdot \text{density}(pt)$
- $\text{phase}(pt, w, w') = \text{PhaseHG}(w, w', g_{\text{scatter}})$
Parameters: $g_{\text{scatter}}$ is the Henyey-Greenstein anisotropy parameter [S8, unofficial].

### Smeared Heaviside Function for Mean Curvature
A smeared-out Heaviside function $H$ used to compute mean curvature for regularization, defined piecewise based on a threshold $\epsilon_H$ [S9].
Formula:
$$ H(\hat{\phi}) = \begin{cases} 0 & \hat{\phi} < -\epsilon_H \\ \frac{1}{2} + \frac{\hat{\phi}}{2\epsilon_H} + \frac{1}{2\pi} \sin\left(\frac{\pi\hat{\phi}}{\epsilon_H}\right) & -\epsilon_H \leq \hat{\phi} \leq \epsilon_H \\ 1 & \hat{\phi} > \epsilon_H \end{cases} $$
where $\epsilon_H$ is the mean edge distance between sites excluding the 5% longer edges [S9].

### NeuS Volume Rendering Transformation
A method that transforms Signed Distance Function (SDF) values into volumetric densities for alpha compositing, ensuring the CDF of volume rendering matches a scaled, flipped sigmoid shape [S10].
Formula:
The transmittance $T(t)$ is related to the SDF $S(t)$ by $1 - T(t) = \text{sigmoid}(s \cdot -SDF(t))$, where $s$ is a global scaling coefficient. The weighting function $w(t)$ (derivative of CDF) has local maxima at SDF value 0 [S10].
Comparison: NeuS ties the magnitude of volume density globally to its sharpness via the sigmoid parameter $s$, which is an alpha-invariant formulation by default but restrictive [S10].

### Squared Sigmoid (Fsq)
A variant of the sigmoid function used in GenDR where the absolute part of the distance is squared before applying the base sigmoid, interpreted as applying a transformed CDF Fsq [S11].
Steps:
1. Multiply the absolute distance by the signed distance: $|d(p, t)| \cdot d(p, t)$.
2. Divide by temperature: $(|d(p, t)| \cdot d(p, t))/\tau$.
3. Apply the base sigmoid function $F$ to the result.
Formula:
$$ Fsq(d(p, t)/\tau) := F(|d(p, t)| \cdot d(p, t)/\tau) $$
Comparison: Fsq is generally worse than F for distributions with exponential decay, but comparable or better for those with linear convergence (Cauchy, Reciprocal Sigmoid) [S11].

## Operating characteristics
-   **Time Complexity:** O(1) per sample point for the conversion function. The overall rendering cost depends on the number of samples per ray and the number of rays.
-   **Memory Complexity:** O(1) for the conversion function. The SDF evaluation may require O(N) memory if storing a grid, or O(1) if analytic.
-   **Performance:** The exponential/sigmoid conversion is computationally cheap. The choice of $\beta$ affects the number of samples needed for accurate rendering; smaller $\beta$ (sharper surface) may require more samples near the surface to avoid aliasing [S6, unofficial].
-   **Strengths:** Enables differentiable rendering of implicit surfaces; allows for smooth transitions and anti-aliasing; supports thin structures and open surfaces [S1][S6, unofficial].
-   **Limitations:** The choice of parameters ($\alpha, \beta$) is critical and often requires tuning or learning; the assumption of alpha-invariance in some formulations (like NeuS) can be restrictive [S10].
-   **When to use:** When rendering neural implicit surfaces (NeRF, NeuS, VolSDF) or procedural volumes where the geometry is defined by an SDF [S1][S4, S6, unofficial].
-   **When not to use:** When explicit meshes are available and preferred for performance or compatibility with rasterization pipelines [S5, unofficial].

## Background
The method originates from the need to render implicit surfaces using volume rendering techniques, particularly in the context of Neural Radiance Fields (NeRF) and differentiable rendering. Key papers include VolSDF [Yariv et al. 2021] and NeuS [Wang et al. 2021], which introduced the transformation of SDFs to densities using sigmoid and exponential functions, respectively [S1, S10]. The technique has since been adopted in various forms for procedural volume rendering [S7] and differentiable mesh generation [S9].

## Sources
[S1] Moment Bounds are Differentiable: Efficiently Approximating Measures in Inverse Rendering, https://doi.org/10.1145/3730899, fetched 2026-10-27
[S2] ShengzhiWu/taichi-volume-renderer, https://github.com/ShengzhiWu/taichi-volume-renderer, fetched 2026-10-26 (unofficial)
[S3] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S4] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
[S6] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S7] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S8] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S9] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S10] Alpha Invariance: On Inverse Scaling Between Distance and Volume Density in Neural Radiance Fields, https://www.youtube.com/watch?v=YQhBEXC464w, fetched 2026-10-31
[S11] GenDR: A Generalized Differentiable Renderer, https://doi.org/10.1109/cvpr52688.2022.00397, fetched 2026-10-26
