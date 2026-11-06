# Mandelbulb Fractal Generation

Also called: Mandelbulb

## Overview
Generates a 3D fractal by iteratively applying a polar coordinate power transformation to a complex 3D point [S1, unofficial]. It differs from its siblings, such as [Worley Noise](1726-worley-noise.md) or [Gradient-based Noise (Perlin/Simplex)](3860-gradient-based-noise-perlin-simplex.md), by producing a deterministic, self-similar geometric structure rather than a stochastic or continuous field, and from [Menger Sponge Fractal Generation](3550-menger-sponge-fractal-generation.md) by using continuous polar transformations instead of discrete axis-aligned box subtraction [S1, S2, unofficial].

## How it works
**Inputs:** Query point $P = (x, y, z)$, maximum iteration count $N_{max}$, escape radius $R_{max}$, and power exponent $E$.
**Outputs:** Boolean indicating if the point is inside the fractal (does not escape) or a distance estimate (if derivative tracking is added, though basic generation uses escape time).

**Data Structures:**
*   `point3d`: A structure or tuple containing three floating-point fields: `x`, `y`, `z`.
*   `float`: Standard 32-bit or 64-bit floating-point number.

**Steps:**
1.  **Initialization:**
    *   Set current point $z = P$.
    *   Set initial distance $z_{dist} = \|z\| = \sqrt{x^2 + y^2 + z^2}$.
    *   Set iteration counter $i = 0$.

2.  **Iteration Loop:**
    While $i < N_{max}$ and $z_{dist} \le R_{max}$:
    *   **Convert to Polar Coordinates:**
        *   Calculate radius $r = z_{dist}$.
        *   Calculate azimuthal angle $\theta = \arccos(z / r)$. Note: If $r=0$, $\theta$ is undefined; handle as singularity.
        *   Calculate polar angle $\phi = \arctan2(y, x)$.
    *   **Apply Power Transformation:**
        *   Update radius: $r_{new} = r^E$.
        *   Update angles: $\theta_{new} = \theta \times E$, $\phi_{new} = \phi \times E$.
    *   **Convert Back to Cartesian Coordinates:**
        *   $x_{new} = r_{new} \sin(\theta_{new}) \cos(\phi_{new})$
        *   $y_{new} = r_{new} \sin(\theta_{new}) \sin(\phi_{new})$
        *   $z_{new} = r_{new} \cos(\theta_{new})$
    *   **Update Point:**
        *   $z = (x_{new}, y_{new}, z_{new}) + P$ (Add original input point $P$ to the transformed point).
    *   **Update Distance:**
        *   $z_{dist} = \|z\|$.
    *   **Increment Counter:**
        *   $i = i + 1$.

3.  **Escape Check:**
    *   If $z_{dist} > R_{max}$ during the loop, the point escapes.
    *   If the loop completes all $N_{max}$ iterations without escaping, the point is considered part of the Mandelbulb.

**Parameters:**
*   $E = 8.0$ (Default power exponent [S1, unofficial]).
*   $N_{max} = 5$ (Default maximum iterations [S1, unofficial]).
*   $R_{max} = 20.0$ (Default escape radius [S1, unofficial]).

**Edge Cases and Pitfalls:**
*   **Singularity at Origin:** When $r=0$, $\theta$ is undefined. In practice, if the initial point $P$ is $(0,0,0)$, the fractal is centered there. Iterations from the origin remain at the origin if $P=(0,0,0)$, but typically $P$ is non-zero. If $z$ becomes zero during iteration, $\theta$ calculation must handle division by zero.
*   **Precision:** Floating-point precision limits the maximum useful iteration count. Higher powers $E$ cause faster divergence, requiring careful handling of large numbers.
*   **Boundary:** The escape radius $R_{max}$ determines the cutoff for "inside" vs "outside". Points with $\|z\| > R_{max}$ are outside.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def mandelbulb_iterate(p, power=8.0, max_iterations=5, max_distance=20.0):
    """
    Determines if point p is inside the Mandelbulb fractal.
    
    Args:
        p: tuple (x, y, z) representing the query point.
        power: The exponent E for the polar transformation.
        max_iterations: Maximum number of iterations N_max.
        max_distance: Escape radius R_max.
        
    Returns:
        bool: True if the point is inside (does not escape), False otherwise.
    """
    x, y, z = p
    z_dist = math.sqrt(x*x + y*y + z*z)
    
    for i in range(max_iterations):
        if z_dist > max_distance:
            return False
        
        # Convert to polar coordinates
        r = z_dist
        if r == 0:
            # Handle singularity; typically stays at origin or breaks
            # For Mandelbulb, if p is (0,0,0), it stays (0,0,0)
            # But if intermediate z becomes 0, we need to handle theta
            theta = 0.0 
            phi = 0.0
        else:
            theta = math.acos(z / r)
            phi = math.atan2(y, x)
        
        # Apply power transformation
        r_new = r ** power
        theta_new = theta * power
        phi_new = phi * power
        
        # Convert back to Cartesian
        x_new = r_new * math.sin(theta_new) * math.cos(phi_new)
        y_new = r_new * math.sin(theta_new) * math.sin(phi_new)
        z_new = r_new * math.cos(theta_new)
        
        # Update z = new_p + p_in
        x = x_new + p[0]
        y = y_new + p[1]
        z = z_new + p[2]
        
        # Update distance
        z_dist = math.sqrt(x*x + y*y + z*z)
        
    return True
```

## Subtopics

### [Bristorbrot Fractal Generation](3328-bristorbrot-fractal-generation.md)
Bristorbrot generates a 3D fractal using a specific iterative polynomial mapping in Cartesian coordinates. It differs from its parent, the Mandelbulb, which uses polar coordinate power transformations, by employing a distinct set of quadratic and cubic-like mixing terms that do not rely on spherical harmonics.

### [Menger Sponge Fractal Generation](3550-menger-sponge-fractal-generation.md)
Generates a 3D voxel grid or Signed Distance Field (SDF) representing a Menger sponge by iteratively dividing space into $3 \times 3 \times 3$ subcubes and removing those where two or more coordinates fall into the middle third. It differs from its parent [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md), which uses polar coordinate power transformations, by relying on discrete coordinate modulo arithmetic and axis-aligned box subtraction.

### Quaternion Fractal Generation
A fractal distance function based on quaternion iteration, similar to Mandelbrot sets but in 3D [S2, unofficial].
**Definition:** Uses quaternion algebra to iterate points in 4D space, projecting to 3D.
**Steps:**
1.  Initialize $v = (p_x, p_y, p_z, 0.0)$ and derivative $d = (1.0, 0.0, 0.0, 0.0)$.
2.  Loop $n$ from 1 to $ITERATIONS-1$:
    *   Update derivative: $d = 2.0 \times (v_x d_x - \text{dot}(v_{yzw}, d_{yzw}), v_x d_{yzw} + d_x v_{yzw} + \text{cross}(v_{yzw}, d_{yzw}))$.
    *   Update value: $v = (v_x^2 - \text{dot}(v_{yzw}, v_{yzw}), 2.0 v_x v_{yzw}) + c$.
3.  Check divergence: If $\text{dot}(v, v) > 10.0$, break.
**Formula:** Final distance estimate: $0.5 \times r \times \log(r) / \|d\|$, where $r = \|v\|$.
**Parameters:** $ITERATIONS = 5$, $c = (0.18, 0.88, 0.24, 0.16)$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_{max})$ per voxel, where $N_{max}$ is the maximum iteration count. Each iteration involves trigonometric functions and vector operations.
*   **Memory Complexity:** $O(1)$ auxiliary space per voxel query.
*   **Performance:** The use of trigonometric functions (`sin`, `cos`, `acos`, `atan2`) makes each iteration computationally expensive compared to polynomial-only fractals. Typical default parameters ($N_{max}=5, E=8$) are relatively fast, but increasing iterations or power significantly increases render time.
*   **Strengths:** Produces highly detailed, organic-looking 3D fractal structures with self-similarity.
*   **Limitations:** High computational cost per voxel. The escape radius and power parameter heavily influence the visual result and convergence speed. Not suitable for real-time rendering without optimization (e.g., GPU acceleration, lower iteration counts).
*   **When to use:** When generating static 3D fractal models or high-quality offline renders.
*   **When not to use:** When real-time performance is critical and simpler noise functions (like [Worley Noise](1726-worley-noise.md)) suffice.

## Background
The Mandelbulb is a 3D fractal discovered in 2009 by Daniel Shapira and others, inspired by the 2D Mandelbrot set. It extends the concept of complex number iteration to 3D using spherical coordinates. The method described here follows the implementation details found in procedural fractal generation code [S1, unofficial]. It is used in computer graphics for generating complex, natural-looking 3D shapes and textures.

## Sources
[S1] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
[S2] nopjia/WebGL-RayMarch, https://github.com/nopjia/WebGL-RayMarch, fetched 2026-10-26 (unofficial)
