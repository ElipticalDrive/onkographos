# Fractals and Mathematical Surfaces


## Overview
This family covers rendering techniques for procedural surfaces defined by mathematical functions, specifically focusing on fractals (such as Mandelbrot and Julia sets) and scale-invariant integral surfaces. It addresses the challenge of rendering complex, infinitely detailed, or computationally expensive implicit surfaces without relying on brute-force sampling. It differs from its siblings by focusing on specific mathematical structures: [Scale-Invariant Integral Surface Rendering](4266-scale-invariant-integral-surface-rendering.md) optimizes field evaluation for skeleton-based surfaces, while fractal variants rely on escape-time algorithms or deterministic ray-tracing for self-similar structures.

## Shared principles
The family operates on the principle that a scene is defined by a scalar field $f(\mathbf{p})$ where the zero-level set $f(\mathbf{p}) = 0$ represents the surface [G]. For sphere tracing to be valid, the function must satisfy the Lipschitz condition with constant 1, ensuring $|f(\mathbf{p}_1) - f(\mathbf{p}_2)| \le \|\mathbf{p}_1 - \mathbf{p}_2\|$ [G]. This guarantees that the value $f(\mathbf{p})$ is a safe lower bound on the distance to the surface, allowing the ray marcher to step forward by exactly $f(\mathbf{p})$ without tunneling [G].

Composition is achieved through Boolean operations on the distance values:
*   **Union**: $f_{union}(\mathbf{p}) = \min(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [G].
*   **Intersection**: $f_{intersect}(\mathbf{p}) = \max(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [G].
*   **Subtraction**: $f_{sub}(\mathbf{p}) = \max(-f_1(\mathbf{p}), f_2(\mathbf{p}))$ [G].

Transformations are applied by mapping the query point $\mathbf{p}$ into the local coordinate space of the primitive using the inverse transformation matrix $M^{-1}$ before evaluation [G]. Procedural noise (e.g., FBM) is often used to perturb these analytical surfaces, creating hypertextures where the isosurface is defined by $SDF(\mathbf{p}) + B_n(\mathbf{p}) = 0$ [S3, unofficial].

## Subtopics

### [Scale-Invariant Integral Surface Rendering](4266-scale-invariant-integral-surface-rendering.md)
[Scale-Invariant Integral Surface Rendering](4266-scale-invariant-integral-surface-rendering.md) is a rendering algorithm for scale-invariant integral surfaces that reduces computation time by up to 70% compared to revised previous work. It addresses the excessive step counts of sphere-tracing along grazing rays by employing an initial sampling strategy based on homothetic distance to segment primitives, combined with a field mapping using quadratic polynomial interpolation. Unlike sphere-tracing, which suffers from a well-known problem of maximal number of steps along grazing rays, SCALIS provides a reduction in the maximal number of steps required.

### Ray Subdivision with Polynomial Interpolation for SCALIS Fields
- **Definition**: A ray-tracing method for Scale-Invariant Integral Surfaces (SCALIS) that avoids expensive brute-force field evaluations by subdividing the ray into intervals based on local minima of homothetic distance fields, then using iterative quadratic polynomial interpolation to find isosurface crossings [S1].
- **Step**: Map the SCALIS field value $f$ to a normalized field $g(f)$ using the inverse of the primitive field function followed by squaring, effectively approximating the squared homothetic distance [S1].
- **Formula**: The normalized field $g(f)$ for an infinite line primitive with radius $\tau_0$ is defined as: $g(f) = \left( f_{line,1}^{-1}(f) \right)^2 = \left( \tilde{k}^{-1} \left( \frac{f}{\tilde{\lambda}} \right) \right)^2 = \left( \frac{d}{\tau_0} \right)^2$, where $\lambda = c/\tilde{k}(1)$ [S1].

### Ray Tracing Deterministic 3-D Fractals
- **Definition**: A ray-tracing algorithm applied to 3-D deterministic fractals (such as Julia sets of quadratic functions in higher dimensions) that uses the storage facilities of a graphics workstation frame buffer, replacing the boundary-tracking algorithm which required large amounts of storage [S2, unofficial].
- **Step**: Apply a ray-tracing algorithm to 3-D deterministic fractals [S2, unofficial].
- **Comparison**: Requires only frame buffer storage, whereas the original boundary-tracking algorithm required a large amount of storage space [S2, unofficial].

### Mandelbrot Set
- **Definition**: A fractal defined by an escape-time algorithm that renders a zoomable set with infinite detail at its boundary [S3, unofficial].
- **Parameter**: Iteration count: Increasing iterations from 128 to 512 reveals finer filaments but demands optimized early-exit logic [S3, unofficial].
- **Pitfall**: Higher iteration counts require optimized early-exit logic to maintain performance [S3, unofficial].

### Julia Set Fractal Generation
- **Definition**: A fractal offering complementary dynamics to the Mandelbrot set, where varying the constant parameter across animation frames creates morphing landscapes [S3, unofficial].

## Background
The rendering of mathematical surfaces has evolved from boundary-tracking algorithms, which required significant storage, to ray-tracing methods that leverage frame buffer storage [S2, unofficial]. Fractals, such as the Mandelbrot set popularized by Benoit Mandelbrot in 1980, remain a direct expression of mathematical art in procedural graphics [S3, unofficial]. Modern implementations often combine distance-estimated 3D fractals with ray marching to achieve high performance, such as 60 fps at 4K resolution [S3, unofficial]. Scale-Invariant Integral Surfaces (SCALIS) were developed to handle complex skeleton-based structures efficiently, addressing the computational costs of brute-force ray marching [S1].

## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S2] (PDF) Accelerating Sphere Tracing - ResearchGate, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S3] Procedural Graphics: Noise, Fractals and Mathematical Art, https://www.mattcurrent.org/procedural-graphics-guide/, fetched 2026-10-27 (unofficial)
