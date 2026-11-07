# Reflected Ray Directions

Also called: Reflection, reflect, Refract & Reflect Utilities, Snell's Law Implementation, Vector Reflection

## Overview
A method to determine the direction of a ray after it interacts with a surface during distance-field ray marching [S2, unofficial]. It calculates the new propagation direction based on the incident ray and the surface normal, enabling specular reflections in volume rendering [S3, unofficial]. It differs from [Ray Marching with Refraction and Beer's Law Absorption](1947-ray-marching-with-refraction-and-beer-s-law-absorption.md) by preserving the ray within the same medium, and from [Approximate Refraction](3072-approximate-refraction.md) by providing an exact geometric reflection rather than an approximation [G].

## How it works
**Inputs:**
- Incident ray direction $\vec{v}$ (normalized) [S3, unofficial].
- Surface normal $\vec{n}$ at the intersection point (normalized) [S3, unofficial].
- Optional distance field value $f(p)$ to determine if a hit occurred [S2, unofficial].

**Outputs:**
- Reflected ray direction $\vec{v}_{new}$ (normalized) [S3, unofficial].

**Data Structures:**
- Vectors $\vec{v}, \vec{n}$ are 3D floating-point vectors [G].

**Steps:**
1. **Surface Hit Detection:** Determine if the ray has hit a surface by evaluating if the distance field value $f(p_i) \approx 0$ [S2, unofficial]. If no surface is hit, the direction remains unchanged: $\vec{v}_{i+1} = \vec{v}_i$ [S2, unofficial].
2. **Normal Retrieval:** If a surface is hit, compute the surface normal $\vec{N}_i$ at the current position $p_i$ [S3, unofficial].
3. **Reflection Vector Calculation:** Apply the reflection formula to compute the unnormalized reflected direction $\vec{r} = \vec{v} - 2(\vec{v} \cdot \vec{n})\vec{n}$ [S1, unofficial][S3, unofficial]. This subtracts twice the projection of $\vec{v}$ onto $\vec{n}$ from $\vec{v}$ [S1, unofficial].
4. **Normalization:** Normalize the resulting vector to ensure it remains a unit direction vector: $\vec{v}_{new} = \frac{\vec{r}}{\|\vec{r}\|}$ [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reflect_ray(incident_direction: list[float], surface_normal: list[float]) -> list[float]:
    # Step 1: Calculate dot product v . n
    dot_product = sum(a * b for a, b in zip(incident_direction, surface_normal))
    # Step 2: Calculate reflection vector: r = v - 2 * (v . n) * n
    reflection_vector = [
        incident_direction[i] - 2.0 * dot_product * surface_normal[i]
        for i in range(3)
    ]
    # Step 3: Normalize the result
    magnitude = sum(x * x for x in reflection_vector) ** 0.5
    if magnitude == 0.0:
        return [0.0, 0.0, 0.0]
    return [x / magnitude for x in reflection_vector]
```

**Parameters:**
- No specific parameters other than the input vectors [S1, unofficial][S3, unofficial].

**Edge Cases and Pitfalls:**
- **Precision Errors:** Floating-point arithmetic may introduce small errors, making explicit normalization necessary [S3, unofficial].
- **Degenerate Input:** If the incident ray is parallel to the surface ($\vec{v} \cdot \vec{n} = 0$), the reflection equals the incident ray. If anti-parallel ($\vec{v} \cdot \vec{n} = -1$), the reflection reverses the incident ray [G].

## Subtopics

### Refracted Ray Directions
Calculates the refracted direction of a ray given surface normals and refractive indices, handling entry and exit from spheres [S3, unofficial]. It differs from reflected directions by applying Snell's Law to change the ray's direction based on the ratio of refractive indices [S3, unofficial]. See [Ray Marching with Refraction and Beer's Law Absorption](1947-ray-marching-with-refraction-and-beer-s-law-absorption.md) [G].

**Steps:**
1. Get surface normal $\vec{N}$ [S3, unofficial].
2. Set $n_1$ to current refractive index and $n_2$ to object's refractive index [S3, unofficial].
3. Calculate $s = \vec{ray} \cdot \vec{N}$ [S3, unofficial].
4. If object is a sphere and $s > 0$ (exiting sphere), swap $n_1$ and $n_2$ [S3, unofficial].
5. Calculate $n = n_1 / n_2$ [S3, unofficial].
6. Calculate refracted ray as $n \cdot (\vec{ray} - s \cdot \vec{N}) - \vec{N} \cdot \sqrt{1 - n^2 \cdot (1 - s^2)}$ [S3, unofficial].
7. Normalize the result and update ref to $n_2$ [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per reflection event, requiring a few dot products, multiplications, and one normalization [G].
- **Memory Complexity:** $O(1)$ auxiliary space [G].
- **Measured performance, scaling, strengths, limitations, and usage guidelines:** Not given in the sources [G].

## Background
Originated from basic geometric optics and vector algebra [G]. The reflection formula $\vec{r} = \vec{v} - 2(\vec{v} \cdot \vec{n})\vec{n}$ is a standard result in computer graphics, widely used in ray tracing and ray marching implementations [S1, unofficial][S3, unofficial]. It is a fundamental component of lighting models in volume rendering, particularly for simulating specular highlights on implicit surfaces defined by distance fields [S2, unofficial].

## Sources
[S1] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
[S2] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S3] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
