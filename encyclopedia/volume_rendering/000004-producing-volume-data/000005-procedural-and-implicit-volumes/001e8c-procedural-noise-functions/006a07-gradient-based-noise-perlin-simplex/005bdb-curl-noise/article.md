# Curl Noise

Also called: Bridson Curl Noise

## Overview
Curl noise is a procedural noise function designed to generate time-varying, incompressible turbulent velocity fields [S2]. It solves the problem of creating visually plausible fluid-like motion without the computational cost of solving the Navier-Stokes equations [S2]. Unlike standard gradient noise which produces scalar fields, curl noise produces vector fields where the divergence is identically zero, ensuring the flow is solenoidal [S2].

## How it works
Curl noise constructs a divergence-free vector field by taking the curl of a vector potential derived from Perlin noise.

**Inputs and Outputs:**
- Input: Query point $\mathbf{P} = (x, y, z)$, time $t$, noise scale $s$.
- Output: Velocity vector $\mathbf{v} = (v_x, v_y, v_z)$.

**Data Structures:**
- Permutation table $P$: Array of size 512 (or 1024) containing integers $0..255$ to map coordinates to gradient indices [G].
- Gradient vectors $\mathbf{g}$: For 3D Perlin noise, typically 12 face diagonals of a cube: $(\pm 1, \pm 1, 0)$ and permutations [G].

**Steps:**
1.  **Generate Vector Potential:** Compute three independent Perlin noise values at $\mathbf{P}$ to form a vector potential $\mathbf{A} = (A_x, A_y, A_z)$.
    $$ A_x = \text{PerlinNoise}(x, y, z, t) $$
    $$ A_y = \text{PerlinNoise}(x + \text{offset}_x, y + \text{offset}_y, z + \text{offset}_z, t) $$
    $$ A_z = \text{PerlinNoise}(x + \text{offset}'_x, y + \text{offset}'_y, z + \text{offset}'_z, t) $$
    The offsets must be large enough to ensure statistical independence between $A_x, A_y, A_z$ but small enough to maintain spatial coherence. Common practice uses offsets like $(100, 200, 300)$ or distinct phase shifts [G].

2.  **Compute Curl:** Calculate the curl of $\mathbf{A}$ to obtain the velocity field $\mathbf{v}$.
    $$ \mathbf{v} = \nabla \times \mathbf{A} = \begin{pmatrix} \frac{\partial A_z}{\partial y} - \frac{\partial A_y}{\partial z} \\ \frac{\partial A_x}{\partial z} - \frac{\partial A_z}{\partial x} \\ \frac{\partial A_y}{\partial x} - \frac{\partial A_x}{\partial y} \end{pmatrix} $$

3.  **Numerical Derivatives:** Since Perlin noise is interpolated, derivatives are computed using the gradient contributions from the surrounding lattice points. For a scalar noise function $N(\mathbf{P})$, the partial derivative $\frac{\partial N}{\partial x}$ is computed by summing the dot products of the gradient vectors with the derivative of the fade function and distance vectors [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def curl_noise(p: Vec3, time: float, scale: float) -> Vec3:
    # Scale input for noise frequency
    s = p / scale
    
    # Generate three independent scalar noise fields for the vector potential A
    # Offsets ensure A_x, A_y, A_z are uncorrelated
    offset1 = Vec3(100.0, 200.0, 300.0)
    offset2 = Vec3(300.0, 100.0, 200.0)
    
    ax = perlin_noise(s, time)
    ay = perlin_noise(s + offset1, time)
    az = perlin_noise(s + offset2, time)
    
    # Compute derivatives of the scalar noise fields
    # dN/dx, dN/dy, dN/dz are computed via the gradient noise derivative logic
    dax_dx, dax_dy, dax_dz = perlin_gradient(s, time)
    day_dx, day_dy, day_dz = perlin_gradient(s + offset1, time)
    daz_dx, daz_dy, daz_dz = perlin_gradient(s + offset2, time)
    
    # Curl calculation: v = curl(A)
    vx = daz_dy - day_dz
    vy = dax_dz - daz_dx
    vz = day_dx - dax_dy
    
    return Vec3(vx, vy, vz)
```

**Edge Cases and Pitfalls:**
- **Correlation:** If offsets between the three noise components are too small, the resulting field may not be sufficiently turbulent or may exhibit artifacts. If too large, local coherence is lost [G].
- **Precision:** Computing derivatives of interpolated noise requires careful handling of the fade function derivatives to maintain $C^1$ continuity [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, proportional to the number of lattice points in the cell (8 for 3D Perlin) times the number of noise components (3).
- **Memory Complexity:** $O(1)$ for the permutation table and gradient set.
- **Strengths:** Produces incompressible flow naturally; visually pleasing turbulence; efficient to evaluate.
- **Limitations:** The flow is not physically accurate (does not conserve momentum or energy in a fluid dynamics sense); requires careful tuning of offsets and scale [S2].

## Background
Curl noise was presented by Robert Bridson, Ron Fedkiw, and Jos Stam in 2007 [S2]. It is based on the mathematical property that the curl of any vector field is divergence-free. It is widely used in computer graphics for simulating smoke, clouds, and other fluid-like phenomena where physical accuracy is less important than visual plausibility [S2].

## Subtopics

### [Simplex Noise](978-simplex-noise.md)
A gradient noise variant with reduced directional artifacts. Use when isotropic noise is required.

### [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md)
Combines multiple octaves of noise. Use for multi-scale detail.

### [Improved Perlin Noise (4D)](3723-improved-perlin-noise-4d.md)
Uses 4D noise for 3D spatial variation. Use for higher quality 3D textures.

### [Perlin Noise 2D](4211-perlin-noise-2d.md)
The 2D version of Perlin noise. Use for 2D textures.

### [Perlin Gradient and Interpolation Functions](5415-perlin-gradient-and-interpolation-functions.md)
Details on the fade and gradient functions. Use for implementation details.

## Sources
[S1] Part4 Chapter16 Fluid Simulation Demo, https://github.com/Razria93/Graphics_Study_Archive/issues/33, fetched 2026-10-26
[S2] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
