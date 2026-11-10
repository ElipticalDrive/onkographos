# Relightable 3DGS

Also called: Handling Challenging Objects (Reflections/Transparency), Relightable 3D GS, Specular 3D GS

## Overview
Relightable 3DGS extends standard 3D Gaussian Splatting by embedding physically-based material properties into each Gaussian to enable deferred shading [S1, unofficial]. It solves the inability of standard rasterization to accurately represent specular reflections, glossy materials, and global illumination by decomposing outgoing radiance into a BRDF integration [S1, unofficial]. Unlike standard 3DGS which relies on view-dependent spherical harmonics, this method supports relighting under novel illumination conditions while preserving real-time performance [S1, unofficial].

## How it works
**Inputs:**
*   A set of 3D Gaussians $\{G_i\}$, where each Gaussian $i$ contains position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and newly introduced material parameters: diffuse albedo $f_d$, specular BRDF parameters (e.g., roughness $r$), and normal $\mathbf{n}_i$ [S1, unofficial][S2].
*   A lighting environment defined by incident radiance $L_i(\omega_i)$ or point light sources [S1, unofficial].
*   Camera view direction $\omega_0$ [S1, unofficial].

**Outputs:**
*   The final pixel color $c'$ for each view [S1, unofficial].

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian stores standard 3DGS attributes plus:
    *   Normal vector $\mathbf{n}_i$ (unit length) [S2].
    *   Roughness parameter $r$ for the microfacet model [S1, unofficial].
    *   Diffuse albedo $f_d$ [S1, unofficial].
    *   (Optional) Mirror attributes and plane-mirror imaging data for reflective surfaces [S2].

**Steps:**
1.  **Rasterization:** Project Gaussians to screen space and sort by depth, similar to standard 3DGS MVP [S1, unofficial].
2.  **BRDF Evaluation:** For each Gaussian contributing to a pixel, compute the specular BRDF $f_s$ using the Cook-Torrance microfacet model [S1, unofficial]:
    $$f_s(\omega_0, \omega_i) = \frac{D(h; r) \cdot F(\omega_0, h) \cdot G(\omega_i, \omega_0, h; r)}{(n \cdot \omega_i) \cdot (n \cdot \omega_0)}$$
    Where:
    *   $\omega_0$ is the view direction [S1, unofficial].
    *   $\omega_i$ is the incident lighting direction [S1, unofficial].
    *   $h = (\omega_i + \omega_0)/2$ is the half vector [S1, unofficial].
    *   $D$ is the normal distribution function [S1, unofficial].
    *   $F$ is the Fresnel term [S1, unofficial].
    *   $G$ is the geometry term [S1, unofficial].
    *   $n$ is the surface normal [S1, unofficial].
    *   $r$ is the roughness [S1, unofficial].
3.  **Radiance Integration:** Compute the PBR color $c'(\omega_0)$ by integrating the sum of diffuse and specular components over the incident lighting directions [S1, unofficial]:
    $$c'(\omega_0) = \sum_{i=0}^{N_s} (f_d + f_s(\omega_0, \omega_i)) L_i(\omega_i) (\omega_i \cdot n) \Delta \omega_i$$
    Where:
    *   $f_d$ is the diffuse component [S1, unofficial].
    *   $L_i(\omega_i)$ is the incident radiance [S1, unofficial].
    *   $\Delta \omega_i$ is the solid angle associated with the sampled incident direction [S1, unofficial].
4.  **Shadow Casting (Point Lights):** For direct illumination from point sources, trace secondary rays from each Gaussian to the light source and evaluate a shadow map to determine occlusion [S1, unofficial].
5.  **Mirror Handling (Optional):** For highly reflective surfaces, use mirror attributes and plane-mirror imaging to reconstruct mirror geometry and reflections [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_relightable_color(gaussian, view_dir, light_sources, shadow_map):
    """
    Computes the color for a single Gaussian contribution in Relightable 3DGS.
    gaussian: Object containing p, n, alpha, f_d, roughness r.
    view_dir: Vector omega_0.
    light_sources: List of light sources with position and radiance.
    shadow_map: Function to check occlusion.
    """
    total_color = Vector3(0, 0, 0)
    
    # 1. Compute Diffuse Component
    # f_d is stored in gaussian
    diffuse_term = gaussian.f_d
    
    # 2. Compute Specular Component using Cook-Torrance BRDF
    # Sum over light sources (discrete approximation of integral)
    specular_sum = Vector3(0, 0, 0)
    
    for light in light_sources:
        # Incident direction from surface to light
        light_dir = normalize(light.position - gaussian.p)
        
        # Half vector h = (omega_i + omega_0) / 2
        half_vec = normalize(light_dir + view_dir)
        
        # Microfacet BRDF components
        # D: Normal Distribution Function (e.g., GGX)
        D = compute_NDF(half_vec, gaussian.n, gaussian.roughness)
        
        # F: Fresnel Term (e.g., Schlick's approximation)
        F = compute_Fresnel(view_dir, half_vec, gaussian.f_d) # Simplified, usually uses base reflectivity
        
        # G: Geometry Term (e.g., Schlick-GGX)
        G = compute_geometry(gaussian.n, light_dir, view_dir, gaussian.roughness)
        
        # Denominator: (n . omega_i) * (n . omega_0)
        # Note: Standard BRDF formulation divides by these. 
        # However, the integral formula [S1, unofficial] includes (omega_i . n) in the sum.
        # The formula [S1, unofficial] is: c' = sum( (f_d + f_s) * L_i * (omega_i . n) * delta_omega )
        # And f_s is defined with the denominator.
        
        denom = max(dot(gaussian.n, light_dir), 1e-6) * max(dot(gaussian.n, view_dir), 1e-6)
        
        if denom > 1e-6:
            fs = (D * F * G) / denom
        else:
            fs = Vector3(0, 0, 0)
            
        # Shadow check
        is_shadowed = shadow_map.check(gaussian.p, light.position)
        shadow_factor = 0.0 if is_shadowed else 1.0
        
        # Radiance contribution
        # L_i is light.intensity / distance^2 typically, or pre-filtered
        Li = light.intensity * shadow_factor
        
        # Contribution: (f_d + f_s) * L_i * (n . omega_i)
        # Note: The formula [S1, unofficial] has (omega_i . n) inside the sum.
        # f_s already contains the 1/(n.omega_i) term in its definition [S1, unofficial].
        # So: (f_d + fs) * Li * (n.omega_i)
        
        contribution = (diffuse_term + fs) * Li * max(dot(gaussian.n, light_dir), 0.0)
        specular_sum += contribution

    total_color = diffuse_term * light_sources[0].intensity * max(dot(gaussian.n, light_sources[0].dir), 0.0) + specular_sum # Simplified for single light demo
    
    # Apply opacity and alpha blending would happen in the rasterizer loop
    return total_color
```

**Parameters:**
*   Roughness $r$: Controls the spread of the specular lobe [S1, unofficial].
*   Diffuse albedo $f_d$: Base color of the material [S1, unofficial].
*   Number of light samples $N_s$: Determines the accuracy of the integration [S1, unofficial].

**Edge Cases:**
*   **Degenerate Normals:** If $\mathbf{n} \cdot \omega_i \approx 0$, the BRDF denominator approaches zero. Guard against division by zero [S1, unofficial].
*   **Mirror Geometry:** Standard rasterization fails for mirrors; explicit plane-mirror imaging is required [S2].

## Subtopics

### GaussianShader
Uses a neural shader to predict specular and diffuse components directly from view direction and learnable material features, simplifying the BRDF [S1, unofficial].

### GS-IR
Performs inverse rendering by jointly optimizing Gaussian geometry and an environment light probe, backpropagating gradients through the renderer [S1, unofficial].

### Anisotropic 3DGS
Uses anisotropic view-dependent appearance fields to capture specular and anisotropic components more faithfully [S2].

### Mirror 3DGS
Uses mirror attributes and plane-mirror imaging to reconstruct mirror geometry and reflections [S2].

## Operating characteristics
*   **Performance:** Preserves real-time performance during view synthesis [S1, unofficial].
*   **Accuracy:** Significantly improves fidelity for reflective, transparent, highly glossy, or anisotropic surfaces compared to standard 3DGS [S2].
*   **Limitations:** Robust reconstruction of reflective and transparent objects in general scenes remains challenging [S2]. Standard 3DGS approximations still contribute to artifacts like aliasing and blur, limiting illumination estimation accuracy [S2].
*   **Use Case:** Best for scenes requiring dynamic relighting or accurate specular reflections; not necessary for diffuse-only scenes where standard 3DGS suffices.

## Background
Relightable 3DGS was introduced to address the limitations of view-dependent spherical harmonics in standard 3DGS, which fail to represent specular reflections, glossy materials, and global illumination [S1, unofficial]. It represents a paradigm shift by embedding physically-based material properties into each Gaussian for deferred shading [S1, unofficial]. The method builds on the Cook-Torrance microfacet BRDF model [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
