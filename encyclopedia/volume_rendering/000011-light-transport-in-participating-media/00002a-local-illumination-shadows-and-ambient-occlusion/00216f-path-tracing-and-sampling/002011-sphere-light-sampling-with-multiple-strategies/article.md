# Sphere Light Sampling with Multiple Strategies

Also called: getPixelColorWithSphereLightDBG

## Overview
Calculates direct lighting from a spherical light source using Monte Carlo integration with various sampling strategies, including Importance Solid Angle, Importance Surface Area, Uniform Solid Angle, and others [S1, unofficial]. It sets itself apart from siblings like [Dwivedi Sampling](5537-dwivedi-sampling.md) by explicitly handling the geometry of a spherical light source through multiple distinct probability density functions (PDFs) rather than a single fixed approach, allowing the renderer to choose the most efficient estimator for the light's position relative to the shading point.

## How it works
**Inputs:**
*   Shading point $\mathbf{X}$ (position in world space).
*   Surface normal $\mathbf{n}$ at $\mathbf{X}$.
*   Light source: a sphere defined by center $\mathbf{C}$, radius $R$, and emitted radiance $L_i$.
*   Sampling mode $\mathcal{M}$: an enum selecting the strategy (e.g., `Importance_Solid_Angle`, `Uniform_Solid_Angle`, `Cosine_Weighted`).
*   Number of samples $N$.

**Outputs:**
*   Estimated direct radiance $L_r$ at $\mathbf{X}$ from the spherical light.

**Data Structures:**
*   `Vector3`: 3D floating-point vector $(x, y, z)$.
*   `Color`: 3D floating-point vector representing RGB radiance.
*   `Ray`: Origin $\mathbf{o}$, direction $\mathbf{d}$, time $t$.
*   `Sphere`: Center $\mathbf{C}$, radius $R$, material properties (exponent for Phong, material type for BRDF).

**Steps:**

1.  **Calculate Geometric Limits:**
    Compute the distance $d$ from $\mathbf{X}$ to the light center $\mathbf{C}$:
    $$ d = \| \mathbf{X} - \mathbf{C} \| $$
    Compute the maximum sine and cosine of the half-angle $\theta_{max}$ subtended by the light sphere:
    $$ \sin\theta_{max} = \frac{R}{d} $$
    $$ \cos\theta_{max} = \sqrt{1 - \sin^2\theta_{max}} $$
    [S1, unofficial]

2.  **Sample Direction:**
    Generate a sample direction $\mathbf{l}$ towards the light sphere based on the selected mode $\mathcal{M}$ and $\sin\theta_{max}$.
    *   For `Importance_Solid_Angle`, `Importance_Surface_Area`, `Importance_Phong`, and `Cosine_Weighted`, the direction is constrained within the cone defined by $\theta_{max}$.
    *   For `Uniform_Solid_Angle` and `Uniform_Hemisphere_Solid_Angle`, the direction is sampled over the full sphere or hemisphere, respectively.
    [S1, unofficial]

3.  **Trace Ray to Light:**
    Construct a ray $\mathbf{r}$ from $\mathbf{X}$ in direction $\mathbf{l}$.
    Find the intersection distance $t$ to the light sphere.
    If $t > \epsilon$ (where $\epsilon$ is a small epsilon, e.g., $10^{-9}$), the ray hits the light. Proceed to shading.
    If no hit or $t \le \epsilon$, the light is not visible in this sample direction.
    [S1, unofficial]

4.  **Shadow Check:**
    Trace a shadow ray from $\mathbf{X}$ (offset by $\epsilon \mathbf{n}$ to prevent self-intersection) to the intersection point on the light sphere $\mathbf{P}_{light} = \mathbf{X} + t \mathbf{l}$.
    Check for intersections with other objects in the scene.
    Let $t_{near}$ be the distance to the first intersection.
    If an intersection exists such that $t_{near}^2 < d^2$ (or simply $t_{near} < t$), the point is in shadow ($inShadow = true$).
    Otherwise, $inShadow = false$.
    [S1, unofficial]

5.  **Compute Shading Terms:**
    *   Light direction: $\mathbf{l}_{dir} = \mathbf{P}_{light} - \mathbf{X}$ (normalized).
    *   Incident angle cosine: $\cos\theta_i = \max(0, \mathbf{l}_{dir} \cdot \mathbf{n})$.
    *   Light surface normal: $\mathbf{n}_{light} = \mathbf{P}_{light} - \mathbf{C}$ (normalized).
    *   Outgoing angle cosine (for BSDF): $\cos\theta_o = \max(0, -\mathbf{n}_{light} \cdot \mathbf{l}_{dir})$.
    [S1, unofficial]

6.  **Evaluate BRDF:**
    *   If Material is `DIFFUSE`:
        $$ f_r = \frac{1}{\pi} $$
    *   If Material is `SPECULAR` (Phong):
        Let $\mathbf{v}$ be the view direction (from $\mathbf{X}$ to camera, or $-\mathbf{l}_{dir}$ if evaluating for reflection).
        Let $\mathbf{r}_{refl}$ be the perfect reflection direction.
        $$ \text{temp} = \mathbf{v} \cdot \mathbf{r}_{refl} $$
        $$ f_r = \text{pow}(\max(0, \text{temp}), \text{exponent}) \cdot \frac{\text{exponent} + 2}{2\pi} $$
    [S1, unofficial]

7.  **Accumulate Radiance:**
    Initialize $L_r = 0$.
    For each of the $N$ samples:
    *   Determine the PDF $p(\mathbf{l})$ based on the mode.
    *   Calculate the contribution:
        $$ L_{sample} = L_i \cdot (1 - inShadow) \cdot f_r \cdot \cos\theta_i \cdot \frac{1}{p(\mathbf{l})} \cdot \text{GeometryFactor} $$
        The specific formula depends on the mode:
        *   **Importance_Solid_Angle:**
            $$ p(\mathbf{l}) = 2\pi (1 - \cos\theta_{max}) $$
            $$ L_r += L_i \cdot (1 - inShadow) \cdot \cos\theta_i \cdot \frac{1}{p(\mathbf{l})} \cdot \text{Note: Source code implies direct multiplication by PDF inverse in estimator, but source snippet shows `L_r += L_i * (1 - inShadow) * cosThetaI * pdfConstant` where `pdfConstant` is the denominator? No, standard MC is $L \cdot f \cdot \cos / p$. The source snippet says `L_r += L_i * (1 - inShadow) * cosThetaI * pdfConstant`. This implies the BRDF and other terms are handled elsewhere or the snippet is simplified. Let's look at Uniform: `L_r += fr * L_i * (1 - inShadow) * cosThetaI * 4 * M_PI`. Here $p = 1/4\pi$, so $1/p = 4\pi$. This matches. So for Importance Solid Angle, $p = 1 / (2\pi(1-\cos\theta_{max}))$, so $1/p = 2\pi(1-\cos\theta_{max})$. The source says `pdfConstant = 2 * M_PI * (1 - cosThetaMax)`. So the term is $L_i \cdot (1-inShadow) \cdot \cos\theta_i \cdot \frac{1}{p}$. Wait, the source snippet for Importance Surface Area says `L_r += L_i * (1 - inShadow) * cosThetaI * cosThetaO * Area * brdf`. This is not divided by PDF. This suggests the source code might be implementing a specific estimator form or the "Area" term includes the PDF inverse. Let's stick to the explicit formulas provided in the notes.
            
            *Correction based on Source Snippets:*
            The source notes provide specific accumulation lines. We must follow them exactly.
            
            *   **Importance_Solid_Angle:**
                $$ L_r += L_i \cdot (1 - inShadow) \cdot \cos\theta_i \cdot 2\pi(1 - \cos\theta_{max}) $$
                (Note: This assumes the BRDF is handled separately or this is a simplified direct lighting term. The Uniform case includes `fr`. The Importance Solid Angle note does *not* explicitly list `fr` in the accumulation line, but `Importance_Surface_Area` does. However, physically, BRDF is required. The source snippet for `Uniform_Solid_Angle` includes `fr`. The `Importance_Solid_Angle` note says `L_r += L_i * (1 - inShadow) * cosThetaI * pdfConstant`. It omits `fr`. This might be an error in the source notes or a specific context (e.g., Lambertian assumed). Given `Importance_Surface_Area` includes `brdf`, we will include `brdf` if it's a general shading model, but the note for `Importance_Solid_Angle` is explicit. Let's look at `Importance_Phong`: `L_r += L_i * (1 - inShadow) * cosThetaI`. It also omits `fr`. This suggests that for "Importance" modes, the BRDF might be evaluated differently or the note is incomplete. However, `Uniform_Solid_Angle` and `Cosine_Weighted` include `fr`. To be safe and implementable, we will use the explicit formulas from the notes. If `fr` is missing in the note, we follow the note, but add a comment [G] if it seems physically necessary. Actually, looking at `Importance_Surface_Area`, it has `brdf`. `Importance_Solid_Angle` does not. This is a discrepancy. We will write the formula as given in the note for each mode.)
                
                *Re-reading carefully:*
                `Importance_Solid_Angle uses pdfConstant = 2 * M_PI * (1 - cosThetaMax).`
                `L_r += L_i * (1 - inShadow) * cosThetaI * pdfConstant` -> This looks like $L \cdot \cos \cdot \frac{1}{p}$. It misses $f_r$.
                `Importance_Surface_Area uses Area = ... L_r += L_i * (1 - inShadow) * cosThetaI * cosThetaO * Area * brdf`.
                `Uniform_Solid_Angle uses constant 4 * M_PI. L_r += L_i * (1 - inShadow) * cosThetaI * 4 * M_PI * brdf`.
                `Importance_Phong uses only L_i * (1 - inShadow) * cosThetaI`.
                `Cosine_Weighted uses L_i * (1 - inShadow) * brdf`.
                
                We will implement exactly these accumulation terms.
                
        *   **Importance_Surface_Area:**
            $$ \text{Area} = \frac{2\pi (1 - \sin\theta_{max}) R^2}{d^2} $$
            $$ L_r += L_i \cdot (1 - inShadow) \cdot \cos\theta_i \cdot \cos\theta_o \cdot \text{Area} \cdot f_r $$
            [S1, unofficial]
            
        *   **Uniform_Solid_Angle:**
            $$ L_r += f_r \cdot L_i \cdot (1 - inShadow) \cdot \cos\theta_i \cdot 4\pi $$
            [S1, unofficial]
            
        *   **Uniform_Hemisphere_Solid_Angle:**
            $$ L_r += f_r \cdot L_i \cdot (1 - inShadow) \cdot \cos\theta_i \cdot 2\pi $$
            [S1, unofficial]
            
        *   **Importance_Phong:**
            $$ L_r += L_i \cdot (1 - inShadow) \cdot \cos\theta_i $$
            [S1, unofficial]
            
        *   **Cosine_Weighted:**
            $$ L_r += L_i \cdot (1 - inShadow) \cdot f_r $$
            [S1, unofficial]

8.  **Average:**
    $$ L_r = \frac{1}{N} \sum_{k=1}^{N} L_{sample, k} $$
    [S1, unofficial]

**Pitfalls:**
*   **Self-Intersection:** The shadow ray origin must be offset by $\epsilon \mathbf{n}$ (e.g., $10^{-9}$) along the normal to prevent the ray from intersecting the surface it originated from. [S1, unofficial]
*   **Precision:** When $\mathbf{X}$ is very close to the light center, $d \to 0$, causing division by zero or numerical instability in $\sin\theta_{max}$. Guard against $d < \epsilon$.
*   **Shadow Ray Bias:** Using a fixed epsilon for shadow ray offset can cause acne or peter-panning if not scaled appropriately.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sphere_light_sampling(X, normal, light_sphere, sample_mode, N):
    """
    X: Vector3, shading point
    normal: Vector3, surface normal at X
    light_sphere: Sphere object with center C, radius R, radiance L_i, material
    sample_mode: Enum, one of the strategies
    N: int, number of samples
    Returns: Color, estimated radiance
    """
    C = light_sphere.center
    R = light_sphere.radius
    L_i = light_sphere.get_radiance()
    
    # Distance from shading point to light center
    vec_to_light = C - X
    d_sq = vec_to_light.dot(vec_to_light)
    d = math.sqrt(d_sq)
    
    # Guard against zero distance
    if d < 1e-9:
        return Color(0, 0, 0)
        
    sin_theta_max = R / d
    cos_theta_max = math.sqrt(max(0.0, 1.0 - sin_theta_max * sin_theta_max))
    
    total_L_r = Color(0, 0, 0)
    
    for _ in range(N):
        # 1. Sample direction
        l_dir = get_sample_direction(X, normal, light_sphere, sample_mode, sin_theta_max)
        
        # 2. Trace ray to light
        ray = Ray(X, l_dir)
        t_hit = light_sphere.find_intersection(ray)
        
        if t_hit <= 1e-9:
            continue # Light not hit
            
        P_light = X + l_dir * t_hit
        
        # 3. Shadow Check
        # Offset origin to prevent self-intersection
        shadow_origin = X + normal * 1e-9
        shadow_ray = Ray(shadow_origin, l_dir)
        
        # Check intersection with OTHER objects (scene)
        # Note: The light sphere itself is the target, so we check scene objects
        t_shadow = scene.find_first_intersection(shadow_ray, exclude=light_sphere)
        
        in_shadow = False
        if t_shadow is not None and t_shadow < t_hit:
            in_shadow = True
            
        # 4. Shading Terms
        # Light direction (normalized)
        l_vec = P_light - X
        l_dir_norm = l_vec.normalize()
        
        cos_theta_i = max(0.0, l_dir_norm.dot(normal))
        
        # Light normal
        n_light = (P_light - C).normalize()
        cos_theta_o = max(0.0, -n_light.dot(l_dir_norm))
        
        # BRDF
        fr = evaluate_brdf(light_sphere.material, l_dir_norm, normal, light_sphere.exponent)
        
        # 5. Accumulate
        L_sample = Color(0, 0, 0)
        
        if sample_mode == SampleMode.IMPORTANCE_SOLID_ANGLE:
            pdf_const = 2.0 * math.pi * (1.0 - cos_theta_max)
            # Note: Source note omits fr in accumulation line for this mode
            L_sample = L_i * (1.0 - in_shadow) * cos_theta_i * pdf_const
            
        elif sample_mode == SampleMode.IMPORTANCE_SURFACE_AREA:
            # Area formula from source
            # Area = 2 * M_PI * (1 - sinThetaMax) * lightR^2 / distance2
            area = (2.0 * math.pi * (1.0 - sin_theta_max) * R * R) / d_sq
            L_sample = L_i * (1.0 - in_shadow) * cos_theta_i * cos_theta_o * area * fr
            
        elif sample_mode == SampleMode.UNIFORM_SOLID_ANGLE:
            L_sample = fr * L_i * (1.0 - in_shadow) * cos_theta_i * 4.0 * math.pi
            
        elif sample_mode == SampleMode.UNIFORM_HEMISPHERE_SOLID_ANGLE:
            L_sample = fr * L_i * (1.0 - in_shadow) * cos_theta_i * 2.0 * math.pi
            
        elif sample_mode == SampleMode.IMPORTANCE_PHONG:
            L_sample = L_i * (1.0 - in_shadow) * cos_theta_i
            
        elif sample_mode == SampleMode.COSINE_WEIGHTED:
            L_sample = L_i * (1.0 - in_shadow) * fr
            
        else:
            # Default fallback or error
            pass
            
        total_L_r += L_sample
        
    return total_L_r / N

def get_sample_direction(X, normal, light_sphere, mode, sin_theta_max):
    """
    Returns a direction vector towards the light sphere based on mode.
    """
    C = light_sphere.center
    R = light_sphere.radius
    
    if mode == SampleMode.IMPORTANCE_SOLID_ANGLE:
        return get_sample_importance_solid_angle(C, R, X, normal, sin_theta_max)
    elif mode == SampleMode.IMPORTANCE_SURFACE_AREA:
        return get_sample_importance_surface_area(C, R, X)
    elif mode == SampleMode.UNIFORM_SOLID_ANGLE:
        return get_sample_uniform_solid_angle(X, normal)
    elif mode == SampleMode.UNIFORM_HEMISPHERE_SOLID_ANGLE:
        return get_sample_uniform_hemisphere_solid_angle(X, normal)
    elif mode == SampleMode.IMPORTANCE_PHONG:
        return get_sample_importance_phong(C, R, X, normal)
    elif mode == SampleMode.COSINE_WEIGHTED:
        return get_sample_cosine_weighted(C, R, X, normal, sin_theta_max)
    else:
        return get_sample_importance_solid_angle(C, R, X, normal, sin_theta_max)

def get_sample_uniform_solid_angle(X, normal):
    """
    Sample direction uniformly over the full sphere.
    """
    # Generate random angles
    u1 = random.random()
    u2 = random.random()
    
    theta = 2.0 * math.pi * u1
    phi = math.acos(2.0 * u2 - 1.0)
    
    # Convert to Cartesian
    x = math.sin(phi) * math.cos(theta)
    y = math.sin(phi) * math.sin(theta)
    z = math.cos(phi)
    
    sample = Vector3(x, y, z)
    
    # Align with normal
    # Create coordinate system
    W, V = create_coordinate_system(normal)
    
    # Transform sample
    result = Vector3(
        sample.x * V.x + sample.y * W.x + sample.z * normal.x,
        sample.x * V.y + sample.y * W.y + sample.z * normal.y,
        sample.x * V.z + sample.y * W.z + sample.z * normal.z
    )
    
    return result.normalize()

def get_sample_uniform_hemisphere_solid_angle(X, normal):
    """
    Sample direction uniformly over the hemisphere above the surface.
    """
    u1 = random.random()
    u2 = random.random()
    
    theta = 2.0 * math.pi * u1
    phi = math.acos(math.sqrt(u2)) # For uniform hemisphere, cos(phi) is uniform
    
    x = math.sin(phi) * math.cos(theta)
    y = math.sin(phi) * math.sin(theta)
    z = math.cos(phi)
    
    sample = Vector3(x, y, z)
    
    W, V = create_coordinate_system(normal)
    
    result = Vector3(
        sample.x * V.x + sample.y * W.x + sample.z * normal.x,
        sample.x * V.y + sample.y * W.y + sample.z * normal.y,
        sample.x * V.z + sample.y * W.z + sample.z * normal.z
    )
    
    return result.normalize()

def create_coordinate_system(normal):
    """
    Creates two orthogonal vectors V and W to form a basis with normal.
    """
    # Simple method: pick an arbitrary vector not parallel to normal
    if abs(normal.x) > 0.9:
        aux = Vector3(1, 0, 0)
    else:
        aux = Vector3(0, 1, 0)
        
    V = aux.cross(normal).normalize()
    W = normal.cross(V).normalize()
    
    return V, W

def evaluate_brdf(material, l_dir, normal, exponent):
    if material == Material.DIFFUSE:
        return 1.0 / math.pi
    elif material == Material.SPECULAR:
        # Phong
        # Reflect vector
        # I is incoming direction (l_dir), N is normal
        # R = 2 * (N . -I) * N - (-I) ? 
        # Standard reflection: R = I - 2(I.N)N where I is incoming (pointing TO surface)
        # Here l_dir is pointing FROM surface TO light.
        # View direction v is from surface to camera.
        # For direct lighting estimation, we usually evaluate the BSDF for the direction to the light.
        # If we are just shading the point, we need the BRDF value for the l_dir.
        # Phong BRDF: f_r = k_d/pi + k_s * (R . V)^n / pi ? 
        # Source snippet: pow(-1 * temp, exponent) * (exponent + 2) / (2 * M_PI)
        # temp = outgoing.dot(ray.getDirection())
        # This implies a specific formulation. 
        # Let's assume standard Phong lobe centered at reflection.
        
        # Reflection of l_dir about normal
        # R = l_dir - 2 * (l_dir . normal) * normal ? No, that's if l_dir is incoming.
        # If l_dir is outgoing (to light), the reflection of the view vector would be used.
        # But here we are sampling the light. The "outgoing" direction in the snippet 
        # `temp = outgoing.dot(ray.getDirection())` suggests `outgoing` is the view direction.
        # In a direct lighting pass, we might not have a view direction if we are just computing illumination.
        # However, the snippet implies `outgoing` is available.
        # Let's assume `outgoing` is the view direction passed to the function or -l_dir if evaluating reflection.
        # For simplicity, if we don't have a view direction, we can't compute Phong specular.
        # We will assume `outgoing` is passed or is -l_dir (backscattering).
        
        # Re-reading snippet: `temp = outgoing.dot(ray.getDirection())`
        # `ray.getDirection()` is l_dir.
        # So temp = V . l_dir.
        # This is not the standard Phong reflection lobe.
        # Standard Phong: (R . V)^n.
        # The snippet uses `outgoing` (V) and `ray.getDirection()` (l_dir).
        # This looks like a simplified or specific approximation.
        # We will implement exactly as per snippet.
        
        # Note: The snippet uses `-1 * temp`.
        # If V and l_dir are both pointing away from surface, their dot product is positive for backscattering.
        # We need the view direction. If not provided, we cannot compute this accurately.
        # [G] Assuming `outgoing` is the view direction. If not available, return 0 or diffuse.
        pass 
        
    return 1.0 / math.pi # Fallback

def get_sample_importance_solid_angle(C, R, X, normal, sin_theta_max):
    # Implementation details omitted for brevity, follows cone sampling
    pass

def get_sample_importance_surface_area(C, R, X):
    # Implementation details omitted
    pass

def get_sample_importance_phong(C, R, X, normal):
    # Implementation details omitted
    pass

def get_sample_cosine_weighted(C, R, X, normal, sin_theta_max):
    # Implementation details omitted
    pass
```

## Subtopics

### [Importance Sampling for Sphere Surface Area](2223-importance-sampling-for-sphere-surface-area.md)
Estimates direct illumination from a spherical light by sampling points on its visible surface area, with a probability density function proportional to the projected area of the light as seen from the shading point. It differs from [Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md) by explicitly generating points on the sphere's geometry rather than in solid angle space.

### [Sphere Light Shading with Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md)
This method estimates direct illumination from a spherical light source using Monte Carlo integration over the solid angle subtended by the light. It sets itself apart from surface-area sampling by sampling directions uniformly within the cone defined by the light's extent, which naturally handles the geometric relationship between the shading point and the light without requiring explicit area calculations. It is a specific strategy within the broader [Sphere Light Sampling with Multiple Strategies](2207-sphere-light-sampling-with-multiple-strategies.md) framework.

### [EquiAngular Sampling](2272-equiangular-sampling.md)
EquiAngular Sampling is a Monte Carlo integration technique used to estimate direct lighting from point lights in participating media by importance sampling the distance domain along a ray. It solves the high variance caused by the $1/r^2$ geometric falloff and potential partial occlusion of the light source by sampling propagation distances proportional to the inverse squared length of the connecting edge. Unlike uniform solid angle or surface area sampling, which treat the light as a distant or extended object, this method re-parameterizes the integration into angular steps relative to the orthogonal projection of the light onto the ray, ensuring equal angular contributions. It is specifically designed for single scattering in homogeneous media.

### Sphere Light Shading with Uniform Solid Angle Sampling
Estimates direct illumination from a spherical light by sampling uniformly over the hemisphere above the surface [S1, unofficial]. It differs from full-sphere uniform sampling by restricting samples to the upper hemisphere, which is appropriate for surface shading where light comes from above [S1, unofficial]. The PDF is $1/2\pi$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of samples and $K$ is the cost of ray intersection tests. Each sample requires tracing a ray to the light and a shadow ray.
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample, excluding scene data structures.
*   **Performance:** Importance sampling strategies (Solid Angle, Surface Area) generally provide lower variance than Uniform Solid Angle when the light is small or partially occluded, as they concentrate samples in the direction of the light. Uniform sampling wastes samples on directions that do not hit the light.
*   **Strengths:** Flexible; allows switching strategies based on light size and position. Handles extended spherical lights accurately.
*   **Limitations:** Requires explicit intersection tests with the light sphere. Shadow ray bias can cause artifacts if not handled carefully. The source code's handling of BRDF in Importance modes is inconsistent with standard rendering equations, potentially leading to incorrect results if not corrected [S1, unofficial].

## Background
Originated in the context of volume rendering and participating media, specifically in implementations like `VolumeSingleScattering` by WangYue (2016) [S1, unofficial]. The method is part of a broader effort to efficiently compute direct lighting in scenes with complex geometry and media, leveraging multiple sampling strategies to optimize convergence.

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
