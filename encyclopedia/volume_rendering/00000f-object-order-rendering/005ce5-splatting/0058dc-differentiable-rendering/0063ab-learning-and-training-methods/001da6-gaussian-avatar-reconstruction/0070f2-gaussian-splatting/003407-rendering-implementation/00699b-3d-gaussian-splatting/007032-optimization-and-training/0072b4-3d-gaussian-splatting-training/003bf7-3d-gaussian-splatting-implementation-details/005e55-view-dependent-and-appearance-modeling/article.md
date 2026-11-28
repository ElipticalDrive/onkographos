# View-Dependent and Appearance Modeling


## Overview
This family extends 3D Gaussian Splatting to model complex, view-dependent lighting effects such as specular highlights and anisotropic reflections, which standard isotropic spherical Gaussians cannot capture accurately [G]. It sets itself apart from standard 3DGS by replacing or augmenting the static spherical harmonic color representation with physically based rendering (PBR) models, directional anisotropic functions, or neural networks predicting view-dependent attributes [S1, unofficial][S2, unofficial][S3].

## Shared principles
All methods in this family share the goal of decomposing appearance into geometric and directional components to support view-dependent shading [G].

1. **Primitive Representation:** Gaussians are augmented with parameters beyond position, covariance, and opacity, such as material properties (roughness, metallic, albedo) or directional spread parameters [S1, unofficial][S2, unofficial].
2. **Directional Evaluation:** The contribution of a Gaussian to a pixel is modulated by the viewing direction relative to the primitive's orientation, achieved through Anisotropic Spherical Gaussians (ASG), neural networks, or refined spherical harmonics [S1, unofficial][S2, unofficial][S3].
3. **Physical or Approximate Shading:** Colors are computed using PBR models (diffuse + specular), directional basis functions, or neural residuals, rather than simple SH evaluation [S1, unofficial][S2, unofficial][S3].
4. **Differentiable Optimization:** All parameters are optimized via gradient descent using a photometric loss, requiring differentiable rendering of the directional components [S1, unofficial][S2, unofficial].

The core rendering mechanism for the PBR/ASG branch is implemented as follows:

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

## Subtopics

### [Anisotropic Reflective Spherical 3D Gaussian Splatting](17311-anisotropic-reflective-spherical-3d-gaussian-splatting.md)
ARS-GS embeds Anisotropic Spherical Gaussians (ASG) within a Physically Based Rendering (PBR) pipeline, aligning them with GGX microfacet models to model environmental illumination and enable accurate mesh reconstruction via light-transport simulation. Unlike Spec-Gaussian, which uses ASG primarily for appearance fitting, ARS-GS strictly embeds ASG within a PBR pipeline by aligning it with GGX microfacet models. It solves the problem of vanishing gradients in highly reflective regions by introducing a shortcut connection mechanism that links spatial coordinates directly to ASG color attributes, enabling rapid updates of specular reflections when primitives undergo positional changes. See [Anisotropic Reflective Spherical 3D Gaussian Splatting](17311-anisotropic-reflective-spherical-3d-gaussian-splatting.md).

### [Spec-Gaussian](17362-spec-gaussian.md)
Spec-Gaussian is a method that utilizes anisotropic spherical Gaussians to approximate 3D scene appearance, specifically designed to handle complex shading such as specular and anisotropic components. It sets itself apart from standard 3D Gaussian Splatting by replacing isotropic spherical Gaussians with anisotropic variants to better model directional reflectance properties. See [Spec-Gaussian](17362-spec-gaussian.md).

### View-Dependent Gaussian Splatting (VDGS)
VDGS produces more faithful view-dependent effects by modeling 3DGS to represent 3D shapes and predicting attributes like view-dependent color and opacity with a NeRF-like neural network instead of spherical harmonic coefficients [S1, unofficial]. On the MipNeRF 360 dataset, it achieves a PSNR of 27.66, SSIM of 0.809, and LPIPS of 0.223 [S1, unofficial].

### Neural Illumination Enhancement
NIE is a neural module applied only to non-Lambertian regions to model view-dependent highlights and handle non-Lambertian effects that vanilla 3DGS struggles with [S2, unofficial]. It complements the explicit 3DGS representation by predicting a view-dependent color residual using a lightweight Multilayer Perceptron (MLP) [S2, unofficial]. The MLP takes spatial position, reflection direction, and material-related features (view direction, surface normals, Fresnel term, roughness) as input. Spatial positions are encoded using HashGrid encoding, reflection directions are encoded using Spherical Harmonics (SH), and other features are used directly as identity features. The final rendered image is computed as $I = I_g + I_n$, where $I_g$ is the explicit Gaussian image and $I_n$ is the neural residual [S2, unofficial]. Unlike existing methods that struggle with non-Lambertian effects or cache only partial global illumination, NIE enables modeling of view-dependent highlights alongside global illumination caching [S2, unofficial].

### Revised Spherical Harmonics
This SH refinement introduces a higher-order weighting scheme to dynamically adjust SH coefficients, emphasizing higher-frequency terms in detailed regions and suppressing noise in smooth areas [S3]. The SH-to-RGB calculation is transferred to a specialized SH module rather than computing SH projection directly inside the rasterizer to cut 10-15% of unnecessary computation [S3]. The color is computed as $C(x; d) = \sum_{l=0}^{L} \sum_{m=-l}^{l} w_{l,m} c_{l,m} Y_{l,m}(d)$, where $w_{l,m}$ is a learned weight, $c_{l,m}$ are SH coefficients, and $Y_{l,m}$ are basis functions [S3]. This optimization improves PSNR by approximately 5% in complex scenes while maintaining real-time rendering efficiency [S3].

## Background
Standard 3D Gaussian Splatting uses spherical harmonics to encode view-dependent color, which struggles with high-frequency specularities and anisotropic reflections [G]. Recent advances introduced anisotropic representations and PBR integration to address these limitations [G]. VDGS employs a neural network to predict view-dependent attributes, offering an alternative to explicit directional basis functions [S1, unofficial]. NIE applies a neural module to non-Lambertian regions to model view-dependent highlights [S2, unofficial]. Revised Spherical Harmonics dynamically adjust SH coefficients using learned weights to improve color accuracy [S3]. These methods are used in high-fidelity 3D reconstruction, digital twins, and virtual production where realistic lighting interactions are critical [G].

## Sources
[S1] Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S2] Mesh2GS: White-Box 3DGS Construction via Plenoptic Sampling, https://doi.org/10.48550/arXiv.2606.21898, fetched 2026-10-30 (unofficial)
[S3] Optimization of 3D Gaussian Splatting for Accurate Image Reconstruction, https://doi.org/10.59232/dst-v4i2p101, fetched 2026-10-26
