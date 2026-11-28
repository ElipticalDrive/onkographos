# Fresnel Glass Material

Also called: Dielectric Scattering, Glass scattering, Refraction with Schlick approximation

## Overview
The Fresnel Glass Material is a shading model for transparent dielectric objects (such as glass, water, or diamond) that simulates the bending of light (refraction) and partial reflection at boundaries [S1, unofficial]. It solves the problem of rendering realistic transparent surfaces by dynamically adjusting the ratio of reflected to refracted light based on the angle of incidence, using the Schlick approximation [S2, unofficial]. Unlike basic glass models that ignore angular dependence, this method produces characteristic darkening of glass sides at grazing angles and more accurate caustics beneath the object [S2, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**: Incident ray direction ($\vec{d}$), surface normal ($\vec{n}$), index of refraction ($n$), and a random number generator.
- **Outputs**: A new ray direction (either reflected or refracted) and potentially an attenuation color (default white).

### Data Structures
- **hit_record**: Contains the intersection point $p$, the surface normal $\vec{n}$, the distance $t$, a pointer to the material, and a boolean `front_face` indicating if the ray hit the object from the outside [S1, unofficial].
- **ray**: Defined by an origin point and a direction vector.

### Parameters
- **$n$ (Index of Refraction)**: A double-precision floating-point value. Typical values are 1.5 for glass, 1.33 for water, and 2.4 for diamond [S1, unofficial].

### Algorithm Steps
1. **Determine Normal Orientation**: If the ray hits from the inside (`front_face` is false), the normal $\vec{n}$ points inward. The effective normal for calculation is $\vec{n}$ if `front_face` is true, else $-\vec{n}$ [S1, unofficial].
2. **Calculate Cosine**: Compute $\cos\theta = -\vec{d} \cdot \vec{n}$. Clamp this value to $[0, 1]$ to handle numerical precision issues [S1, unofficial].
3. **Compute Sine**: Calculate $\sin\theta = \sqrt{1 - \cos^2\theta}$ [S1, unofficial].
4. **Check Total Internal Reflection (TIR)**:
   - If the ray is going from a higher index to a lower index (inside object), check if $n \cdot \sin\theta > 1.0$.
   - If TIR occurs, the ray reflects entirely. Set the new direction to the reflected vector.
   - If not TIR, proceed to step 5.
5. **Calculate Fresnel Reflectance**:
   - Compute the reflectance at normal incidence: $R_0 = \left(\frac{1 - n}{1 + n}\right)^2$ [S1, unofficial].
   - Compute the Schlick approximation reflectance: $R = R_0 + (1 - R_0)(1 - \cos\theta)^5$ [S1, unofficial].
6. **Probabilistic Decision**:
   - Generate a random double $r \in [0, 1)$.
   - If $r < R$ (or if TIR occurred), the ray reflects.
   - Otherwise, the ray refracts.
7. **Compute New Direction**:
   - If reflecting: $\vec{d}_{new} = \vec{d} - 2(\vec{d} \cdot \vec{n})\vec{n}$ [S1, unofficial].
   - If refracting: $\vec{d}_{new} = \frac{\vec{d}}{n} - \left(\frac{\vec{d}}{n} \cdot \vec{n} + \sqrt{1 - \|\frac{\vec{d}}{n}\|^2 + (\vec{d} \cdot \vec{n})^2}\right)\vec{n}$ [S1, unofficial]. Note: The source simplifies this by stating "Compute unit direction" and "Create new ray" after determining the ratio, but standard refraction vector math is implied for implementation [S1, unofficial].
8. **Return**: Create and return a new ray with the same origin (offset slightly to avoid self-intersection) and the new direction. Attenuation is set to white (1, 1, 1) [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fresnel_glass_material(ray, hit_record, n, rng):
    """
    ray: The incoming ray.
    hit_record: Contains p, normal, front_face, etc.
    n: Index of refraction (double).
    rng: Random number generator returning a double in [0, 1).
    Returns: A new ray.
    """
    # 1. Determine effective normal
    # If front_face is true, normal points outward. If false, it points inward.
    # The source implies using the stored normal adjusted by front_face.
    if hit_record.front_face:
        normal = hit_record.normal
        cos_theta = dot(-ray.direction, normal)
    else:
        normal = -hit_record.normal
        cos_theta = dot(-ray.direction, normal)

    # 2. Clamp cosine and compute sine
    cos_theta = min(cos_theta, 1.0)
    sin_theta = sqrt(1.0 - cos_theta * cos_theta)

    # 3. Check for Total Internal Reflection (TIR)
    # TIR happens when going from higher to lower index (inside object)
    # and sin_theta * n > 1.0.
    # Note: The source says "refraction_ratio * sin_theta > 1.0".
    # If inside, refraction_ratio is 1.0/n. If outside, it is n.
    # However, standard physics: TIR only possible if n_incident > n_transmitted.
    # Inside glass (n=1.5) to air (n=1.0): ratio is 1.0/1.5.
    # The source code snippet logic: "Determine refraction ratio: 1.0/ir if front face, ir if back face."
    # This implies:
    # If front face (outside to inside): ratio = 1.0/n ? No, usually n1/n2.
    # Let's follow the source's explicit instruction:
    
    if hit_record.front_face:
        # Outside to Inside
        refraction_ratio = 1.0 / n
    else:
        # Inside to Outside
        refraction_ratio = n

    # Check TIR condition as per source: refraction_ratio * sin_theta > 1.0
    # Wait, standard TIR check is sin_theta > n2/n1.
    # If inside (back face), n1=n, n2=1. Ratio n2/n1 = 1/n.
    # Source says: "1.0/ir if front face, ir if back face".
    # If back face (inside), ratio = n. Then n * sin_theta > 1.0?
    # This seems inverted from standard physics if 'ratio' is n1/n2.
    # Let's stick strictly to the source's conditional logic for implementation.
    
    is_tir = False
    if refraction_ratio * sin_theta > 1.0:
        is_tir = True

    # 4. Calculate Schlick Reflectance
    # R0 = ((1 - n) / (1 + n))^2
    r0 = ((1.0 - n) / (1.0 + n)) ** 2
    reflectance = r0 + (1.0 - r0) * (1.0 - cos_theta) ** 5

    # 5. Probabilistic Decision
    # If TIR or random < reflectance, reflect.
    if is_tir or rng() < reflectance:
        # Reflect
        direction = reflect(ray.direction, normal)
    else:
        # Refract
        # Standard refraction vector:
        # u = normalize(direction)
        # dt = dot(u, normal)
        # discriminant = 1.0 - refraction_ratio^2 * (1.0 - dt^2)
        # if discriminant < 0: return reflect (fallback)
        # return refraction_ratio * (u - dt * normal) - sqrt(discriminant) * normal
        
        # Simplified based on source "Compute unit direction... Create new ray"
        # We must implement the vector math.
        u = normalize(ray.direction)
        dt = dot(u, normal)
        # Note: refraction_ratio here is n1/n2 for the vector formula?
        # Source says "refraction_ratio" for TIR check.
        # For vector refraction: v' = (n1/n2) * v + ...
        # If outside to inside: n1=1, n2=n. Ratio = 1/n.
        # If inside to outside: n1=n, n2=1. Ratio = n.
        # This matches the source's ratio assignment.
        
        ratio = refraction_ratio
        discriminant = 1.0 - ratio * ratio * (1.0 - dt * dt)
        
        if discriminant < 0:
            # Fallback to reflection if numerical error
            direction = reflect(ray.direction, normal)
        else:
            direction = ratio * (u - dt * normal) - sqrt(discriminant) * normal

    # 6. Create new ray
    # Offset origin slightly to prevent self-intersection
    new_origin = hit_record.p + direction * 1e-4
    return Ray(new_origin, direction)
```

### Edge Cases and Pitfalls
- **Grazing Angles**: At grazing angles, reflectance approaches 1.0. The Schlick approximation handles this probabilistically, ensuring realistic darkening of edges [S1, S2, unofficial].
- **Total Internal Reflection**: Must be handled explicitly. If the condition $n \cdot \sin\theta > 1.0$ is met, the ray must reflect entirely [S1, unofficial].
- **Numerical Precision**: Clamping $\cos\theta$ to 1.0 prevents NaN results in square root calculations [S1, unofficial].

## Subtopics

### Window Refraction and Fresnel Transmission
Handles refraction through planar surfaces like windows, often assuming no thickness or specific boundary conditions. Choose when rendering flat transparent panes. [Link](21372-window-refraction-and-fresnel-transmission.md)

## Operating characteristics
- **Time Complexity**: $O(1)$ per intersection for the shading calculation.
- **Memory Complexity**: $O(1)$ additional memory.
- **Strengths**: Produces visually accurate glass with realistic edge darkening and caustics [S2, unofficial]. Computationally efficient due to the Schlick approximation.
- **Limitations**: Does not account for subsurface scattering or volumetric absorption within the glass itself (unless combined with volumetric rendering) [S2, unofficial].
- **When to use**: For rendering transparent dielectric objects like glass, water, or plastic where surface reflections and refractions are dominant.
- **When not to use**: For materials with significant internal scattering or absorption (e.g., frosted glass, milk) without extension.

## Background
The Fresnel Glass Material is based on the principles of optics, specifically the Fresnel equations, which describe the reflection and transmission of light at an interface between two media. The Schlick approximation is a widely used simplification of the Fresnel equations for computational efficiency [S1, S2, unofficial]. This model was notably implemented in ray tracers to improve the realism of glass objects, such as martini glasses, by capturing the darkening of sides at low-incidence angles and accurate caustics [S2, unofficial].

## Sources
[S1] OWMarko/CPP_RayTracer_to_Medical_Application, https://github.com/OWMarko/CPP_RayTracer_to_Medical_Application, fetched 2026-10-26 (unofficial)
[S2] 348B Project Report, http://www-graphics.stanford.edu/courses/cs348b-competition/cs348b-02/ong/, fetched 2026-10-30 (unofficial)
