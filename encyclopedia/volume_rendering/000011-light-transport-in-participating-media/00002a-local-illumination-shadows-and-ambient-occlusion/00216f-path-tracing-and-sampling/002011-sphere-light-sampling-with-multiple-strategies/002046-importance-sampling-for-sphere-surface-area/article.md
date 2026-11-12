# Importance Sampling for Sphere Surface Area

Also called: Sphere Light Shading with Importance Surface Area Sampling, getSampleImportanceSolidAngle

## Overview
Estimates direct illumination from a spherical light source by sampling points uniformly on the visible surface area of the sphere, with a probability density function (PDF) proportional to the projected area [S1, unofficial]. Unlike [Importance Sampling for Sphere Surface Area](2236-sphere-light-shading-with-importance-solid-angle-sampling.md) which samples the solid angle subtended by the light, this method samples the physical surface points directly, requiring a geometric correction factor in the estimator to account for the varying projected area relative to the viewing direction [S1, unofficial].

## How it works
**Inputs:**
- Intersection point $X$ on the shading surface.
- Surface normal $N$ at $X$.
- Sphere light center $C$.
- Sphere light radius $R$.
- Two uniform random numbers $\eta_1, \eta_2 \in [0, 1)$.

**Outputs:**
- Sample direction $D$ from $X$ to a point on the sphere surface.
- Contribution to radiance estimator.

**Data Structures:**
- Vectors $U, V, W$ form an orthonormal basis aligned with the local geometry.
- Scalar `distance2XC`: squared Euclidean distance between $X$ and $C$.

**Steps:**

1.  **Compute Geometry:**
    Calculate the vector from the intersection point to the light center:
    $$ \vec{XC} = C - X $$
    Compute the squared distance:
    $$ \text{distance2XC} = \|\vec{XC}\|^2 $$
    Compute the half-angle sine and cosine of the light cone:
    $$ \sin\theta_{\max} = \frac{R}{\|\vec{XC}\|} $$
    $$ \cos\theta_{\max} = \sqrt{1 - \sin^2\theta_{\max}} $$

2.  **Construct Local Basis:**
    Define the local coordinate system basis vectors to align with the surface normal and the light direction:
    $$ W = \frac{\vec{XC}}{\|\vec{XC}\|} $$
    $$ V = \frac{W \times N}{\|W \times N\|} $$
    $$ U = V \times W $$
    [G] If $W$ and $N$ are parallel (degenerate case), $V$ is undefined. In this case, choose an arbitrary vector orthogonal to $W$ for $V$ (e.g., if $|W_x| < 0.9$, $V = (1,0,0) \times W$, else $V = (0,1,0) \times W$), then recompute $U$.

3.  **Sample Angles:**
    Generate two uniform random numbers $\eta_1, \eta_2 \in [0, 1)$.
    Compute the polar angle $\Theta$ and azimuthal angle $\Phi$:
    $$ \Theta = \arccos\left(1 - \eta_1 + \eta_1 \sqrt{1 - \frac{R^2}{\text{distance2XC}}}\right) $$
    $$ \Phi = 2\pi \eta_2 $$

4.  **Construct Direction:**
    Construct the sample direction in the local coordinate system:
    $$ D_{\text{local}} = U (\cos\Phi \sin\Theta) + V (\sin\Phi \sin\Theta) + W \cos\Theta $$
    Normalize $D_{\text{local}}$ and return $D = D_{\text{local}} / \|D_{\text{local}}\|$.

5.  **Estimator Update:**
    Trace a ray from $X$ in direction $D$ to determine shadow status $(1 - \text{inShadow})$.
    Calculate the projected area factor:
    $$ \text{Area} = \frac{2\pi (1 - \sin\theta_{\max}) R^2}{\text{distance2XC}} $$
    Update the radiance estimator $L_r$:
    $$ L_r += f_r(\omega_i, \omega_o) L_i \cdot (1 - \text{inShadow}) \cdot \cos\theta_i \cdot \cos\theta_o \cdot \text{Area} $$
    Average over $N$ samples.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_sphere_surface_area(X, normal, light_center, light_radius, eta1, eta2):
    # 1. Geometry
    XC = light_center - X
    distance2XC = dot(XC, XC)
    dist = sqrt(distance2XC)
    
    sin_theta_max = light_radius / dist
    # cos_theta_max = sqrt(1 - sin_theta_max**2) # Not strictly needed for direction, but for Area
    
    # 2. Local Basis
    W = normalize(XC)
    
    # Handle degenerate case where W is parallel to normal
    cross_WN = cross(W, normal)
    if length(cross_WN) < 1e-6:
        # Choose arbitrary orthogonal vector
        if abs(W.x) < 0.9:
            V = normalize(cross(W, Vector3(1, 0, 0)))
        else:
            V = normalize(cross(W, Vector3(0, 1, 0)))
    else:
        V = normalize(cross_WN)
        
    U = cross(V, W)
    
    # 3. Sample Angles
    # Formula: Theta = acos(1 - eta1 + eta1 * sqrt(1 - (R^2 / distance2XC)))
    term = sqrt(1.0 - (light_radius * light_radius) / distance2XC)
    cos_theta = 1.0 - eta1 + eta1 * term
    # Clamp for numerical stability
    cos_theta = max(-1.0, min(1.0, cos_theta))
    theta = acos(cos_theta)
    phi = 2.0 * PI * eta2
    
    sin_theta = sin(theta)
    cos_phi = cos(phi)
    sin_phi = sin(phi)
    
    # 4. Construct Direction
    # D_local = U * (cos(Fi) * sin(Theta)) + V * sin(Fi) * sin(Theta) + W * cos(Theta)
    D_local = U * (cos_phi * sin_theta) + V * (sin_phi * sin_theta) + W * cos_theta
    D = normalize(D_local)
    
    return D

def update_estimator(X, normal, D, light_center, light_radius, fr, Li, in_shadow):
    XC = light_center - X
    distance2XC = dot(XC, XC)
    dist = sqrt(distance2XC)
    
    sin_theta_max = light_radius / dist
    
    # Area factor derived from projected solid angle integration
    # Area = 2 * pi * (1 - sin(theta_max)) * R^2 / d^2
    Area = 2.0 * PI * (1.0 - sin_theta_max) * (light_radius * light_radius) / distance2XC
    
    # Cosine of angle between surface normal and incoming light direction D
    # Note: D points from X to Light. Normal points out.
    # If D and Normal are in same hemisphere, dot > 0.
    cos_theta_i = dot(normal, D)
    
    # Cosine of angle between outgoing ray (ray direction) and normal
    # Assuming ray direction is opposite to normal for reflection? 
    # In standard rendering: L_r += fr * Li * cos_theta_i * Area
    # The source code includes cos_theta_o. If ray direction is V, and normal is N,
    # cos_theta_o = dot(V, N). If V is the ray direction from previous bounce.
    # For direct lighting from surface point X, outgoing direction is V.
    # However, in the source snippet, cosThetaO is used. 
    # Standard rendering equation: L_o = int f_r L_i cos(theta_i) d omega
    # Here Area approximates the integral over the solid angle.
    
    # Source code: L_r += fr * L_i * (1 - inShadow) * cosThetaI * cosThetaO * Area
    # We assume cosThetaO is provided or is 1 if calculating irradiance, 
    # but typically for BRDF shading it is dot(ray_dir, normal).
    # Since this is a "Light Sampling" step, we are computing contribution to L_o.
    # The term cosThetaO usually refers to the outgoing direction's cosine.
    # If this function is called from a path tracing loop, ray_dir is known.
    # For the purpose of this article, we assume cosThetaO is passed or computed externally.
    # However, looking at the source snippet:
    # L_r += fr * L_i * (1 - inShadow) * cosThetaI * cosThetaO * Area
    # It implies cosThetaO is part of the estimator.
    
    contribution = fr * Li * (1.0 - in_shadow) * cos_theta_i # * cosThetaO * Area
    # Note: The source code multiplies by Area. 
    # The standard MIS weight or PDF division is implicit in the Area term derivation 
    # which accounts for the PDF of the surface area sampling.
    
    return contribution * Area
```

## Subtopics

### Importance Surface Area Sampling
A variant that explicitly loops to generate a point $Y$ on the sphere surface and checks visibility. It generates $\eta_1, \eta_2$, computes $\theta = \arccos(1 - 2\eta_1)$ and $\phi = 2\pi\eta_2$, then constructs $Y = (R\sin\theta\cos\phi, R\sin\theta\sin\phi, R\cos\theta) + C$. It computes the normal $C_Y$ at $Y$ and the vector $YX = X - Y$. If $C_Y \cdot YX \le \epsilon$, the sample is rejected and the loop repeats. This rejection sampling approach ensures the sampled point faces the shading point $X$. Parameter $\epsilon$ is a floating-point threshold for the dot product test [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample for the analytical method. The rejection-based variant has expected $O(1)$ but can be slower due to rejections if the light is small or occluded.
- **Memory Complexity:** $O(1)$.
- **Strengths:** Intuitive geometric interpretation; samples the physical light surface directly.
- **Limitations:** Requires computing the local basis and handling degenerate cases. The PDF is not uniform in solid angle, requiring the specific Area correction factor.
- **When to use:** When the light source is a sphere and direct surface sampling is preferred over solid angle sampling, or when implementing specific light sampling strategies in a path tracer.

## Background
Originated in the context of ray tracing implementations for spherical lights, specifically in the `VolumeSingleScattering` repository by wangyuehi [S1, unofficial]. It is one of several sampling strategies (alongside Importance Solid Angle, Uniform Solid Angle, etc.) implemented to estimate direct illumination in participating media and surface rendering.

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
