# Cosine-Weighted Hemisphere Sampling

Also called: Cosine Hemisphere Sampling, Lambertian Sampling

## Overview
Cosine-weighted hemisphere sampling generates a random direction on a hemisphere oriented around a surface normal, weighted by the cosine of the polar angle relative to that normal [S3, S5, S6, unofficial]. It is used to compute indirect lighting contributions for diffuse surfaces by integrating the rendering equation [S2, S5, unofficial]. Unlike uniform hemisphere sampling, it concentrates samples near the normal, reducing variance for diffuse shading [S5, unofficial]. It builds on [Random Hemisphere Sampling](2034-random-hemisphere-sampling.md) by applying a cosine-weighted distribution to the generated directions [S6, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- Two independent uniform random numbers $r_1, r_2 \in [0, 1)$ [S1, S2, S3, S6, unofficial].
- A unit surface normal vector $\mathbf{n}$ [S3, S5, S6, unofficial].

**Outputs:**
- A unit direction vector $\mathbf{d}$ in the local hemisphere defined by $\mathbf{n}$ [S3, S5, S6, unofficial].
- The probability density function (PDF) value for the generated direction [S2, unofficial].

### Data Structures
The method operates on scalar random numbers and 3D vectors. If using quasi-random sequences, a sequence generator state is maintained [S4, unofficial].

### Steps

1.  **Generate Uniform Random Numbers**: Obtain two independent uniform random numbers $r_1, r_2 \in [0, 1)$ [S1, S2, S3, S6, unofficial]. If using quasi-random sampling, generate the next 2D sample from the sequence and shift it to $[-0.5, 0.5)^2$ [S4, unofficial].
2.  **Compute Local Coordinates**: Calculate the radial distance $r = \sqrt{r_1}$ and azimuthal angle $\phi = 2\pi r_2$ [S1, S3, S6, unofficial]. (Note: S2 uses $\phi = 2\pi r_1$ and $\cos\theta = \sqrt{1-r_2}$, which is mathematically equivalent [S2, unofficial].)
3.  **Construct Local Direction**: Compute the components of the direction vector $\mathbf{d}_{local}$ in a tangent space where the $z$-axis aligns with $\mathbf{n}$:
    $$
    x = r \cos(\phi), \quad y = r \sin(\phi), \quad z = \sqrt{\max(0.0, 1.0 - r_1)}
    $$
    [S5, S6, unofficial]. Thus $\mathbf{d}_{local} = (x, y, z)$ [S6, unofficial].
4.  **Construct Local Basis**: Create an orthonormal basis $(\mathbf{e}_1, \mathbf{e}_2, \mathbf{e}_3)$ where $\mathbf{e}_3 = \mathbf{n}$ [S3, S5, unofficial]. To avoid degeneracy when $\mathbf{n}$ is parallel to a world axis, conditionally choose $\mathbf{e}_1$: if $|n_x| < \epsilon$, set $\mathbf{e}_1 = \text{normalize}((0, -n_z, n_y))$, else set $\mathbf{e}_1 = \text{normalize}((n_y, -n_x, 0))$ [S5, unofficial]. Compute $\mathbf{e}_2 = \mathbf{n} \times \mathbf{e}_1$ [S5, unofficial].
5.  **Transform to World Space**: Combine the local components with the basis vectors:
    $$
    \mathbf{d} = x \mathbf{e}_1 + y \mathbf{e}_2 + z \mathbf{n}
    $$
    [S3, S5, unofficial]. Some implementations explicitly normalize the result to handle floating-point drift [S1, unofficial].
6.  **Compute PDF**: The probability density function for the generated direction is:
    $$
    p(\mathbf{d}) = \frac{z}{\pi}
    $$
    [S2, unofficial]. Here $z$ is the local cosine of the polar angle $\theta$ between $\mathbf{d}$ and $\mathbf{n}$ [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cosine_weighted_hemisphere_sampling(r1: float, r2: float, normal: tuple) -> tuple:
    """
    Generates a cosine-weighted random direction on the hemisphere defined by normal.
    
    Args:
        r1: Uniform random number in [0, 1).
        r2: Uniform random number in [0, 1).
        normal: Unit normal vector (nx, ny, nz).
        
    Returns:
        direction: Unit direction vector (dx, dy, dz).
        pdf: Probability density function value.
    """
    # Step 2: Compute local coordinates
    r = math.sqrt(r1)
    phi = 2.0 * math.pi * r2
    
    # Step 3: Construct local direction components
    x = r * math.cos(phi)
    y = r * math.sin(phi)
    z = math.sqrt(max(0.0, 1.0 - r1))
    
    # Step 4: Construct local basis
    nx, ny, nz = normal
    
    # Avoid degeneracy when normal is parallel to world up (e.g., Z-axis)
    if abs(nx) < 1e-6:
        e1 = normalize((0.0, -nz, ny))
    else:
        e1 = normalize((ny, -nx, 0.0))
        
    e2 = cross(normal, e1)
    
    # Step 5: Transform to world space
    dx = x * e1[0] + y * e2[0] + z * nx
    dy = x * e1[1] + y * e2[1] + z * ny
    dz = x * e1[2] + y * e2[2] + z * nz
    direction = (dx, dy, dz)
    
    # Step 6: Compute PDF
    pdf = z / math.pi
    
    return direction, pdf

def normalize(v: tuple) -> tuple:
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return (v[0]/length, v[1]/length, v[2]/length) if length > 0 else (0.0, 0.0, 1.0)

def cross(a: tuple, b: tuple) -> tuple:
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])
```

### Parameters
- $r_1, r_2$: Uniform random numbers in $[0, 1)$ [S1, S2, S3, S6, unofficial].
- $n$: Number of samples to generate for quasi-random sequences [S4, unofficial].

### Edge Cases and Pitfalls
- **Degenerate Normal**: If $\mathbf{n}$ is parallel to a world axis, the cross product for basis construction can fail. A conditional branch checking the normal's components selects an appropriate reference axis to maintain orthogonality [S2, S5, unofficial].
- **Precision**: Compute $z$ as $\sqrt{\max(0.0, 1.0 - r_1)}$ to prevent domain errors when $r_1$ approaches 1.0 due to floating-point inaccuracies [S5, S6, unofficial].
- **PDF Evaluation**: The PDF must use the local $z$ component ($\cos \theta$), not world-space components, to correctly weight the Monte Carlo estimator [S2, unofficial].

## Operating characteristics

### Time and Memory Complexity
- **Time**: $O(1)$ per sample, requiring square roots, trigonometric functions, and vector operations [S2, S5, S6, unofficial].
- **Memory**: $O(1)$ auxiliary space per sample [S4, unofficial].

### Measured Performance
- Not given in the sources.

### Scaling
- Scales linearly with the number of samples $n$ [S4, unofficial].

### Strengths and Limitations
- **Strengths**: Concentrates samples near the normal, reducing variance for diffuse shading integration [S5, unofficial].
- **Limitations**: Not optimal for non-diffuse materials (e.g., glossy or specular BRDFs) [S5, unofficial]. Requires careful basis construction to avoid degeneracy [S2, S5, unofficial].

### When to Use
- Use for diffuse shading in path tracing or ray marching to compute indirect lighting [S2, S5, S6, unofficial].

### When Not to Use
- Not given in the sources.

## Background

### Origin
- The method is a standard technique in computer graphics for importance sampling the diffuse BRDF [S2, S5, S6, unofficial].

### History and Development
- Variations using quasi-random sequences (e.g., Halton) have been implemented to generate cosine-weighted samples [S4, unofficial].

### Where It Is Used
- Path tracing [S2, S5, unofficial].
- Ray marching [S5, unofficial].
- Voxel rendering [S6, unofficial].
- Ambient occlusion [S4, unofficial].

## Sources

[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S2] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S3] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S4] RvanderLaan/SVDAG-Compression, https://github.com/RvanderLaan/SVDAG-Compression, fetched 2026-10-26 (unofficial)
[S5] ollituohenmaa/raymarching-pathtracer, https://github.com/ollituohenmaa/raymarching-pathtracer, fetched 2026-10-26 (unofficial)
[S6] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
