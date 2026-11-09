# EquiAngular Sampling

Also called: Equiangular distance sampling, Kulla and Fajardo sampling, density sampling for point lights

## Overview
EquiAngular Sampling is a Monte Carlo integration technique used to estimate direct lighting from point lights in participating media by importance sampling the distance domain along a ray [S1][S3, unofficial][S4]. It solves the high variance caused by the $1/r^2$ geometric falloff and potential partial occlusion of the light source by sampling propagation distances proportional to the inverse squared length of the connecting edge [S1, S4]. Unlike uniform solid angle or surface area sampling, which treat the light as a distant or extended object, this method re-parameterizes the integration into angular steps relative to the orthogonal projection of the light onto the ray, ensuring equal angular contributions [S4][S5, unofficial]. It is specifically designed for single scattering in homogeneous media [S3, unofficial][S4].

## How it works
**Inputs:**
- Ray origin $\mathbf{x}$ and direction $\vec{\omega}$ [S4].
- Point light position $\mathbf{K}$ [S3, unofficial][S4].
- Light flux $\Phi$ [S3, unofficial][S4].
- Medium extinction coefficient $\sigma_t$ and scattering coefficient $\sigma_s$ [S3, unofficial][S4].
- Integration bounds $a, b$ along the ray (typically $0$ to $t_{max}$ or intersection distance) [S4].
- Uniform random number $\xi \in [0, 1)$ [S3, unofficial][S4].

**Outputs:**
- Sampled distance $t$ along the ray [S4].
- PDF value $pdf(t)$ [S4].
- Radiance contribution estimator [S3, unofficial][S4].

**Data Structures:**
- No complex data structures required. Standard floating-point variables for geometry and physics parameters.

**Steps:**

1.  **Calculate Geometric Parameters:**
    Compute the orthogonal projection of the light source $\mathbf{K}$ onto the ray defined by $\mathbf{x}$ and $\vec{\omega}$.
    Let $\mathbf{v} = \mathbf{K} - \mathbf{x}$.
    The signed distance from the ray origin to the projection point is $\Delta = \mathbf{v} \cdot \vec{\omega}$ [S4].
    The perpendicular distance from the light to the ray is $D = \|\mathbf{v} - \Delta \vec{\omega}\|$ [S4].
    *Note:* If $D=0$, the light lies on the ray, causing a singularity. In practice, clamp $D$ to a small epsilon or handle as a special case [G].

2.  **Determine Angular Bounds:**
    Calculate the angles subtended by the integration bounds $a$ and $b$ relative to the projection point.
    The angles $\theta_a$ and $\theta_b$ are defined as:
    $$ \theta_a = \arctan\left(\frac{a - \Delta}{D}\right) $$
    $$ \theta_b = \arctan\left(\frac{b - \Delta}{D}\right) $$
    [S4]. Note that $a$ and $b$ can be negative relative to the new origin, so $\theta_a$ and $\theta_b$ can be negative [S4].

3.  **Sample Distance:**
    Generate a uniform random number $\xi \in [0, 1)$ [S3, unofficial][S4].
    Compute the sampled distance $t$ along the ray using the inverse CDF:
    $$ t = D \tan\left((1 - \xi)\theta_a + \xi\theta_b\right) + \Delta $$
    [S4].
    *Clarification:* Some sources define the formula without the $\Delta$ shift if the integration variable $t$ is defined relative to the projection point [S2, S3, unofficial]. If $t$ is defined relative to the ray origin $\mathbf{x}$, the $\Delta$ term must be added. The PDF derivation in [S4] assumes $t$ is the distance from the projection point for the $D^2+t^2$ term, but the final estimator uses the absolute position. We define $t_{proj} = D \tan(\dots)$ and the absolute distance from origin is $t_{abs} = t_{proj} + \Delta$. The PDF $pdf(t)$ in [S4] is for the variable $t$ in the denominator $D^2+t^2$, which corresponds to the distance from the projection point.

4.  **Calculate PDF:**
    The probability density function for the sampled distance $t$ (where $t$ in the formula represents the distance from the orthogonal projection, i.e., $t_{proj}$) is:
    $$ pdf(t_{proj}) = \frac{D}{(\theta_b - \theta_a)(D^2 + t_{proj}^2)} $$
    [S4].
    If the estimator requires the PDF with respect to the distance from the ray origin, the Jacobian of the transformation is 1, so the value remains the same for the specific sample location.

5.  **Estimate Radiance:**
    Compute the in-scattered radiance contribution. The transmittance from the ray origin $\mathbf{x}$ to the sample point $\mathbf{x}_t$ (at distance $t_{abs}$) and from $\mathbf{x}_t$ to the light $\mathbf{K}$ must be calculated.
    Distance from $\mathbf{x}$ to $\mathbf{x}_t$ is $t_{abs}$ [S4].
    Distance from $\mathbf{x}_t$ to $\mathbf{K}$ is $r = \sqrt{D^2 + t_{proj}^2}$ [S4].
    Transmittance $T_r(\mathbf{x}, \mathbf{x}_t) = e^{-\sigma_t t_{abs}}$ [S3, unofficial][S4].
    Transmittance $T_r(\mathbf{x}_t, \mathbf{K}) = e^{-\sigma_t r}$ [S4].
    Visibility $Vis(\mathbf{x}_t, \mathbf{K})$ is determined by shadow ray tracing [S3, unofficial].
    The estimator for a single sample is:
    $$ L = \frac{\sigma_s \cdot T_r(\mathbf{x}, \mathbf{x}_t) \cdot T_r(\mathbf{x}_t, \mathbf{K}) \cdot Vis(\mathbf{x}_t, \mathbf{K}) \cdot \frac{\Phi}{r^2}}{pdf(t_{proj})} $$
    [S3, unofficial][S4].
    *Note:* The term $\frac{\Phi}{r^2}$ is the incident radiance from the point light at distance $r$. The phase function $p(\vec{\omega}_i, \vec{\omega}_o)$ is omitted here for simplicity but should be included if anisotropic scattering is modeled [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def equiangular_sample(ray_origin, ray_direction, light_pos, flux, sigma_t, sigma_s, t_min, t_max):
    """
    Samples a point on the ray towards a point light using equiangular sampling.
    
    Args:
        ray_origin: Vector3 (x, y, z)
        ray_direction: Vector3 (normalized)
        light_pos: Vector3 (x, y, z)
        flux: float (Phi)
        sigma_t: float (extinction coefficient)
        sigma_s: float (scattering coefficient)
        t_min: float (start of integration interval, usually 0)
        t_max: float (end of integration interval, usually distance to intersection)
        
    Returns:
        t_abs: float (sampled distance from ray_origin)
        pdf: float (probability density)
        radiance_contrib: float (estimated radiance contribution)
    """
    
    # 1. Geometric setup
    v = light_pos - ray_origin
    delta = dot(v, ray_direction) # Signed distance to projection
    
    # Perpendicular distance D
    proj_point = ray_origin + ray_direction * delta
    perp_vec = light_pos - proj_point
    D = length(perp_vec)
    
    # Handle singularity
    if D < 1e-6:
        D = 1e-6
        
    # 2. Angular bounds
    # Angles relative to the projection point
    theta_a = math.atan((t_min - delta) / D)
    theta_b = math.atan((t_max - delta) / D)
    
    # 3. Sample
    xi = random.uniform(0.0, 1.0)
    
    # Sampled distance from projection point
    t_proj = D * math.tan((1.0 - xi) * theta_a + xi * theta_b)
    
    # Absolute distance from ray origin
    t_abs = t_proj + delta
    
    # 4. PDF
    # PDF is for t_proj. Since dt_abs/dt_proj = 1, pdf(t_abs) = pdf(t_proj)
    pdf = D / ((theta_b - theta_a) * (D**2 + t_proj**2))
    
    # 5. Radiance Estimator
    # Distance from sample to light
    r = math.sqrt(D**2 + t_proj**2)
    
    # Transmittances
    # T(x, x_t)
    T1 = math.exp(-sigma_t * t_abs)
    # T(x_t, K)
    T2 = math.exp(-sigma_t * r)
    
    # Visibility (Shadow Ray)
    # In a real implementation, this would be a function call
    # For the sake of the algorithm description, we assume a shadow ray is cast
    # and returns 1.0 if visible, 0.0 if occluded.
    # Note: The shadow ray origin should be slightly offset from x_t to avoid self-intersection.
    shadow_origin = ray_origin + ray_direction * (t_abs + 1e-4)
    shadow_dir = normalize(light_pos - shadow_origin)
    visibility = cast_shadow_ray(shadow_origin, shadow_dir) 
    
    # Incident Radiance from Point Light
    # L_incident = Phi / r^2
    L_incident = flux / (r**2)
    
    # Estimator
    # Numerator: sigma_s * T1 * T2 * Vis * L_incident
    # Denominator: pdf
    numerator = sigma_s * T1 * T2 * visibility * L_incident
    radiance_contrib = numerator / pdf
    
    return t_abs, pdf, radiance_contrib
```

**Parameters:**
- $D$: Perpendicular distance from light to ray [S4].
- $\theta_a, \theta_b$: Angular bounds [S4].
- $t_{max}$: Maximum distance along the ray [S2, unofficial].
- $spr$ (samples per ray): Typical value around 4-5 yields good results with low noise [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Singularity:** If $D=0$ (light on ray), the PDF becomes undefined. Clamp $D$ to a small value [G].
- **Occlusion:** The method assumes the visibility term is handled separately via shadow rays. If the light is partially occluded, the variance is reduced compared to uniform sampling, but noise remains if the shadow ray is expensive or inaccurate [S3, unofficial].
- **Large Lights:** This method is derived for point lights. For area lights, $D$ becomes a function of the sampled point on the light surface, introducing approximation errors if the center is used for $D$ [S4][S5, unofficial].

## Subtopics

### [Area Light Sampling with Ray-Origin Centering](8975-area-light-sampling-with-ray-origin-centering.md)
Generalizes equiangular sampling to arbitrary area lights by using the sampled light point as the center for the ray-segment sampling distribution, reducing variance at light edges.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample for the sampling and PDF calculation. Dominated by shadow ray tracing for visibility [S3, unofficial].
- **Memory Complexity:** $O(1)$.
- **Performance:** Renders single scattering in homogeneous media in less than 10 minutes with $spp < 100$ and $spr = 4$ on standard hardware [S3, unofficial].
- **Strengths:** Significantly lower variance than free-path sampling or uniform solid angle sampling for single scattering [S3, unofficial]. Handles partial occlusion better than transmittance sampling [S1].
- **Limitations:** Not suitable for multiple scattering (free-path sampling is preferred for that) [S3, unofficial]. Approximation errors increase for large area lights if not generalized [S4][S5, unofficial]. Cannot match the tail behavior of exponential sampling in some diffusion contexts [S5, unofficial].

## Background
Equiangular sampling was proposed by Kulla and Fajardo [KF12] and used by Novák et al. [NNDJ12] for rendering single scattering in participating media [S5, unofficial]. It is based on the idea of importance sampling the geometry term $G(\mathbf{b}, \mathbf{c}) \propto 1/\|\mathbf{b}-\mathbf{c}\|^2$ [S1]. The technique was further analyzed in the context of area lights in participating media [S4].

## Sources
[S1] Joint importance sampling of low-order volumetric scattering, https://doi.org/10.1145/2508363.2508411, fetched 2026-10-28
[S2] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S3] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S4] Importance sampling of area lights in participating media, https://doi.org/10.1145/2037826.2037899, fetched 2026-11-02
[S5] Photon Beam Diffusion for Rendering, https://www.scribd.com/document/255860491/Photon-Beam-Diffusion, fetched 2026-10-29 (unofficial)
