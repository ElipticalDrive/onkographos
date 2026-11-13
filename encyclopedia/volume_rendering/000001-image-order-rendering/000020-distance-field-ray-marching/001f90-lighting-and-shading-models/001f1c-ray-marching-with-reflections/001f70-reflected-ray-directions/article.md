# Reflected Ray Directions

Also called: Reflection, reflect, Refract & Reflect Utilities, Snell's Law Implementation, Vector Reflection

## Overview
Reflected Ray Directions is a geometric method used in volume rendering and ray marching to determine the propagation direction of a ray after it interacts with a surface boundary. It solves the problem of simulating specular reflection by computing a new ray vector based on the incident direction and the surface normal. It differs from sibling methods like [Refracted Ray Directions](3915-refracted-ray-directions.md) by computing transmission (bending) rather than reflection (bouncing), and serves as the geometric foundation for energy-splitting techniques like [Fresnel Reflection/Refraction](9067-fresnel-reflection-refraction.md).

## How it works
**Inputs:**
- Incident ray direction vector $\vec{v}$ (normalized).
- Surface normal vector $\vec{n}$ at the intersection point (normalized).
- Implicit surface function $f(\vec{p})$ and its gradient (for normal calculation).

**Outputs:**
- Reflected ray direction vector $\vec{v}_{out}$ (normalized).

**Data Structures:**
- Vectors are represented as 3D floating-point tuples $(x, y, z)$.
- Normals and directions are unit vectors ($|\vec{v}| = 1$).

**Steps:**
1. **Evaluate Surface Hit:** Determine if the current ray position $\vec{p}_i$ is on the surface. This is typically done by checking if the signed distance function value $f(\vec{p}_i) \approx 0$ or is below a small threshold $\epsilon$ [S2, unofficial].
2. **Compute Normal:** If a hit is detected, compute the surface normal $\vec{n}$ at $\vec{p}_i$. For an implicit surface, $\vec{n} = \nabla f(\vec{p}_i)$, normalized [S3, unofficial].
3. **Calculate Reflection:** Compute the reflected direction using the vector reflection formula [S1, S2, S3, unofficial]:
   $$ \vec{v}_{out} = \vec{v} - 2(\vec{v} \cdot \vec{n})\vec{n} $$
   Alternatively, expressed as:
   $$ \vec{v}_{out} = \vec{v} - \vec{a}, \quad \text{where } \vec{a} = \vec{n} \cdot (2 \cdot (\vec{v} \cdot \vec{n})) $$
   [S1, unofficial]
4. **Normalize:** Ensure $\vec{v}_{out}$ is normalized to unit length to prevent drift in subsequent ray marching steps [S3, unofficial].
5. **Fallback:** If the position is not on the surface ($f(\vec{p}_i) \not\approx 0$), the direction remains unchanged: $\vec{v}_{out} = \vec{v}$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def reflect_ray(v: tuple, n: tuple) -> tuple:
    """
    Calculate the reflected ray direction.
    
    Args:
        v: Incident ray direction (x, y, z), assumed normalized.
        n: Surface normal (x, y, z), assumed normalized.
        
    Returns:
        Reflected ray direction (x, y, z), normalized.
    """
    # Dot product of incident direction and normal
    dot_vn = v[0]*n[0] + v[1]*n[1] + v[2]*n[2]
    
    # Reflection formula: v_out = v - 2 * (v . n) * n
    # Calculate the scalar component 2 * (v . n)
    scalar = 2.0 * dot_vn
    
    # Calculate the vector component to subtract: a = n * scalar
    a_x = n[0] * scalar
    a_y = n[1] * scalar
    a_z = n[2] * scalar
    
    # Result: v - a
    r_x = v[0] - a_x
    r_y = v[1] - a_y
    r_z = v[2] - a_z
    
    # Normalize the result
    length = math.sqrt(r_x*r_x + r_y*r_y + r_z*r_z)
    
    # Guard against zero length (degenerate case)
    if length < 1e-8:
        return (0.0, 0.0, 0.0)
        
    inv_len = 1.0 / length
    return (r_x * inv_len, r_y * inv_len, r_z * inv_len)

def compute_reflected_direction(v: tuple, p: tuple, sdf_func) -> tuple:
    """
    Determine if reflection should occur based on surface proximity.
    
    Args:
        v: Current ray direction.
        p: Current ray position.
        sdf_func: Signed Distance Function f(p).
        
    Returns:
        New ray direction.
    """
    # Check if on surface (approximate zero)
    # Threshold epsilon depends on implementation; 1e-4 is common for SDFs
    epsilon = 1e-4
    dist = sdf_func(p)
    
    if abs(dist) < epsilon:
        # Compute normal from SDF gradient or pre-computed
        # For this example, assume normal is passed or computed externally
        # Here we simulate getting normal N_i
        n = compute_normal(p) # [G] Helper to compute gradient of SDF at p
        
        return reflect_ray(v, n)
    else:
        # Not on surface, continue straight
        return v
```

**Parameters:**
- $\epsilon$: Surface proximity threshold. Typical value: $10^{-4}$ to $10^{-3}$ depending on scale [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision Errors:** Floating-point inaccuracies can cause the reflected vector to drift from unit length. Always re-normalize [S3, unofficial].
- **Total Internal Reflection (TIR):** While primarily a reflection method, if used in conjunction with refraction logic, TIR cases must be handled. In pure reflection, TIR is not applicable, but if the normal calculation is inverted (back-face vs front-face), the reflection will be incorrect. Ensure the normal points against the incident ray ($\vec{v} \cdot \vec{n} < 0$ for incoming rays) [S1, unofficial].
- **Degenerate Input:** If $\vec{v}$ and $\vec{n}$ are parallel or zero, the result may be undefined. Guard against zero-length vectors [S1, G, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ per reflection event. Requires one dot product, a few multiplications, and a normalization.
- **Memory Complexity:** $O(1)$. No additional data structures required.
- **Strengths:** Computationally cheap; geometrically exact for specular surfaces; easy to implement.
- **Limitations:** Only models perfect specular reflection; does not account for roughness (requires microfacet models); sensitive to normal accuracy.
- **When to use:** When simulating mirror-like surfaces in volume rendering or ray marching.
- **When not to use:** For diffuse surfaces (use Lambertian models) or rough surfaces (use GGX/Beckmann).

## Background
The vector reflection formula is a fundamental concept in geometric optics and computer graphics, derived from the law of reflection. It is widely used in ray tracing and ray marching implementations. The specific implementation details in modern codebases often mirror the standard derivation found in graphics literature.

## Sources
[S1] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
[S2] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S3] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
