# Refracted Ray Directions

Also called: Vector Snell's Law Refraction, refract

## Overview
Calculates the direction of a ray after it passes through a discrete interface between two media with different refractive indices, handling both entry and exit scenarios [S1, unofficial]. It differs from [Nested Raymarching for Refraction](1957-nested-raymarching-for-refraction.md) and [Piecewise evaluation of Snell's law](16699-piecewise-evaluation-of-snell-s-law.md) by treating the medium as homogeneous within the object and applying refraction only at the boundary, rather than integrating a gradient or recursively marching through nested volumes [S1, unofficial].

## How it works
**Inputs:**
- `ray`: Incident ray direction (normalized vector).
- `N`: Surface normal at the intersection point (normalized vector, pointing outward from the object surface).
- `ref`: Current refractive index of the medium the ray is traveling in.
- `n_obj`: Refractive index of the object being intersected.

**Outputs:**
- `direction_refracted`: The new ray direction after refraction (normalized vector).
- `ref_new`: The refractive index of the medium the ray is now traveling in.

**Data Structures:**
- Vectors are 3D floating-point vectors.
- Scalars are double-precision floats.

**Steps:**
1. Compute the dot product $s = \text{dot}(\text{ray}, N)$.
2. Determine the relative refractive index $\mu$ and the indices $n_1, n_2$:
   - If the ray is exiting the object (detected by checking if the object is a sphere and $s > 0$, or generally if the normal points in the same direction as the ray), swap indices: $n_1 = n_{obj}$, $n_2 = ref$, and $\mu = n_{obj} / ref$. The normal $N$ remains as computed.
   - If the ray is entering the object, $n_1 = ref$, $n_2 = n_{obj}$, and $\mu = n_{obj} / ref$.
   - [G] Note: In standard implementations, the normal $N$ is always the outward surface normal. If $s > 0$, the ray is exiting, and the effective normal for Snell's law calculation should be $-N$ to maintain the convention that the normal points into the incident medium, or the formula must account for the sign of $s$. The source code implies handling the swap of $n_1/n_2$ and potentially the normal direction.
3. Compute the dot product $c = \text{dot}(\text{ray}, N)$.
4. Calculate the intermediate variable $tmp$:
   $$ tmp = 1 - \mu^2 (1 - c^2) $$
5. Check for Total Internal Reflection (TIR):
   - If $tmp < 0$, TIR occurs. The ray does not refract. The caller should handle this (e.g., by reflecting the ray or terminating the path).
6. Calculate the refracted direction:
   $$ \text{direction\_refracted} = \sqrt{tmp} \cdot N + \mu \cdot (\text{ray} - c \cdot N) $$
7. Normalize $\text{direction\_refracted}$.
8. Update the current refractive index: $ref_{new} = n_2$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def refract(ray, N, ref, n_obj):
    """
    ray: incident ray direction (normalized)
    N: surface normal (normalized, outward)
    ref: current medium refractive index
    n_obj: object refractive index
    returns: (refracted_direction, new_ref_index)
    """
    s = dot(ray, N)
    
    # Determine direction of travel relative to surface
    # If s > 0, ray is exiting (normal points same way as ray)
    # If s < 0, ray is entering (normal points opposite to ray)
    
    if s > 0:
        # Exiting
        n1 = n_obj
        n2 = ref
        mu = n_obj / ref
        # Normal N is outward. For Snell's law vector form, 
        # we typically use the normal pointing into the incident medium.
        # However, the formula derived in [S1, unofficial] uses N directly with specific mu definitions.
        # Let's follow [S1, unofficial] logic:
        # S1 says: if exiting, swap n1/n2.
        # S1 formula: direction_refracted = sqrt(tmp) * normals + mu * (directions - c * normals)
        # This formula assumes N is the normal used in the dot product c.
        # If we are exiting, the ray goes from n_obj to ref.
        # The standard vector refraction formula is:
        # v_out = (n1/n2) * v_in + ((n1/n2)*cos(theta_i) - sqrt(1 - (n1/n2)^2 * sin^2(theta_i))) * N
        # where N is the normal pointing OUT of the surface.
        # Let's stick to the [S1, unofficial] specific implementation details.
        
        # [S1, unofficial] Code snippet:
        # float n1 = ref; float n2 = ob->getRefractive();
        # if(exiting) swap n1, n2.
        # So if exiting: n1=n_obj, n2=ref.
        # mu = n1/n2 ? No, [S1, unofficial] says mu = n_ext / n_lens for entering.
        # Let's re-read [S1, unofficial] carefully.
        # S2: mu = n_ext / n_lens for entering.
        # S2: mu = n_lens / n_ext for exiting.
        # This implies mu = n_incident / n_transmitted.
        
        mu = n1 / n2 # n_incident / n_transmitted
        
        # c = dot(ray, N)
        c = dot(ray, N)
        
        # tmp = 1 - mu^2 * (1 - c^2)
        tmp = 1.0 - mu * mu * (1.0 - c * c)
        
        if tmp < 0:
            return None # Total Internal Reflection
        
        # direction_refracted = sqrt(tmp) * N + mu * (ray - c * N)
        dir_ref = sqrt(tmp) * N + mu * (ray - c * N)
        dir_ref = normalize(dir_ref)
        
        return dir_ref, n2
        
    else:
        # Entering
        n1 = ref
        n2 = n_obj
        mu = n1 / n2 # n_incident / n_transmitted
        
        c = dot(ray, N)
        tmp = 1.0 - mu * mu * (1.0 - c * c)
        
        if tmp < 0:
            return None # Total Internal Reflection
            
        dir_ref = sqrt(tmp) * N + mu * (ray - c * N)
        dir_ref = normalize(dir_ref)
        
        return dir_ref, n2
```

**Edge Cases and Pitfalls:**
- **Total Internal Reflection:** If $tmp < 0$, the square root is invalid. This occurs when the angle of incidence exceeds the critical angle. The method must detect this and return a failure flag or handle reflection instead.
- **Precision:** Near grazing angles, $tmp$ may be very close to 0. Use epsilon guards if necessary.
- **Normal Orientation:** The formula relies on consistent normal orientation. If $N$ is not the outward normal, the sign of $s$ and the resulting direction will be incorrect.

## Subtopics

### [Fresnel Reflection/Refraction](9067-fresnel-reflection-refraction.md)
Calculates the Fresnel reflectance for unpolarized light at an interface between two media with different indices of refraction. It determines the ratio of reflected to refracted light based on the angle of incidence, enabling realistic rendering of transparent and reflective surfaces. Unlike basic glass models that ignore angular dependence, this method produces characteristic darkening of glass sides at grazing angles and more accurate caustics beneath the object. It serves as the core weighting function for combining reflected and refracted ray contributions in path tracing and ray marching.

### [Piecewise evaluation of Snell's law](16699-piecewise-evaluation-of-snell-s-law.md)
An approximation technique for continuous refraction that repeatedly evaluates Snell's law at discrete steps along a ray path. It serves as a middle ground between constant refraction (single-step) and higher-order numerical integration (e.g., RK4) for rays traversing media with spatially varying refractive indices. Unlike [Refracted Ray Directions](3915-refracted-ray-directions.md), which handles discrete interfaces, this method approximates the curved trajectory within a gradient medium by treating it as a sequence of small, homogeneous slabs.

### [Thick Lens (Snell's Law Refraction)](21374-thick-lens-snell-s-law-refraction.md)
Models a thick lens using Snell's law at two spherical interfaces, accounting for lens thickness and curvature. It differs from thin lens approximations by explicitly computing ray propagation through the lens material volume.

## Operating characteristics
- **Time Complexity:** $O(1)$ per interface intersection.
- **Memory Complexity:** $O(1)$.
- **Strengths:** Exact solution for planar or spherical interfaces between homogeneous media. Computationally inexpensive.
- **Limitations:** Does not handle gradient-index (GRIN) media. Requires accurate surface normals. Fails for total internal reflection without additional handling.
- **When to use:** Rendering transparent objects with constant refractive indices (glass, water, plastic).
- **When not to use:** Rendering atmospheric effects, heat haze, or GRIN lenses where refraction varies continuously.

## Background
Originated from standard geometric optics and Snell's Law. The specific vector formulation and implementation details for ray tracing are found in early ray tracing literature and graphics programming repositories [S1, unofficial]. It is a fundamental building block for rendering transparent and refractive materials.

## Sources
[S1] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
[S2] magis-slac/gradoptics, https://github.com/magis-slac/gradoptics, fetched 2026-10-27 (unofficial)
