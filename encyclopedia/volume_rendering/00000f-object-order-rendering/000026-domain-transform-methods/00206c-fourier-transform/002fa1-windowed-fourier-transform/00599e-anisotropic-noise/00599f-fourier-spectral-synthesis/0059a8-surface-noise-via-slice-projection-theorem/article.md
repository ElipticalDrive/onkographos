# Surface Noise via Slice-Projection Theorem
## Overview
Surface noise via the slice-projection theorem is a method to generate band-pass noise on a curved surface by integrating a 3D band-pass solid noise field perpendicular to that surface [S1]. It solves the problem that naively sampling (slicing) a 3D band-pass noise function along a 2D surface results in a non-band-pass power spectrum, causing visual artifacts such as low-frequency drift or aliasing [S1]. Unlike solid texturing, which creates the illusion of an object sculpted from solid matter, this method allows anisotropic textures to flow around the object's topology, adapting to curvature changes [S1]. It differs from planar parametrization by avoiding the distortions and seams inherent in mapping 2D noise directly onto complex surfaces [S1].

## How it works
**Inputs:**
1. A 3D band-pass noise function $N(x, y, z)$ with a known band-pass power spectrum $P_{3D}(u, v, w)$ [S1].
2. A surface defined by a position function $\mathbf{S}(u, v)$ and a unit normal vector field $\mathbf{n}(u, v)$ [S1].

**Outputs:**
1. A 2D noise field $N_{surf}(u, v)$ defined on the surface parameter domain, which exhibits band-pass characteristics [S1].

**Data Structures:**
- The 3D noise function $N$ is typically implemented as a procedural function (e.g., value noise, gradient noise) or a sparse convolution grid. No specific pre-computed table layout is mandated by the theorem itself, though explicit noises may rely on pre-computed tables [S1].
- The surface is represented by its parametric coordinates $(u, v)$, position $\mathbf{S}$, and normal $\mathbf{n}$.

**Method Steps:**
1. **Identify the Problem:** Recognize that evaluating $N$ at $\mathbf{S}(u, v)$ corresponds to "slicing" the 3D domain. According to the slice-projection theorem, slicing in the spatial domain corresponds to projection (integration) in the frequency domain [S1]. Integrating a band-pass spectrum results in a non-band-pass spectrum (typically accumulating energy at low frequencies) [S1].
2. **Apply Projection (Integration):** To restore the band-pass property, integrate the solid noise field along the direction perpendicular to the surface (along the normal $\mathbf{n}$) [S1]. This operation corresponds to "slicing" the frequency spectrum, which preserves the band-pass nature [S1].
3. **Compute Surface Noise:** For each point on the surface, compute the integral of the 3D noise function along the normal vector. The value $N_{surf}(u, v)$ is obtained by integrating $N$ along the line segment defined by the normal at $\mathbf{S}(u, v)$ [S1].

**Formulas:**
The slice-projection theorem states that the Fourier transform of a slice of a function is a projection of its Fourier transform [S1].
Let $P_{3D}(u, v, w)$ be the power spectrum of the 3D noise.
Slicing the spatial domain (evaluating on surface) $\implies$ Projecting the frequency domain (integration of spectrum).
To get band-pass noise on the surface, we must perform the inverse operation in the spatial domain:
$$ N_{surf}(u, v) = \int_{-\infty}^{\infty} N(\mathbf{S}(u, v) + t \cdot \mathbf{n}(u, v)) \, dt $$
In practice, this integral is approximated over a finite range or using a specific kernel that mimics the integration effect, ensuring the resulting power spectrum remains band-pass [S1].

**Parameters:**
- Integration range: Not given in the sources.
- Kernel width: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** The integration must be performed accurately to maintain the band-pass property. Numerical errors in integration may introduce low-frequency artifacts [S1].
- **Boundary Conditions:** The method assumes an infinite or periodic domain for the 3D noise. If the noise is defined on a finite domain, boundary effects may distort the spectrum [S1].
- **Degenerate Input:** If the surface normal is undefined (e.g., at singularities), the integration direction is ambiguous [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_noise_via_slice_projection(surface_position, surface_normal, noise_3d):
    """
    Computes surface noise by integrating 3D noise along the surface normal.
    
    Args:
        surface_position: (u, v) -> (x, y, z) function returning 3D position.
        surface_normal: (u, v) -> (nx, ny, nz) function returning unit normal.
        noise_3d: (x, y, z) -> float function returning 3D noise value.
        
    Returns:
        (u, v) -> float function returning surface noise value.
    """
    def compute_noise(u, v):
        # Get position and normal at the surface point
        pos = surface_position(u, v)
        nrm = surface_normal(u, v)
        
        # Integrate noise along the normal direction
        # Note: The exact integration range and method (e.g., quadrature) 
        # are not specified in the sources [S1]. 
        # A simple approximation might use a fixed range [-L, L] and 
        # sample multiple points along the normal.
        
        integration_range = 1.0  # Not given in the sources [G]
        num_samples = 10         # Not given in the sources [G]
        
        integral_sum = 0.0
        for i in range(num_samples):
            # Sample point along the normal
            t = -integration_range + (2.0 * integration_range * i) / (num_samples - 1)
            sample_point = pos + t * nrm
            integral_sum += noise_3d(sample_point.x, sample_point.y, sample_point.z)
            
        # Average the samples to approximate the integral
        # The scaling factor depends on the integration method [G]
        return integral_sum / num_samples

    return compute_noise
```

## Subtopics

### Surface Noise via Planar Parametrization
This variant maps 2D noise onto surfaces using planar parametrization, similar to regular texture maps [S1]. It compensates for mapping distortions by locally adapting the noise content at every pixel, estimating local distortion and its impact on the noise spectrum [S1]. The weights of summed noise bands are updated to approximate a pre-distorted spectrum, generating noise with inversely pre-distorted frequency content [S1]. This approach can be performed efficiently in a GPU shader [S1]. However, it can introduce distortions and seams, breaking properties like uniform frequency content, continuity, and band-pass nature [S1]. It only recovers from distortions and cannot hide seams [S1]. This idea was also suggested by van Wijk [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Produces band-pass noise on surfaces, allowing anisotropic textures to flow around objects and adapt to topology changes [S1]. Avoids the "sculpted from solid matter" look of solid texturing, which is desirable for fiber-like materials [S1].
- **Limitations:** Requires integration of 3D noise, which can be computationally expensive if not optimized [S1]. The method relies on the slice-projection theorem, which assumes ideal conditions [S1].
- **When to Use:** When anisotropic textures need to flow around complex curved surfaces without distortion [S1].
- **When Not to Use:** When the "solid texturing" effect is desired, or when computational resources are limited and integration is too costly [S1].

## Background
The method is based on the slice-projection theorem, which states that slicing in one domain corresponds to projection or integration in the other [S1]. Cook and DeRose [CD05] observed that sampling solid band-pass noise along a surface does not result in band-pass noise on the surface, and proposed using the slice-projection theorem to solve this by integrating the noise perpendicular to the surface [S1]. The theorem is attributed to [Bra99, Mal93] [S1]. This approach contrasts with solid texturing, introduced by Perlin [Per85] and Peachy [Pea85], which samples 3D noise directly on the surface [S1].

## Sources
[S1] A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
