# Random Hemisphere Sampling

Also called: GenerateSampleVector, Hemisphere Sampling for Ambient Gathering, Lambertian Material Scattering, Random Hemisphere Vector, Uniform Hemisphere Sampling, hemisphere random direction, sampleHemisphere

## Overview
Random Hemisphere Sampling generates a random direction uniformly distributed over the upper hemisphere oriented around a given surface normal vector $\mathbf{n}$ [S2, S4, unofficial]. It solves the problem of scattering rays diffusely from a surface or volume boundary, ensuring that all generated directions point into the visible half-space defined by the normal [S1, unofficial]. It sets itself apart from its parent [Uniform Sphere Sampling](1669-uniform-sphere-sampling.md) by restricting the domain to a hemisphere, and from [Cosine-Weighted Hemisphere Sampling](2283-cosine-weighted-hemisphere-sampling.md) by maintaining a uniform probability density function (PDF) rather than biasing samples toward the normal [S2, S4, unofficial].

## How it works
**Inputs:**
*   $\mathbf{n}$: The surface normal vector (unit length) [S2, S4, unofficial].
*   $r_1, r_2$: Two independent uniform random numbers in the range $[0, 1)$ [S2, unofficial].
*   $i$: An integer index for hash-based generation (optional) [S4, unofficial].

**Outputs:**
*   $\mathbf{d}$: A unit vector representing the sampled direction on the upper hemisphere relative to $\mathbf{n}$ [S2, S4, unofficial].

**Data Structures:**
*   The method operates on 3D vectors, scalar random values, and optionally a hash function [S2, S4, unofficial].

**Steps:**
1.  Generate two independent uniform random numbers $r_1, r_2 \in [0, 1)$ [S2, unofficial].
2.  Compute the polar component radius: $r = \sqrt{1.0 - r_1^2}$ [S2, unofficial].
3.  Compute the azimuthal angle: $\phi = 2\pi r_2$ [S2, unofficial].
4.  Construct the direction in a local coordinate system where $\mathbf{n}$ aligns with the Z-axis: $\mathbf{d}_{\text{local}} = (\cos\phi \cdot r, \sin\phi \cdot r, r_1)$ [S2, unofficial].
5.  Rotate $\mathbf{d}_{\text{local}}$ to align with the world-space normal $\mathbf{n}$ by constructing an orthonormal basis $\{\mathbf{T}, \mathbf{B}, \mathbf{N}\}$ where $\mathbf{N}=\mathbf{n}$, and computing $\mathbf{d} = \mathbf{T}(\cos\phi \cdot r) + \mathbf{B}(\sin\phi \cdot r) + \mathbf{N}(r_1)$ [G].

*Alternative Rejection/Reflection Approach:* Generate a random unit vector $\mathbf{v}$. If $\mathbf{v} \cdot \mathbf{n} > 0$, return $\mathbf{v}$; otherwise return $-\mathbf{v}$ [S1, unofficial]. Alternatively, compute $\mathbf{v} = \text{normalize}(\text{Hash33}(\mathbf{n} + i) - \mathbf{0.5})$ and reflect it across the plane defined by $\mathbf{n}$ [S4, unofficial].

**PDF Calculation:**
The standard PDF for uniform hemisphere sampling is $p(\omega) = \frac{1}{2\pi}$ [G]. One implementation calculates it as $\frac{1}{4\pi}$ for specific configuration contexts [S3, unofficial].

**Parameters:**
*   `config::hemisphere_u`: Number of U samples for hemisphere sampling [S3, unofficial].
*   `config::hemisphere_t`: Number of T samples for hemisphere sampling [S3, unofficial].
*   `config::ambience`: Enum controlling ambient shading mode; `AMBIENT_FULL` triggers hemisphere sampling [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-Intersection:** The ray origin must be offset slightly along the normal to prevent self-intersection or precision errors, e.g., `hit.point + sample * 0.0` [S3, S5, unofficial].
*   **Degenerate Normals:** When $|n_z| \ge 0.999$, the cross product for the tangent basis becomes degenerate; the code must switch to a reference vector like $\{1,0,0\}$ [S7, unofficial].
*   **Zero Vector:** If the computed scatter direction magnitude is $< 10^{-8}$, reset it to the normal vector [S1, unofficial].

**Pseudocode (Cylindrical Method):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vector3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

def cross_product(v1, v2):
    return Vector3(
        v1.y * v2.z - v1.z * v2.y,
        v1.z * v2.x - v1.x * v2.z,
        v1.x * v2.y - v1.y * v2.x
    )

def normalize(v):
    length = math.sqrt(v.x**2 + v.y**2 + v.z**2)
    if length == 0:
        return Vector3(0, 0, 1)
    return Vector3(v.x / length, v.y / length, v.z / length)

def random_hemisphere_sampling(normal, r1, r2):
    """
    Generates a random direction uniformly distributed on the hemisphere
    defined by 'normal' using cylindrical coordinate conversion [S2, unofficial].
    """
    # 1. Compute local coordinates
    r = math.sqrt(1.0 - r1 * r1)
    phi = 2.0 * math.pi * r2
    
    local_x = math.cos(phi) * r
    local_y = math.sin(phi) * r
    local_z = r1
    
    # 2. Construct orthonormal basis (T, B, N)
    n = normal
    
    # Handle degenerate case where normal is parallel to world Z [S7, unofficial]
    if abs(n.z) < 0.999:
        t = cross_product(Vector3(0, 0, 1), n)
    else:
        t = cross_product(Vector3(1, 0, 0), n)
        
    t = normalize(t)
    b = cross_product(n, t)
    
    # 3. Transform local direction to world space [G]
    world_x = t.x * local_x + b.x * local_y + n.x * local_z
    world_y = t.y * local_x + b.y * local_y + n.y * local_z
    world_z = t.z * local_x + b.z * local_y + n.z * local_z
    
    return Vector3(world_x, world_y, world_z)
```

## Subtopics

### [Cosine-Weighted Hemisphere Sampling](2283-cosine-weighted-hemisphere-sampling.md)
Cosine-Weighted Hemisphere Sampling generates a random direction on a hemisphere oriented around a normal vector $\mathbf{n}$, weighted by the cosine of the polar angle. It is used to compute the indirect lighting contribution at a surface point by sampling the hemisphere above the surface normal and recursively tracing rays. It sets itself apart from its parent [Random Hemisphere Sampling](2034-random-hemisphere-sampling.md) by biasing samples toward the normal, which matches the cosine lobe of a Lambertian BRDF.

### [Quadratic Scale Hemisphere Sampling](3988-quadratic-scale-hemisphere-sampling.md)
Quadratic Scale Hemisphere Sampling generates a set of 3D sample vectors for screen-space ambient occlusion (SSAO) by mapping a 3D Halton sequence to a hemisphere and applying a quadratic scale distribution. It differs from its parent, [Random Hemisphere Sampling](2034-random-hemisphere-sampling.md), by using a deterministic low-discrepancy sequence (Halton) rather than independent random numbers, and from its sibling, [Cosine-Weighted Hemisphere Sampling](2283-cosine-weighted-hemisphere-sampling.md), by applying a specific quadratic bias to the sample radius to concentrate samples near the origin while maintaining uniform angular coverage.

### Hemisphere Assembling with Normal Modulation
A technique to create a hemisphere of sampling directions on a surface by modulating precomputed random sphere points with the surface normal to reduce banding artifacts from discrete points [S5, unofficial]. It works by reflecting the surface normal against a random point on a sphere to get a random point on a sphere with variance added by the normal vector [S5, unofficial]. It checks if the resulting cone direction is below the horizon by computing the dot product of the surface normal and the cone direction; if negative, it flips the cone direction [S5, unofficial].

### Uniform Hemisphere Solid Angle Sampling
This variant calls `getSampleUniformSolidAngle` to generate a direction on the full sphere [S6, unofficial]. If the dot product of the sample direction and the normal is less than 0 (pointing below the hemisphere), it negates the sample direction to flip it to the upper hemisphere [S6, unofficial]. It returns the resulting hemisphere direction [S6, unofficial].

### Temporal Hemisphere Sampling for Diffuse GI
A technique to sample the upper hemisphere for diffuse global illumination using two temporal coordinates (`temporalx`, `temporaly`) to generate a sample direction, enabling accurate lighting with few samples per frame via temporal anti-aliasing [S7, unofficial]. The sample direction $\omega_i$ is obtained as $\omega_i = \{T, B, N\} \cdot H$, where $H = \{\cos\phi\sin\theta, \sin\phi\sin\theta, \cos\theta\}$, $\phi = 2\pi \cdot \text{temporalx}$, and $\theta = 0.5\pi \cdot \text{temporaly}$ [S7, unofficial]. The tangent $T$ is computed as $\{0,0,1\} \times N$ if $|n_z| < 0.999$, otherwise $\{1,0,0\} \times N$, and $B = N \times T$ [S7, unofficial]. A pitfall occurs when $|n_z| \ge 0.999$, where the cross product becomes degenerate, handled by switching the reference vector [S7, unofficial]. Using only one sample per frame creates a checkerboard pattern without temporal AA, but with temporal AA enabled, it produces an accurate image within a few frames via Monte Carlo integration over time [S7, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample [G].
*   **Memory Complexity:** $O(1)$ [G].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Linear with the number of samples [G].
*   **Strengths:** Simple to implement; provides uniform coverage of the hemisphere [G].
*   **Limitations:** High variance compared to importance sampling methods for Lambertian surfaces [G].
*   **When to use:** When uniform distribution is required, or as a base for other sampling techniques [G].
*   **When not to use:** When computing diffuse lighting on Lambertian surfaces, where cosine-weighted sampling is more efficient [G].

## Background
Random Hemisphere Sampling is a fundamental technique in computer graphics for diffuse shading and global illumination. It is widely used in path tracing, voxel-based global illumination, and real-time rendering pipelines. The method is often implemented using either cylindrical coordinate conversion or reflection-based rejection. Variants such as temporal hemisphere sampling have been developed for real-time applications to reduce noise with limited samples per frame [S7, unofficial].

## Sources
[S1] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S2] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
[S3] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S4] cahekp/Raymarching, https://github.com/cahekp/Raymarching, fetched 2026-10-27 (unofficial)
[S5] Voxel-based Global Illumination – Wicked Engine, https://turanszkij.wordpress.com/2017/08/30/voxel-based-global-illumination/, fetched 2026-10-28 (unofficial)
[S6] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S7] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
