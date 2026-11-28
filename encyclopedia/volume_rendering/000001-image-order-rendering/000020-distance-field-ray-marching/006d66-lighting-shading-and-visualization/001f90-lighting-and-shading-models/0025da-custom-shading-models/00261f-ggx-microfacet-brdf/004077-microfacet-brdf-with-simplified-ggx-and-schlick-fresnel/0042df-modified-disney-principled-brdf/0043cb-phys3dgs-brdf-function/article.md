# Phys3DGS BRDF Function

Also called: Phys3DGS BRDF

## Overview
A physically-based BRDF function used in 3D Gaussian Splatting for inverse rendering, consisting of a diffuse and a specular component [S1, unofficial]. It differs from its parent, [Modified Disney Principled BRDF](10860-modified-disney-principled-brdf.md), by employing a specific roughness remapping ($\alpha = r^2$) and a correlated Smith masking-shadowing function rather than a separable approximation [S1, unofficial].

## How it works
**Inputs:**
- $\omega_i$: Incident light direction.
- $\omega_o$: Outgoing view direction.
- $\bm{n}$: Surface normal.
- $\bm{a}$: Base color (diffuse albedo).
- $m$: Metallic value (scalar).
- $r$: Roughness parameter (scalar).
[S1, unofficial]

**Outputs:**
- $f(\omega_o, \omega_i)$: Reflected radiance factor.
[S1, unofficial]

**Steps:**
1. **Compute Roughness Remapping:** $\alpha = r^2$ [S1, unofficial].
2. **Compute Half Vector:** $\bm{h} = (\omega_i + \omega_o) / \|\omega_i + \omega_o\|$ [S1, unofficial].
3. **Compute Fresnel Term ($F$):** $F_0 = 0.04(1 - m) + \bm{a}m$ [S1, unofficial]. Then $F(\bm{h}, \bm{n}) = F_0 + (1 - F_0)(1 - \bm{h} \cdot \bm{n})^5$ [S1, unofficial].
4. **Compute Normal Distribution Function ($D$):** $D(\alpha, \bm{h}, \bm{n}) = \frac{\alpha^2}{\pi ((\bm{h} \cdot \bm{n})^2 (\alpha^2 - 1) + 1)^2}$ [S1, unofficial].
5. **Compute Masking-Shadowing Function ($G$):** $G_{\lambda}(\alpha, \bm{n} \cdot \omega_i) = \frac{1}{2(\sqrt{1 + \alpha(1 - (\bm{n} \cdot \omega_i)^2)/(\bm{n} \cdot \omega_i)^2} - 1)}$ [S1, unofficial]. $G(\alpha, \bm{n}, \omega_i, \omega_o) = \frac{1}{1 + G_{\lambda}(\alpha, \bm{n} \cdot \omega_i) + G_{\lambda}(\alpha, \bm{n} \cdot \omega_o)}$ [S1, unofficial].
6. **Compute Final BRDF:** $f(\omega_o, \omega_i) = \frac{1-m}{\pi}\bm{a} + \frac{DFG}{4(\omega_i \cdot \bm{n})(\omega_o \cdot \bm{n})}$ [S1, unofficial].

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
        if cos_theta == 0:
            return 0.0
        term = 1 + (alpha * (1 - cos_theta * cos_theta)) / (cos_theta * cos_theta)
        term = max(term, 0.0)
        return 1.0 / (2.0 * (sqrt(term) - 1.0))

    cos_theta_i = dot(n, omega_i)
    cos_theta_o = dot(n, omega_o)
    
    lambda_i = lambda_ggx(alpha, cos_theta_i)
    lambda_o = lambda_ggx(alpha, cos_theta_o)
    
    G = 1.0 / (1.0 + lambda_i + lambda_o)
    
    # 6. Final BRDF
    denom_spec = 4.0 * cos_theta_i * cos_theta_o
    if denom_spec == 0:
        return 0.0
        
    specular = (D * F * G) / denom_spec
    diffuse = ((1 - m) / pi) * a
    
    return diffuse + specular
```

**Parameters:**
- $m$ (metallic): Optimized during training. Regularizer weight $\lambda_{smooth}$ is 0.02 [S1, unofficial].
- $r$ (roughness): Optimized during training. Regularizer weight $\lambda_{smooth}$ is 0.02 [S1, unofficial].
- $\bm{a}$ (base color): Optimized during training. Regularizer weight is 0.06 [S1, unofficial].

**Edge Cases:**
- **Grazing Angles:** If $\bm{n} \cdot \omega_i$ or $\bm{n} \cdot \omega_o$ approaches 0, the specular denominator approaches 0 [G].
- **Zero Half Vector:** If $\omega_i = -\omega_o$, the half vector is undefined [G].
- **Precision in Lambda:** The term inside the square root for $G_{\lambda}$ must be non-negative; numerical errors at grazing angles require a `max(term, 0.0)` guard [G].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. Standard vector operations are $O(1)$ per evaluation [G].

**Strengths and Limitations:**
Not given in the sources.

**When to use:**
Not given in the sources.

## Background
The Phys3DGS BRDF function was introduced in "Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering" (ECCV 2024 Submission #12158) [S1, unofficial]. It provides a differentiable, physically-based material model within the 3D Gaussian Splatting framework for inverse rendering [S1, unofficial].

## Sources
[S1] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
