# Gradient-based Noise (Perlin/Simplex)
Also called: Gradient Noise, Perlin Noise, Simplex Noise

## Overview
Gradient-based noise generates smooth, continuous scalar fields by interpolating dot products between pseudo-random gradient vectors assigned to lattice points and distance vectors from those points to the query location [G]. It solves the problem of creating organic, non-periodic volumetric structures without the tiling artifacts or discrete steps of value noise [G]. It is distinguished from [Value Noise](4008-value-noise.md) by its use of gradient vectors rather than scalar values at lattice corners, which ensures $C^1$ continuity across cell boundaries [G].

## Shared principles
The family shares the following mechanism, terms, and math:

1.  **Gradient Assignment**: Each integer lattice point (vertex) in the domain is assigned a pseudo-random gradient vector $\mathbf{g}$. In 3D, common sets include 12 face diagonals of a cube or 12 axis-aligned vectors [G].
2.  **Dot Product Contribution**: The noise value at a query point $\mathbf{P}$ is derived from the dot product between the gradient vector $\mathbf{g}_i$ at a lattice corner $i$ and the distance vector $\mathbf{d}_i = \mathbf{P} - \mathbf{V}_i$ from that corner to $\mathbf{P}$:
    $$ n_i = \mathbf{g}_i \cdot \mathbf{d}_i $$
    [G]
3.  **Smoothing/Fade Function**: To ensure smooth transitions ($C^1$ continuity), the fractional coordinates of $\mathbf{P}$ within the cell are passed through a fade function $f(t)$ before interpolation. The quintic function $f(t) = 6t^5 - 15t^4 + 10t^3$ is standard for Perlin variants, while Simplex uses a different geometric approach [5415-perlin-gradient-and-interpolation-functions.md].
4.  **Interpolation**: The contributions from the cell corners are combined using linear interpolation (`lerp`) along the axes, weighted by the fade values [1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md].
5.  **Hashing**: Integer coordinates are mapped to gradient indices using a permutation table to avoid modulo operations and ensure uniform distribution [4211-perlin-noise-2d.md].

## Subtopics

### [Simplex Noise](978-simplex-noise.md)
Simplex Noise is a gradient-based procedural noise function that generates smooth, continuous scalar fields by evaluating dot products between input coordinates and pseudo-random gradient vectors assigned to simplex cell vertices. It solves the directional artifacts and computational inefficiency of cubic-grid Perlin noise by using a skewed simplex (triangular in 2D, tetrahedral in 3D) grid, which reduces the number of gradient evaluations and dot products required per sample. Unlike its sibling [Simplex Noise 2D](4155-simplex-noise-2d.md) which operates in 2D, this 3D variant specifically optimizes for volumetric terrain and texture generation by reducing the number of gradient evaluations and dot products required per sample.

### [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md)
3D Perlin-like Noise generates smooth, pseudo-random scalar values on a 3D unit lattice using gradient interpolation, while Fractal Brownian Motion (FBM) constructs multi-scale, self-similar detail by summing multiple octaves of this base noise with increasing frequency and decreasing amplitude. It solves the problem of creating organic, continuous volumetric structures (such as clouds, smoke, or terrain density) without the tiling artifacts or discrete steps of simpler noise methods. It is distinguished from [Simplex Noise](978-simplex-noise.md) by its use of a cubic lattice and trilinear interpolation rather than a simplex grid, and from [Improved Perlin Noise (4D)](3723-improved-perlin-noise-4d.md) by its specific 3D domain and standard gradient sets.

### [Improved Perlin Noise (4D)](3723-improved-perlin-noise-4d.md)
Improved Perlin Noise (4D) is a procedural noise function that generates smooth, continuous random values in four dimensions by interpolating dot products of gradient vectors at the corners of a 4D hypercube. It extends the 3D Improved Perlin Noise algorithm to support time-varying texture generation and 4D volume rendering, where the fourth dimension typically represents time or an additional spatial coordinate. Unlike its 3D sibling, it utilizes a 4D gradient texture and optimized 2D permutation tables to maintain performance on GPUs, avoiding branching and expensive modulo operations.

### [Perlin Noise 2D](4211-perlin-noise-2d.md)
Perlin Noise 2D is a gradient noise function that generates smooth, continuous random values by interpolating dot products of gradient vectors and distance vectors at the corners of the unit grid cell containing the query point. It solves the problem of discontinuous or visually artifact-prone noise by ensuring $C^1$ continuity across grid boundaries through a quintic fade function. It sets itself apart from its siblings, such as Simplex Noise, by operating on a square grid topology rather than a simplex lattice, and from 3D variants by restricting the domain to two dimensions.

### [Perlin Gradient and Interpolation Functions](5415-perlin-gradient-and-interpolation-functions.md)
This article defines the core mathematical primitives—gradient selection, fading, and linear interpolation—used to construct Perlin noise. It solves the problem of generating smooth, continuous scalar fields from discrete lattice points by ensuring $C^1$ continuity (continuous first derivatives) across cell boundaries. It differs from [Simplex Noise](978-simplex-noise.md) by using a hypercubic lattice and standard linear interpolation, and from [Improved Perlin Noise (4D)](3723-improved-perlin-noise-4d.md) by focusing on the fundamental 3D/2D building blocks rather than higher-dimensional optimizations.

### Gradient Displacement Noise
- **Definition**: Simulates fluid transport by displacing coordinates using per-octave gradient vectors derived from noise [S1].
- **Steps**: Loop from $i = 1.0$ to $7.0$. Compute gradient vector $\mathbf{gr}$ using `gradn` with time-modulated position [S1]. Scale gradient by a matrix generated by `makem2` with time-modulated parameters [S1]. Update position $\mathbf{p}$ by adding half the gradient [S1]. Accumulate result $rz$ using a sinusoidal function of noise at $\mathbf{p}$, divided by $z$ [S1]. Mix current position $\mathbf{p}$ with base position $\mathbf{bp}$ using factor $0.77$ [S1]. Specifically: 1. Compute gradient $\mathbf{gr} = \text{gradn}(i \cdot \mathbf{p} \cdot 0.34 + \text{time} \cdot 1.0)$ using finite differences of noise [S1]. 2. Rotate gradient using `makem2` [S1]. 3. Displace $\mathbf{p} += \mathbf{gr} \cdot \text{GRAD\_SCALE}$ [S1]. 4. Accumulate sinusoidal ridged noise [S1]. 5. Mix displaced $\mathbf{p}$ with base $\mathbf{bp}$ using `ADVECTION` [S1]. 6. Scale $z$, $\mathbf{p}$, and $\mathbf{bp}$ for next octave [S1].
- **Parameters**: $\text{FLOW\_SPEED}=0.6$, $\text{BASE\_SPEED}=1.9$, $\text{ADVECTION}=0.77$, $\text{GRAD\_SCALE}=0.5$ [S1].

### Hash-based Value Noise
- **Definition**: A simpler 3D noise function using 1D hash interpolation and 3D trilinear interpolation of hash values at the 8 corners of the unit cube [S2, unofficial].
- **Steps**: Define $\text{step} = \text{float3}(110, 241, 171)$ [S2, unofficial]. Compute integer part $\mathbf{i} = \lfloor \mathbf{x} \rfloor$ and fractional part $\mathbf{f} = \text{frac}(\mathbf{x})$ [S2, unofficial]. Compute smooth interpolation weights: $u = f \cdot f \cdot (3.0 - 2.0 \cdot f)$ [S2, unofficial]. Perform 3D trilinear interpolation of hash values at the 8 corners of the unit cube using $\text{hash}(n + \text{dot}(\text{step}, \text{float3}(x,y,z)))$ [S2, unofficial].

## Background
Not given in the sources. The provided evidence focuses on shader implementations for fluid simulation, domain warping, and isosurface generation [S1][S2, unofficial].

## Sources
[S1] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S2] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
