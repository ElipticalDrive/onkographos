# Sphere Light Sampling with Multiple Strategies

Also called: getPixelColorWithSphereLightDBG

## Overview
This method estimates direct illumination from a spherical light source using Monte Carlo integration, supporting multiple sampling strategies including Importance Solid Angle, Importance Surface Area, Uniform Solid Angle, Uniform Hemisphere Solid Angle, Importance Phong, and Cosine Weighted sampling [S1, unofficial]. It differs from its siblings by providing a unified framework for shading against a spherical light, allowing the renderer to switch between strategies based on the geometric relationship between the shading point and the light, or to balance variance and bias [S1, unofficial]. Unlike [Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md) which samples directions within the cone of the light, or [Importance Surface Area Sampling](2223-importance-sampling-for-sphere-surface-area.md) which samples points on the sphere's geometry, this method encapsulates all these variants and [Uniform Solid Angle Sampling](2207-sphere-light-sampling-with-multiple-strategies.md#sphere-light-shading-with-uniform-solid-angle-sampling) into a single callable routine [S1, unofficial].

## How it works
**Inputs:**
- Shading point $\mathbf{X}$ (position).
- Surface normal $\mathbf{n}$ at $\mathbf{X}$.
- Spherical light object (center $\mathbf{C}$, radius $R$, emitted radiance $L_i$, material type: DIFFUSE, SPECULAR, or GLOSSY).
- Sampling mode enum: `Importance_Solid_Angle`, `Importance_Surface_Area`, `Uniform_Solid_Angle`, `Uniform_Hemisphere_Solid_Angle`, `Importance_Phong`, `Cosine_Weighted`.
- Number of samples $N$.

**Outputs:**
- Estimated direct radiance $L_r$ at $\mathbf{X}$.

**Data Structures:**
- `Vector3`: 3D vector with components $(x, y, z)$ and methods `dot`, `norm`, `cross`.
- `Sphere`: Object with `getSphereCenter()`, `getSphereRadius()`, `getNormalAt()`, `findIntersection()`.
- `Ray`: Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, with method `getTime()` for animation.
- `Color`: RGB color representation.

**Parameters:**
- `eps`: Small epsilon for intersection tests (e.g., $10^{-9}$).
- `exponent`: Phong specular exponent (default varies by material).
- `M_PI`: Mathematical constant $\pi$.

**Steps:**

1.  **Calculate Geometry:**
    Compute the vector from the shading point to the light center: $\mathbf{d}_{XC} = \mathbf{C} - \mathbf{X}$.
    Compute the distance: $D = \|\mathbf{d}_{XC}\|$.
    Compute the maximum sine and cosine of the half-angle subtended by the light:
    $$ \sin\theta_{max} = \frac{R}{D} $$
    $$ \cos\theta_{max} = \sqrt{1 - \sin^2\theta_{max}} $$
    [S1, unofficial]

2.  **Sample Direction:**
    Call the sampling function `getSample(sphere, X, mode, normal, sinThetaMax)` which returns a direction vector $\mathbf{l}$ pointing towards the light. The implementation of `getSample` switches based on `mode`:
    -   **Importance_Solid_Angle**: Samples uniformly within the cone defined by $\theta_{max}$.
    -   **Importance_Surface_Area**: Samples points on the sphere surface.
    -   **Uniform_Solid_Angle**: Samples uniformly over the full $4\pi$ sphere.
    -   **Uniform_Hemisphere_Solid_Angle**: Samples uniformly over the hemisphere above the surface.
    -   **Importance_Phong**: Samples based on the Phong lobe.
    -   **Cosine_Weighted**: Samples with cosine weighting relative to the normal.
    [S1, unofficial]

3.  **Trace Ray to Light:**
    Create a ray from $\mathbf{X}$ in direction $\mathbf{l}$. Offset the origin by $\epsilon \mathbf{n}$ to prevent self-intersection: $\mathbf{o}_{ray} = \mathbf{X} + \epsilon \mathbf{n}$.
    Trace this ray against the sphere light.
    If the ray intersects the light at distance $t > \epsilon$, proceed. Otherwise, the light is not visible in this direction (or the sample missed the light in uniform strategies), and the contribution is 0 for this sample.
    [S1, unofficial]

4.  **Shadow Check:**
    Create a shadow ray from $\mathbf{X} + \epsilon \mathbf{n}$ to the intersection point on the light $\mathbf{P}_{light}$.
    Trace this ray against all other geometry in the scene.
    If an intersection occurs at distance $t_{near}$ such that $t_{near}^2 < \|\mathbf{P}_{light} - \mathbf{X}\|^2$, the point is in shadow (`inShadow = true`). Otherwise, `inShadow = false`.
    [S1, unofficial]

5.  **Calculate Geometric Terms:**
    Let $\mathbf{l}$ be the normalized direction to the light sample.
    Let $\mathbf{n}$ be the surface normal.
    Let $\mathbf{n}_{light}$ be the normal at the intersection point on the light sphere.
    
    Calculate incident angle cosine:
    $$ \cos\theta_i = \max(0, \mathbf{l} \cdot \mathbf{n}) $$
    [S1, unofficial]
    
    Calculate outgoing angle cosine (for solid angle sampling):
    $$ \cos\theta_o = \max(0, -\mathbf{n}_{light} \cdot \mathbf{l}) $$
    [S1, unofficial]

6.  **Calculate BRDF:**
    -   **DIFFUSE**:
        $$ f_r = \frac{1}{\pi} $$
    -   **SPECULAR (Phong)**:
        Let $\mathbf{v}$ be the view direction (from $\mathbf{X}$ to camera). Let $\mathbf{r}$ be the reflection direction of $\mathbf{l}$ about $\mathbf{n}$.
        $$ \mathbf{h} = \frac{\mathbf{v} + \mathbf{l}}{\|\mathbf{v} + \mathbf{l}\|} $$
        $$ \text{temp} = \mathbf{h} \cdot \mathbf{n} $$
        $$ f_r = \frac{(\alpha + 2)}{2\pi} (-\text{temp})^\alpha $$
        *Note: The source code snippet uses `outgoing.dot(ray.getDirection())` which implies a specific reflection model. The standard Phong BRDF is used here for clarity, citing the source's structure.*
        [S1, unofficial]

7.  **Accumulate Radiance:**
    Initialize $L_r = 0$.
    For each of the $N$ samples:
    
    -   **Importance_Solid_Angle**:
        $$ \text{pdf} = 2\pi (1 - \cos\theta_{max}) $$
        $$ L_r += f_r L_i (1 - \text{inShadow}) \cos\theta_i \frac{1}{\text{pdf}} $$
        [S1, unofficial]
        
    -   **Importance_Surface_Area**:
        The projected area of the light cap is $A_{proj} = \pi R^2 \sin^2\theta_{max}$? No, the source uses a specific formula.
        $$ \text{Area} = 2\pi (1 - \sin\theta_{max}) \frac{R^2}{D^2} $$
        *Correction*: The source code says `Area = 2 * M_PI * (1 - sinThetaMax) * lightR^2 / distance2`. This looks like a projected area term.
        $$ L_r += L_i (1 - \text{inShadow}) \cos\theta_i \cos\theta_o \cdot \text{Area} \cdot f_r $$
        [S1, unofficial]
        
    -   **Uniform_Solid_Angle**:
        $$ \text{pdf} = 4\pi $$
        $$ L_r += f_r L_i (1 - \text{inShadow}) \cos\theta_i \frac{1}{\text{pdf}} $$
        Simplified in source:
        $$ L_r += f_r L_i (1 - \text{inShadow}) \cos\theta_i 4\pi \quad \text{(Wait, source says multiply by 4pi? No, divide by pdf. If pdf=4pi, then multiply by 1/4pi. The source code snippet `L_r += fr * L_i * (1 - inShadow) * cosThetaI * 4 * M_PI` suggests the PDF was $1/4\pi$ or the estimator is unweighted? Let's look closer. If sampling uniformly over 4pi, pdf is $1/4\pi$. The estimator is $L_i f_r \cos\theta_i / \text{pdf}$. So $L_i f_r \cos\theta_i 4\pi$. Yes.)} $$
        [S1, unofficial]
        
    -   **Uniform_Hemisphere_Solid_Angle**:
        $$ \text{pdf} = \frac{1}{2\pi} $$
        $$ L_r += f_r L_i (1 - \text{inShadow}) \cos\theta_i 2\pi $$
        [S1, unofficial]
        
    -   **Importance_Phong**:
        $$ L_r += L_i (1 - \text{inShadow}) \cos\theta_i $$
        *Note: The source omits BRDF and PDF terms here, implying the sampling distribution matches the Phong lobe directly.*
        [S1, unofficial]
        
    -   **Cosine_Weighted**:
        $$ L_r += L_i (1 - \text{inShadow}) f_r $$
        *Note: Source omits $\cos\theta_i$ and PDF, implying the sampling distribution includes the cosine term.*
        [S1, unofficial]

8.  **Average:**
    $$ L_r = \frac{L_r}{N} $$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sphere_light_sampling(X, normal, sphere, mode, N):
    """
    X: Vector3, shading point
    normal: Vector3, surface normal
    sphere: Sphere object with getSphereCenter(), getSphereRadius(), getNormalAt(), findIntersection()
    mode: Enum, sampling strategy
    N: int, number of samples
    Returns: Color, estimated radiance
    """
    C = sphere.getSphereCenter()
    R = sphere.getSphereRadius()
    Li = sphere.getRadiance() # Emitted radiance
    
    d_XC = C - X
    D = d_XC.norm()
    
    if D <= 0:
        return Color(0, 0, 0)
        
    sin_theta_max = R / D
    cos_theta_max = math.sqrt(1 - sin_theta_max**2)
    
    L_r = Color(0, 0, 0)
    
    for _ in range(N):
        # 1. Sample direction
        l = get_sample(sphere, X, mode, normal, sin_theta_max)
        
        # 2. Trace ray to light
        # Offset origin to prevent self-intersection
        eps = 1e-9
        ray_origin = X + normal * eps
        ray = Ray(ray_origin, l)
        
        t_light = sphere.findIntersection(ray)
        
        if t_light <= eps:
            continue # Missed light
            
        P_light = ray_origin + ray.direction * t_light
        n_light = sphere.getNormalAt(P_light)
        
        # 3. Shadow check
        shadow_ray_origin = X + normal * eps
        shadow_ray = Ray(shadow_ray_origin, l)
        
        # Trace against all other objects in scene
        # Assuming a global function trace_scene_ray exists
        t_near = trace_scene_ray(shadow_ray, exclude=sphere)
        
        dist_sq = (P_light - X).dot(P_light - X)
        in_shadow = False
        if t_near is not None and t_near**2 < dist_sq:
            in_shadow = True
            
        if in_shadow:
            continue
            
        # 4. Geometric terms
        cos_theta_i = max(0.0, l.dot(normal))
        cos_theta_o = max(0.0, -n_light.dot(l))
        
        # 5. BRDF
        material = sphere.getMaterial()
        exponent = sphere.getExponent()
        
        if material == DIFFUSE:
            fr = 1.0 / math.pi
        elif material == SPECULAR:
            # Phong BRDF
            # View direction is assumed to be from X to camera, but for direct lighting 
            # estimation in this context, we often use the reflection vector relative to l.
            # However, the source snippet uses: temp = outgoing.dot(ray.getDirection())
            # This implies 'outgoing' is the view direction. 
            # For a general path tracer, 'outgoing' is known. Here we assume it's passed or global.
            # For simplicity in this self-contained function, we assume a standard Phong lobe 
            # centered on the reflection vector.
            # Let's stick to the source's variable names if possible, but 'outgoing' is undefined.
            # We will use the standard Phong formulation: f_r = (alpha+2)/(2pi) * (h.n)^alpha
            # where h is half vector between l and v.
            # Since v is not passed, we cannot compute h. 
            # The source code snippet for Importance_Phong does not use BRDF.
            # The source code snippet for SPECULAR in the general case uses:
            # temp = outgoing.dot(ray.getDirection()) -> This looks like a Blinn-Phong term where h is approximated?
            # Or maybe 'outgoing' is the reflection of l?
            # Given the ambiguity, we will use the standard Blinn-Phong if v is available.
            # Since v is not available in this function signature, we assume the caller handles it 
            # or we use a simplified model. 
            # Let's assume the source implies a standard specular reflection.
            # We will use the formula provided in the notes:
            # brdf = pow(-1 * temp, exponent) * (exponent + 2) / (2 * M_PI)
            # where temp = outgoing.dot(ray.getDirection())
            # This is non-standard. 'outgoing' usually means view direction. 'ray.getDirection()' is l.
            # If outgoing is the reflection of l, then outgoing.dot(l) = 1.
            # Let's assume 'outgoing' is the view direction V.
            # Then h = (V + l).normalized().
            # h.dot(n) is the cosine of the half-angle.
            # The source snippet is ambiguous. We will use the standard Blinn-Phong.
            # Note: The source code for SPECULAR in the notes says:
            # brdf = pow(-1 * temp, exponent) * (exponent + 2) / (2 * M_PI)
            # This suggests temp is negative? 
            # We will use the standard Blinn-Phong for implementation clarity.
            # If the source strictly requires the 'temp' variable, it is undefined here.
            # We will skip the specific SPECULAR calculation details as they depend on 'outgoing'
            # which is not an input to this function.
            fr = 0.0 # Placeholder for SPECULAR, requires view direction
        else:
            fr = 0.0
            
        # 6. Accumulate
        if mode == Importance_Solid_Angle:
            pdf = 2 * math.pi * (1 - cos_theta_max)
            L_r += Li * fr * cos_theta_i / pdf
        elif mode == Importance_Surface_Area:
            # Source: Area = 2 * M_PI * (1 - sinThetaMax) * lightR^2 / distance2
            Area = 2 * math.pi * (1 - sin_theta_max) * (R**2) / (D**2)
            L_r += Li * cos_theta_i * cos_theta_o * Area * fr
        elif mode == Uniform_Solid_Angle:
            pdf = 4 * math.pi
            L_r += Li * fr * cos_theta_i / pdf
        elif mode == Uniform_Hemisphere_Solid_Angle:
            pdf = 1.0 / (2 * math.pi)
            L_r += Li * fr * cos_theta_i / pdf
        elif mode == Importance_Phong:
            L_r += Li * cos_theta_i # Source omits fr and pdf
        elif mode == Cosine_Weighted:
            L_r += Li * fr # Source omits cos_theta_i and pdf
            
    L_r /= N
    return L_r

def get_sample(sphere, X, mode, normal, sin_theta_max):
    if mode == Cosine_Weighted:
        return get_sample_cosine_weighted(sphere, X, normal, sin_theta_max)
    elif mode == Importance_Surface_Area:
        return get_sample_importance_surface_area(sphere, X)
    elif mode == Uniform_Solid_Angle:
        return get_sample_uniform_solid_angle(X, normal)
    elif mode == Uniform_Hemisphere_Solid_Angle:
        return get_sample_uniform_solid_angle_hemisphere(X, normal)
    elif mode == Importance_Phong:
        return get_sample_importance_phong(sphere, X, normal)
    else: # Importance_Solid_Angle
        return get_sample_importance_solid_angle(sphere, X, normal, sin_theta_max)
```

## Subtopics

### [Importance Sampling for Sphere Surface Area](2223-importance-sampling-for-sphere-surface-area.md)
Estimates direct illumination by sampling points on the visible surface area of the sphere. The PDF is proportional to the projected area. It differs from solid angle sampling by explicitly generating points on the geometry, which simplifies the geometric term calculation but requires rejection sampling or coordinate transformation to ensure uniform coverage of the visible cap.

### [Sphere Light Shading with Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md)
Estimates direct illumination by sampling directions uniformly within the cone defined by the light's extent. It differs from surface area sampling by working in solid angle space, naturally handling the geometric relationship without explicit area calculations.

### [EquiAngular Sampling](2272-equiangular-sampling.md)
Samples the distance domain along an eye ray to minimize variance when the light is partially occluded. It differs from the siblings by avoiding spherical sampling of the light source entirely, instead distributing samples uniformly in the angular domain subtended by the incident beam.

### Sphere Light Shading with Uniform Solid Angle Sampling
Estimates direct illumination by sampling uniformly over the full $4\pi$ solid angle. It differs from importance sampling strategies by ignoring the light's actual extent, leading to higher variance but simpler implementation. The PDF is $4\pi$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of samples and $K$ is the cost of ray intersection tests (shadow and light intersection).
- **Memory Complexity:** $O(1)$ auxiliary memory per sample.
- **Performance:** Importance sampling strategies (Solid Angle, Surface Area) generally provide lower variance than Uniform Solid Angle when the light is small relative to the scene or occluded. Uniform sampling is robust but inefficient for small lights.
- **Strengths:** Unified interface for multiple strategies; handles moving spheres (via `MotionSphere`); supports various material types.
- **Limitations:** Uniform sampling is inefficient for small lights. Importance sampling requires accurate calculation of $\theta_{max}$ and can fail if the shading point is inside the light. Self-intersection artifacts are mitigated by epsilon offset but can still occur in complex geometries.
- **When to use:** Use Importance Solid Angle or Surface Area for small, distant lights. Use Uniform Solid Angle for debugging or when the light covers a large portion of the hemisphere. Use Importance Phong for glossy materials where the reflection lobe is narrow.

## Background
This method originates from the `VolumeSingleScattering` project by WangYue (2016), which implements a ray tracer for volume rendering and single scattering in participating media [S1, unofficial]. It is part of a broader effort to integrate local illumination models (shadows, ambient occlusion) with volume rendering. The specific implementation `getPixelColorWithSphereLightDBG` suggests it was used for debugging or development purposes within the ray tracer.

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
