# Reflected Ray Directions

Also called: Reflection, reflect, Vector Reflection

## Overview
A method to determine the direction of a ray after it interacts with a surface during distance-field ray marching. It calculates the new propagation direction based on the incident ray and the surface normal, enabling specular highlights and environment mapping in volume rendering. It differs from [Ray Marching with Refraction and Beer's Law Absorption](1947-ray-marching-with-refraction-and-beer-s-law-absorption.md) by preserving the ray within the same medium (no index of refraction change) and from [Approximate Refraction](3072-approximate-refraction.md) by providing an exact geometric reflection rather than an approximation.

## How it works
**Inputs:**
- Incident ray direction $\vec{v}$ (normalized).
- Surface normal $\vec{n}$ at the intersection point (normalized).
- Optional: Distance field value $f(p)$ to determine if a hit occurred.

**Outputs:**
- Reflected ray direction $\vec{v}_{new}$ (normalized).

**Data Structures:**
- Vectors $\vec{v}, \vec{n}$ are 3D floating-point vectors.

**Steps:**
1. **Surface Hit Detection:** Determine if the ray has hit a surface. In distance-field marching, this is typically when the distance field value $f(p_i) \approx 0$ (within a small epsilon) [S2, unofficial]. If no surface is hit, the direction remains unchanged: $\vec{v}_{i+1} = \vec{v}_i$ [S2, unofficial].
2. **Normal Retrieval:** If a surface is hit, compute the surface normal $\vec{N}_i$ at the current position $p_i$. For distance fields, this is usually the normalized gradient: $\vec{N}_i = \frac{\nabla f(p_i)}{\|\nabla f(p_i)\|}$ [S2, S3, unofficial].
3. **Dot Product Calculation:** Compute the dot product between the incident direction and the normal: $d = \vec{v} \cdot \vec{n}$ [S1, S3, unofficial].
4. **Reflection Vector Calculation:** Apply the reflection formula. The reflected direction $\vec{r}$ is calculated as:
   $$ \vec{r} = \vec{v} - 2(\vec{v} \cdot \vec{n})\vec{n} $$
   This is equivalent to subtracting twice the projection of $\vec{v}$ onto $\vec{n}$ from $\vec{v}$ [S1, S2, S3, unofficial].
5. **Normalization:** Normalize the resulting vector to ensure it remains a unit direction vector:
   $$ \vec{v}_{new} = \frac{\vec{r}}{\|\vec{r}\|} $$
   [S3, unofficial]. Note: If $\vec{v}$ and $\vec{n}$ are already normalized and orthogonal components are handled correctly, the result should be normalized, but precision errors may require explicit normalization [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reflect_ray(incident_direction: Vector3, surface_normal: Vector3) -> Vector3:
    """
    Calculates the reflected ray direction.
    
    Args:
        incident_direction: The incoming ray direction (normalized).
        surface_normal: The surface normal at the hit point (normalized).
        
    Returns:
        The reflected ray direction (normalized).
    """
    # Step 1: Calculate dot product
    dot_product = dot(incident_direction, surface_normal)
    
    # Step 2: Calculate reflection vector
    # Formula: r = v - 2 * (v . n) * n
    reflection_vector = subtract(
        incident_direction,
        multiply_vector_scalar(surface_normal, 2.0 * dot_product)
    )
    
    # Step 3: Normalize the result
    # Although mathematically the result should be unit length if inputs are unit,
    # floating point errors may necessitate normalization.
    return normalize(reflection_vector)
```

**Parameters:**
- No specific parameters other than the input vectors.

**Edge Cases and Pitfalls:**
- **Precision Errors:** The result of the reflection formula should theoretically be a unit vector if inputs are unit vectors. However, floating-point arithmetic may introduce small errors. Explicit normalization is recommended [S3, unofficial].
- **Back-face Normals:** Ensure the normal points in the correct direction relative to the incident ray. If the ray is exiting a surface, the normal might need to be flipped depending on the shading model, though for pure reflection direction calculation, the geometric reflection holds regardless of side [S2, unofficial].
- **Degenerate Input:** If the incident ray is parallel to the surface ($\vec{v} \cdot \vec{n} = 0$), the reflection is the incident ray itself. If the incident ray is anti-parallel ($\vec{v} \cdot \vec{n} = -1$), the reflection is the reverse of the incident ray.

## Subtopics

### Refracted Ray Directions
Calculates the refracted direction of a ray given surface normals and refractive indices, handling entry and exit from spheres [S3, unofficial]. It differs from reflected directions by changing the ray's speed and direction based on Snell's Law [S3, unofficial]. Choose this when rendering transparent or translucent objects [S3, unofficial]. See [Ray Marching with Refraction and Beer's Law Absorption](1947-ray-marching-with-refraction-and-beer-s-law-absorption.md).

**Steps:**
1. Get surface normal $\vec{N}$ [S3, unofficial].
2. Set $n_1$ to current refractive index and $n_2$ to object's refractive index [S3, unofficial].
3. Calculate $s = \vec{ray} \cdot \vec{N}$ [S3, unofficial].
4. If object is a sphere and $s > 0$ (exiting sphere), swap $n_1$ and $n_2$ [S3, unofficial].
5. Calculate $n = n_1 / n_2$ [S3, unofficial].
6. Calculate refracted ray as $n \cdot (\vec{ray} - s \cdot \vec{N}) - \vec{N} \cdot \sqrt{1 - n^2 \cdot (1 - s^2)}$ [S3, unofficial].
7. Normalize the result and update ref to $n_2$ [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per reflection event. Requires a few dot products, multiplications, and one normalization.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Simple to implement, computationally inexpensive, provides physically accurate specular reflections for smooth surfaces.
- **Limitations:** Only handles surface reflections; does not account for diffuse interreflection or complex material properties without additional shading models. Does not handle refraction or absorption.
- **When to use:** When rendering glossy or specular surfaces in volume rendering or ray marching.
- **When not to use:** When rendering transparent objects (use refraction) or diffuse surfaces (use Lambertian shading).

## Background
Originated from basic geometric optics and vector algebra. The reflection formula $\vec{r} = \vec{v} - 2(\vec{v} \cdot \vec{n})\vec{n}$ is a standard result in computer graphics, widely used in ray tracing and ray marching implementations [S1, S3, unofficial]. It is a fundamental component of lighting models in volume rendering, particularly for simulating specular highlights on implicit surfaces defined by distance fields [S2, unofficial].

## Sources
[S1] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
[S2] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S3] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
