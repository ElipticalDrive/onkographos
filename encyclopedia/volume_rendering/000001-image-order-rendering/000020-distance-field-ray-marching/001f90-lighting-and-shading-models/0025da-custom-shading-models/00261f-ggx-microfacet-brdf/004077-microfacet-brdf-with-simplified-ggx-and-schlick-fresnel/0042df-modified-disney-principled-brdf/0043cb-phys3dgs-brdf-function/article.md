# Phys3DGS BRDF Function

Also called: Phys3DGS BRDF

## Overview
A physically-based Bidirectional Reflectance Distribution Function (BRDF) used in 3D Gaussian Splatting for inverse rendering, consisting of a diffuse component and a specular component based on the GGX microfacet model with a correlated Smith masking-shadowing function [S1, unofficial]. It differs from its parent, the Modified Disney Principled BRDF, by employing a specific roughness remapping ($\alpha = r^2$) and a correlated Smith geometry function rather than the separable Schlick-GGX approximation, aiming for higher physical accuracy in the context of Gaussian splatting optimization [S1, unofficial].

## How it works
**Inputs:**
- $\omega_i$: Incident light direction (view direction in rendering context).
- $\omega_o$: Outgoing view direction.
- $n$: Surface normal.
- $a$: Base color (diffuse albedo).
- $m$: Metallic value (scalar in $[0, 1]$).
- $r$: Roughness parameter (scalar in $[0, 1]$).

**Outputs:**
- $f(\omega_o, \omega_i)$: The reflected radiance factor (BRDF value).

**Data Structures:**
No complex data structures are required; the function operates on scalar and vector quantities.

**Steps:**
1.  **Compute Roughness Remapping:**
    Calculate the squared roughness parameter $\alpha$:
    $$ \alpha = r^2 $$
    [S1, unofficial]

2.  **Compute Half Vector:**
    Calculate the normalized half vector $h$:
    $$ h = \frac{\omega_i + \omega_o}{\|\omega_i + \omega_o\|} $$
    [S1, unofficial]

3.  **Compute Fresnel Term ($F$):**
    First, calculate the Fresnel reflectance at normal incidence $F_0$:
    $$ F_0 = 0.04(1 - m) + a m $$
    [S1, unofficial]
    Then, compute the Fresnel reflection using the Schlick approximation:
    $$ F(h, n) = F_0 + (1 - F_0)(1 - h \cdot n)^5 $$
    [S1, unofficial]

4.  **Compute Normal Distribution Function ($D$):**
    Using the GGX distribution with the remapped roughness $\alpha$:
    $$ D(\alpha, h, n) = \frac{\alpha^2}{\pi ((h \cdot n)^2 (\alpha^2 - 1) + 1)^2} $$
    [S1, unofficial]

5.  **Compute Masking-Shadowing Function ($G$):**
    Use the correlated Smith masking-shadowing function. First, compute the lambda GGX masking function $G_{\lambda}$ for both the incident and outgoing directions:
    $$ G_{\lambda}(\alpha, n \cdot \omega_i) = \frac{1}{2\left(\sqrt{1 + \frac{\alpha(1 - (n \cdot \omega_i)^2)}{(n \cdot \omega_i)^2}} - 1\right)} $$
    [S1, unofficial]
    $$ G_{\lambda}(\alpha, n \cdot \omega_o) = \frac{1}{2\left(\sqrt{1 + \frac{\alpha(1 - (n \cdot \omega_o)^2)}{(n \cdot \omega_o)^2}} - 1\right)} $$
    [S1, unofficial]
    Then, combine them for the total geometry term $G$:
    $$ G(\alpha, n, \omega_i, \omega_o) = \frac{1}{1 + G_{\lambda}(\alpha, n \cdot \omega_i) + G_{\lambda}(\alpha, n \cdot \omega_o)} $$
    [S1, unofficial]

6.  **Compute Final BRDF:**
    Combine the diffuse and specular components:
    $$ f(\omega_o, \omega_i) = \frac{1 - m}{\pi} a + \frac{D F G}{4 (n \cdot \omega_i) (n \cdot \omega_o)} $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phys3dgs_brdf(omega_i, omega_o, n, a, m, r):
    # 1. Roughness remapping
    alpha = r * r
    
    # 2. Half vector
    h = normalize(omega_i + omega_o)
    
    # 3. Fresnel
    F0 = 0.04 * (1 - m) + a * m
    cos_theta_h = dot(h, n)
    F = F0 + (1 - F0) * (1 - cos_theta_h)**5
    
    # 4. Normal Distribution Function (GGX)
    cos_theta_h_sq = cos_theta_h * cos_theta_h
    denom_D = cos_theta_h_sq * (alpha * alpha - 1) + 1
    D = (alpha * alpha) / (pi * denom_D * denom_D)
    
    # 5. Masking-Shadowing Function (Correlated Smith)
    def lambda_ggx(alpha, cos_theta):
        # cos_theta is n . omega
        # Avoid division by zero if cos_theta is 0
        if cos_theta == 0:
            return 0.0
        term = 1 + (alpha * (1 - cos_theta * cos_theta)) / (cos_theta * cos_theta)
        # Guard against negative sqrt due to precision
        term = max(term, 0.0)
        return 1.0 / (2.0 * (sqrt(term) - 1.0))

    cos_theta_i = dot(n, omega_i)
    cos_theta_o = dot(n, omega_o)
    
    lambda_i = lambda_ggx(alpha, cos_theta_i)
    lambda_o = lambda_ggx(alpha, cos_theta_o)
    
    G = 1.0 / (1.0 + lambda_i + lambda_o)
    
    # 6. Final BRDF
    # Guard against division by zero for grazing angles
    denom_spec = 4.0 * cos_theta_i * cos_theta_o
    if denom_spec == 0:
        return 0.0
        
    specular = (D * F * G) / denom_spec
    diffuse = ((1 - m) / pi) * a
    
    return diffuse + specular
```

**Parameters:**
- $m$ (metallic): Typically optimized during training. In the source implementation, a smoothness regularizer weight $\lambda_{smooth}$ of 0.02 was applied to roughness, but the metallic value itself is a per-Gaussian attribute [S1, unofficial].
- $r$ (roughness): Per-Gaussian attribute.
- $a$ (base color): Per-Gaussian attribute.

**Edge Cases:**
- **Grazing Angles:** If $n \cdot \omega_i$ or $n \cdot \omega_o$ approaches 0, the denominator in the final BRDF formula approaches 0. The pseudocode includes a guard for this [G].
- **Zero Half Vector:** If $\omega_i = -\omega_o$, the half vector is undefined. In practice, this is a singularity that should be handled or avoided [G].
- **Precision in Lambda:** The term inside the square root for $G_{\lambda}$ must be non-negative. Numerical errors can cause negative values for grazing angles; a `max(term, 0.0)` guard is recommended [G].

## Operating characteristics
**Time and Memory Complexity:**
The computation involves standard vector operations, square roots, and powers. It is $O(1)$ per Gaussian per light/view direction. The memory overhead is minimal, requiring only the storage of the per-Gaussian attributes $a, m, r, n$.

**Strengths and Limitations:**
- **Strengths:** Provides a physically plausible specular response with energy conservation (via the $F_0$ definition and geometry term). The correlated Smith function offers a more accurate masking-shadowing model than separable approximations, especially at grazing angles [S1, unofficial].
- **Limitations:** The correlated Smith function is computationally more expensive than separable approximations (Schlick-GGX) due to the square root and division in $G_{\lambda}$ [S1, unofficial]. The roughness remapping $\alpha = r^2$ is a specific choice that may differ from other implementations using $\alpha = r$ or other mappings, affecting the visual appearance of roughness [S1, unofficial].

**When to use:**
Use this BRDF when implementing Phys3DGS for inverse rendering tasks where physical accuracy of the specular lobe is prioritized over raw rendering speed, and when the underlying Gaussian splatting framework supports per-Gaussian BRDF parameter optimization.

## Background
The Phys3DGS BRDF function was introduced in the paper "Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering" (ECCV 2024 Submission #12158) [S1, unofficial]. It is designed to enable inverse rendering by providing a differentiable, physically-based material model within the 3D Gaussian Splatting framework. The method builds on the Modified Disney Principled BRDF but adapts it for the specific requirements of Gaussian splatting optimization, including the use of correlated Smith masking-shadowing and specific roughness remapping [S1, unofficial].

## Sources
[S1] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
