# SDF to Density Conversion

Also called: Glass Ball IOR Distribution, SDF to Density/Feature Volume, SDFVolume, signed distance to density conversion

## Overview
SDF to Density Conversion maps a Signed Distance Function (SDF) value into a volumetric density field suitable for differentiable volume rendering [S3, unofficial][S4, unofficial]. It solves the problem of representing continuous 3D surfaces and thin structures without explicit mesh extraction by treating the zero-level set of the SDF as a volumetric boundary [S2]. Unlike standard volume rendering which takes density directly, this method derives density from the distance to the implicit surface, enabling the representation of open surfaces and varying thickness through an exponential falloff [S1, unofficial]. It sets itself apart from [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md) by providing the specific conversion function rather than the full rendering pipeline, and from [Volume SDF Rendering](1344-volume-sdf-rendering.md) by its specific parameterization of the density transition [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Signed distance value $d$ (or $s = -d$) at a sample point, ray segment length $\Delta t$ (or $\Delta$), and parameters $\alpha$ (alpha) and $\beta$ (beta).
*   **Output:** Volumetric density $\sigma$ (or $\rho$) and optionally a feature/color value.

**Data Structures:**
*   **Parameters:**
    *   $\alpha$ (alpha): Scalar multiplier for intermediate density, controlling overall opacity. Initialized from configuration or learned [S3, unofficial][S4, unofficial].
    *   $\beta$ (beta): Scalar controlling the exponential falloff rate. Smaller values yield sharper surfaces [S3, unofficial][S4, unofficial].
    *   `rainbow`: Boolean flag. If true, computes base color from position; otherwise, base color is 1.0 [S3, unofficial][S4, unofficial].
    *   `feature_param`: Learnable tensor for color features [S3, unofficial].

**Steps:**
1.  **Compute Signed Distance:** Evaluate the SDF at the sample point to get $d$. Define $s = -d$ [S4, unofficial].
2.  **Compute Intermediate Density ($\psi$ or $\rho$):** Apply a piecewise exponential function to $s$ [S3, unofficial][S4, unofficial].
    *   If $s \le 0$ (inside surface): $\psi = 0.5 \cdot \exp(s / \beta)$
    *   If $s > 0$ (outside surface): $\psi = 1 - 0.5 \cdot \exp(-s / \beta)$
    *   Scale by $\alpha$: $\rho = \alpha \cdot \psi$ [S3, unofficial][S4, unofficial].
3.  **Compute Final Density ($\sigma$):** Normalize by the ray segment length $\Delta t$ (or $\Delta$) to ensure density is independent of sampling resolution [S3, unofficial][S4, unofficial].
    *   $\sigma = -\ln(1 - \rho) / \Delta t$
    *   Note: This formula assumes $\rho < 1$. If $\rho$ approaches 1, numerical stability issues may arise [S3, unofficial].
4.  **Compute Feature/Color (Optional):**
    *   If `rainbow` is true: `base_color` = clamp($|sample\_points - center|$, 0.02, 0.98) [S3, unofficial][S4, unofficial].
    *   Else: `base_color` = 1.0 [S3, unofficial][S4, unofficial].
    *   `feature` = `base_color` * `feature_param` * $\rho$ (or $\rho$-derived value) [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def sdf_to_density(signed_distance: torch.Tensor, alpha: float, beta: float) -> torch.Tensor:
    """
    Converts signed distance to intermediate density.
    
    Args:
        signed_distance: Tensor of signed distance values d.
        alpha: Scaling factor for density.
        beta: Falloff rate parameter.
        
    Returns:
        Intermediate density rho.
    """
    # s = -d, where d is signed_distance
    s = -signed_distance
    
    # Piecewise exponential function
    # If s <= 0: psi = 0.5 * exp(s / beta)
    # If s > 0: psi = 1 - 0.5 * exp(-s / beta)
    psi = torch.where(
        s <= 0.0,
        0.5 * torch.exp(s / beta),
        1 - 0.5 * torch.exp(-s / beta)
    )
    
    # Scale by alpha
    rho = alpha * psi
    return rho

def compute_final_density(rho: torch.Tensor, delta_t: torch.Tensor) -> torch.Tensor:
    """
    Computes final volumetric density from intermediate density and segment length.
    
    Args:
        rho: Intermediate density (must be < 1).
        delta_t: Ray segment length.
        
    Returns:
        Final volumetric density sigma.
    """
    # Avoid log(0) or log(negative) by clamping rho slightly below 1
    rho_clamped = torch.clamp(rho, max=0.9999)
    
    # sigma = -ln(1 - rho) / delta_t
    sigma = -torch.log(1.0 - rho_clamped) / delta_t
    return sigma

def sdf_to_density_pipeline(signed_distance: torch.Tensor, delta_t: torch.Tensor, 
                            alpha: float, beta: float, rainbow: bool = False, 
                            center: torch.Tensor = None, feature_param: torch.Tensor = None,
                            sample_points: torch.Tensor = None) -> dict:
    """
    Full pipeline for SDF to Density Conversion.
    
    Args:
        signed_distance: Tensor of signed distance values d.
        delta_t: Ray segment length.
        alpha: Scaling factor.
        beta: Falloff rate.
        rainbow: Boolean flag for position-based coloring.
        center: Center point for rainbow mode.
        feature_param: Learnable feature tensor.
        sample_points: Sample point coordinates for rainbow mode.
        
    Returns:
        Dictionary with 'density' and 'feature'.
    """
    # Step 1 & 2: Compute intermediate density
    rho = sdf_to_density(signed_distance, alpha, beta)
    
    # Step 3: Compute final density
    # Handle edge case: if delta_t is 0 or very small, pad with large value to prevent division by zero
    # This is typically handled before calling this function, but as a guard:
    delta_t_safe = torch.where(delta_t > 1e-10, delta_t, torch.tensor(1e10, device=delta_t.device))
    sigma = compute_final_density(rho, delta_t_safe)
    
    # Step 4: Compute feature/color
    if rainbow and sample_points is not None and center is not None:
        base_color = torch.clamp(torch.abs(sample_points - center), min=0.02, max=0.98)
    else:
        base_color = torch.ones_like(sample_points) if sample_points is not None else torch.tensor(1.0)
        
    if feature_param is not None:
        feature = base_color * feature_param * rho
    else:
        feature = base_color * rho
        
    return {
        "density": sigma,
        "feature": feature
    }
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** The last segment's delta is often padded with a large value (e.g., $10^{10}$) to prevent division by zero when the ray exits the sampling volume [S3, unofficial].
*   **Numerical Stability:** The formula $\sigma = -\ln(1 - \rho) / \Delta t$ requires $\rho < 1$. If $\rho \ge 1$, the log is undefined. Clamp $\rho$ to a value slightly less than 1 [S3, unofficial].
*   **Parameter Sensitivity:** $\beta$ controls the sharpness of the transition. Smaller $\beta$ yields sharper surfaces but may require more sampling points to avoid aliasing [S4, unofficial].

## Subtopics

### [Volume SDF Rendering](1344-volume-sdf-rendering.md)
Volume SDF Rendering is a neural rendering technique that integrates color and density along rays sampled from an implicit Signed Distance Function (SDF). It solves the problem of rendering complex 3D geometry from multi-view images by converting the continuous SDF into a volumetric density field, allowing standard volume rendering to capture both surface appearance and thin structures. Unlike standard volume rendering which takes density directly, this method derives density from the distance to the implicit surface, enabling the representation of open surfaces and varying thickness through the Laplace distribution CDF. It differs from [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md) primarily in its specific parameterization of the density transition and its integration with analytic or neural SDF backends.

### [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md)
VolSDF Volume Rendering is a volume rendering technique that synthesizes images from a neural implicit surface representation by converting Signed Distance Function (SDF) values into volumetric density. It solves the problem of rendering continuous 3D surfaces without explicit mesh extraction by treating the zero-level set of the SDF as a volumetric boundary. It sets itself apart from its parent [SDF to Density Conversion](1282-sdf-to-density-conversion.md) by providing the complete rendering pipeline (sampling, transmittance, aggregation) rather than just the conversion function, and from its sibling [Volume SDF Rendering](1344-volume-sdf-rendering.md) by using a differentiable neural SDF with stratified sampling and alpha-compositing rather than discrete voxel rasterization.

### [Box with Smoothed/Eroded Edges](3417-box-with-smoothed-eroded-edges.md)
This method converts a signed distance field (SDF) value into a volumetric density for a box primitive, applying a sigmoid-based erosion at the boundaries and adding Perlin noise to roughen the surface. It differs from standard SDF-to-density conversions by explicitly expanding the box geometry to create a specific "eroded" falloff zone rather than simply smoothing the existing boundary, and it integrates procedural noise directly into the density calculation.

### [SDF Alpha Computation](3643-sdf-alpha-computation.md)
Computes the output alpha for a Signed Distance Field (SDF) given the Screen Space Ratio (SSR), Signed Distance (SD), Signed Distance Ratio (SDR), isoPerimeter (dilation/contraction), and softness. It differs from [SDF to Density Conversion](1282-sdf-to-density-conversion.md) by operating directly on distance metrics and screen-space scaling factors rather than converting to volumetric density, and from [Volume SDF Rendering](1344-volume-sdf-rendering.md) by being a specific shading/compositing step rather than a full rendering pipeline.

### Procedural Index of Refraction Distribution
Simulating mirages by creating a non-uniform distribution of the index of refraction in air, where IOR decreases with increasing temperature [S1, unofficial]. It creates a normal lapse layer ($index = 0.95 \cdot 2^z$) and an inversion layer ($index = 0.8^z$), combining them with `np.minimum`, and sets IOR to 1.0 for $z < 0$ [S1, unofficial].

### FBM-based Implicit Density Definition
Defines an implicit volumetric object where density $\rho$ is derived from an FBM field $f$. Density is zero when $f(\mathbf{p}) < v$, and remapped via function $h$ when $f(\mathbf{p}) \ge v$. The 0-isosurface of $\rho$ corresponds to the v-isosurface of $f$ [S5]. Formula: $\rho(\mathbf{p}) = h(f(\mathbf{p}))$ when $f(\mathbf{p}) \ge v$, else 0. Pitfall: If $h$ is not positive, the 0-isosurface of $\rho$ will not correspond to the v-isosurface of $f$ [S5].

### Density-based Volume Region
DensityRegion extends VolumeRegion by scaling absorption, scattering, extinction, and emission coefficients by a spatially varying density function $density(pt)$ [S6, unofficial]. Formulas: $\sigma_a(pt) = sig\_a \cdot density(pt)$, $\sigma_s(pt) = sig\_s \cdot density(pt)$, $\sigma_t(pt) = (\sigma_a + \sigma_s) \cdot density(pt)$, $L_{ve}(pt) = le \cdot density(pt)$ [S6, unofficial]. Phase function uses Henyey-Greenstein with parameter $g\_scatter$ [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sample points along the ray. The SDF evaluation is the dominant cost if the SDF is complex (e.g., neural network).
*   **Memory Complexity:** Linear with respect to the number of sample points.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with the number of samples per ray and the number of rays. Higher $\alpha$ or lower $\beta$ may require more samples to accurately resolve the density transition.
*   **Strengths:** Differentiable, allowing gradient-based optimization of SDF parameters. Efficiently represents thin structures and open surfaces.
*   **Limitations:** Sensitive to sampling density. If $\Delta t$ is too large, the density approximation may be inaccurate. Numerical stability issues if $\rho \ge 1$.
*   **When to Use:** When rendering neural implicit surfaces or when a smooth, differentiable representation of a surface is needed.
*   **When Not to Use:** When explicit meshes are available and preferred, or when real-time performance is critical and SDF evaluation is too expensive.

## Background
The method originates from neural volume rendering techniques, specifically those that use SDFs as implicit surface representations [S3, unofficial][S4, unofficial]. It was popularized by works such as VolSDF, which demonstrated the effectiveness of converting SDFs to density for differentiable rendering [S2]. The specific piecewise exponential function is a common choice for mapping signed distances to densities, ensuring smooth transitions and differentiability [S3, unofficial][S4, unofficial].

## Sources
[S1] ShengzhiWu/taichi-volume-renderer, https://github.com/ShengzhiWu/taichi-volume-renderer, fetched 2026-10-26 (unofficial)
[S2] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S3] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S4] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S5] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S6] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
