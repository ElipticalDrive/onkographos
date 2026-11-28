# Spec-Gaussian


## Overview
Spec-Gaussian is a method for modeling complex view-dependent appearance, specifically specular and anisotropic reflections, within 3D Gaussian Splatting by replacing standard spherical harmonics with Anisotropic Spherical Gaussians (ASG) [S1][S2, unofficial]. It solves the limitation of standard 3DGS, which struggles to represent high-frequency directional lighting and sharp specular highlights, by approximating the scene's appearance distribution using ASG basis functions [S1][S2, unofficial]. Unlike standard 3DGS which uses isotropic SH for color, Spec-Gaussian decouples the geometric spread from the directional shading, allowing for physically plausible PBR-like shading [S1][S2, unofficial].

## How it works
**Inputs:** 3D Gaussian primitives (position, covariance, opacity), material properties (roughness, metallic, albedo), normal vectors, and environment lighting represented as SH coefficients.
**Outputs:** Pixel color and opacity.

**Data Structures:**
Each Gaussian stores:
- `center`: 3D position.
- `covariance`: 3x3 matrix (or 6-vector lower triangle) defining spatial spread.
- `opacity`: scalar.
- `roughness`: scalar.
- `metallic`: scalar.
- `albedo`: 3D vector.
- `normal`: 3D vector (often derived from covariance or optimized).
- `light_sh_coeffs`: SH coefficients for the environment map.

**Steps:**
1. **Ray-Gaussian Intersection:** Compute the point of closest approach $p_{closest}$ between the view ray and the Gaussian center. Calculate the maximum Gaussian value $V_{max} = \exp(-0.5 \Delta^T \Sigma^{-1} \Delta)$ where $\Delta = p_{closest} - center$ [G].
2. **View Direction:** Compute the outgoing view direction $\omega_o = -ray\_dir$.
3. **Reflection Direction:** Compute the perfect reflection direction $r = 2(n \cdot \omega_o)n - \omega_o$, where $n$ is the surface normal [G].
4. **ASG Bandwidths:** Compute ASG bandwidth parameters $\lambda$ and $\mu$ from roughness $\rho$:
   $$ \lambda = \mu = \frac{1 - \rho^2}{\rho^2} $$
   Clamp $\rho$ to a minimum of $10^{-6}$ to avoid division by zero [G].
5. **Specular Distribution:** Evaluate the ASG approximation of the GGX distribution $D(r)$ using the frame vectors $(x, y, z)$ of the Gaussian and the reflection direction $r$:
   $$ D(r) = \frac{\lambda \mu \pi}{\lambda (r \cdot x)^2 + \mu (r \cdot y)^2 + 2} $$
   [G]
6. **Specular Color:** Compute the specular contribution using a split-sum approximation. Evaluate the lighting integral using SH evaluation at the reflection direction $r$:
   $$ L_{spec} = SH(r) $$
   Compute the BRDF integral using the Fresnel-Schlick approximation and the ASG distribution. The final specular color is $C_{spec} = L_{spec} \cdot F \cdot D \cdot G$ [G]. Note: The provided pseudocode simplifies this to a base color multiplication.
7. **Diffuse Color:** Compute the diffuse contribution using the cosine lobe kernel. Evaluate the irradiance using SH at the normal $n$:
   $$ L_{diff} = SH(n) \cdot \pi $$
   $$ C_{diff} = albedo \cdot (1 - metallic) \cdot L_{diff} $$
8. **Composite:** The final color for the Gaussian is $C = (C_{diff} + C_{spec}) \cdot V_{max}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def evaluate_gaussian_1d(gaussian, ray_origin, ray_dir):
    """Computes the 1D Gaussian value at the point of closest approach along the ray."""
    inv_cov = np.linalg.inv(gaussian['covariance'])
    t_star = np.dot(gaussian['center'] - ray_origin, ray_dir)
    p_closest = ray_origin + t_star * ray_dir
    diff_closest = p_closest - gaussian['center']
    exponent = -0.5 * np.dot(diff_closest, np.dot(inv_cov, diff_closest))
    max_value = np.exp(exponent)
    return max_value, t_star

def compute_asg_bandwidths(roughness):
    """Computes lambda and mu from roughness rho."""
    rho = max(roughness, 1e-6)
    val = (1 - rho**2) / rho**2
    return val, val

def compute_d_asg(h, x, y, z, lam, mu):
    """Computes the ASG approximation of the GGX distribution."""
    hx = np.dot(h, x)
    hy = np.dot(h, y)
    denom = lam * hx**2 + mu * hy**2 + 2
    return (lam * mu * np.pi) / denom

def compute_g_asg(n, omega, h, x, y, lam, mu):
    """Computes the ASG geometry term for a single direction."""
    n_omega = np.dot(n, omega)
    hx = np.dot(h, x)
    hy = np.dot(h, y)
    denom = n_omega + lam * hx**2 + mu * hy**2 + n_omega**2
    return (2 * n_omega) / denom

def sh_evaluate(sh_coeffs, direction):
    """Standard SH basis evaluation at direction [G]."""
    return np.sum(sh_coeffs * np.array([1, direction[1], direction[2], direction[0], direction[1]*direction[2], direction[0]*direction[2], direction[0]*direction[1], direction[2]**2 - direction[1]**2, direction[0]**2 - 3*direction[1]**2 + 3*direction[2]**2]))

def compute_specular_color(light_sh_coeffs, roughness, metallic, albedo, normal, view_dir, reflection_dir):
    """Computes specular color using split-sum approximation."""
    lam, mu = compute_asg_bandwidths(roughness)
    lighting_integral = sh_evaluate(light_sh_coeffs, reflection_dir)
    f1 = 1.0
    f2 = 0.0
    base_color = (1 - metallic) * 0.04 + metallic * albedo
    brdf_integral = base_color * f1 + f2
    return lighting_integral * brdf_integral

def compute_diffuse_color(light_sh_coeffs, albedo, metallic, normal):
    """Computes diffuse color using cosine lobe kernel."""
    diffuse_irradiance = sh_evaluate(light_sh_coeffs, normal) * np.pi
    return albedo * (1 - metallic) * diffuse_irradiance

def render_gaussian(gaussian, ray_origin, ray_dir, light_sh_coeffs, roughness, metallic, albedo, normal, view_dir):
    """Renders a single Gaussian primitive for a ray."""
    val, t_star = evaluate_gaussian_1d(gaussian, ray_origin, ray_dir)
    if val < 1e-6:
        return 0.0, 0.0
    omega_o = -ray_dir
    t = 2 * np.dot(normal, omega_o) * normal - omega_o
    d_asg = compute_d_asg(t, gaussian['frame_x'], gaussian['frame_y'], gaussian['frame_z'], *compute_asg_bandwidths(roughness))
    specular = compute_specular_color(light_sh_coeffs, roughness, metallic, albedo, normal, omega_o, t)
    diffuse = compute_diffuse_color(light_sh_coeffs, albedo, metallic, normal)
    c_j = diffuse + specular
    return c_j * val, val
```

**Edge Cases:**
- **Roughness Zero:** If roughness is 0, $\lambda, \mu$ become infinite. Clamp roughness to $10^{-6}$ [G].
- **Degenerate Covariance:** If covariance is singular, inverse fails. Add small epsilon to diagonal [G].

## Subtopics

### [Deferred Reflection in 3D Gaussian Splatting](10039-deferred-reflection-in-3d-gaussian-splatting.md)
A two-pass technique that computes specular reflections in screen space to solve high-frequency radiance fitting issues and normal estimation bottlenecks inherent in forward shading methods like Spec-Gaussian.

## Operating characteristics
- **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of splatted Gaussians. ASG evaluation is slightly more expensive than standard SH but cheaper than full ray tracing.
- **Memory Complexity:** $O(N)$ for storing Gaussian parameters including material properties and ASG frame vectors.
- **Strengths:** Accurately represents sharp specular highlights and anisotropic reflections which standard 3DGS blurs.
- **Limitations:** Sensitive to normal estimation errors; requires careful initialization of material properties.

## Background
Spec-Gaussian was introduced to address the inability of standard 3D Gaussian Splatting to model complex shading like specular and anisotropic components [S1][S2, unofficial]. It utilizes Anisotropic Spherical Gaussians to approximate 3D scene appearance, building on the differentiable rendering framework of 3DGS [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
