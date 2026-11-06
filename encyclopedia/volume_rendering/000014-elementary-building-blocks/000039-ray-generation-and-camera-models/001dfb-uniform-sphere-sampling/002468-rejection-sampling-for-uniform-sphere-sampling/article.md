# Rejection Sampling for Uniform Sphere Sampling

Also called: Cube Rejection Sampling, Random Sampling Utilities, Random Unit Vector Generation, Rejection Sampling for Ball, Rejection Sampling for Uniform Sphere/Disk Sampling, Uniform Ball Sampling, Unit Disk Sampling, Unit Sphere Sampling

## Overview
Rejection sampling generates uniformly distributed points within a unit ball (or on its surface) by sampling from a bounding cube and discarding outliers. It differs from spherical coordinate conversion [Uniform Sphere Sampling](1669-uniform-sphere-sampling.md) by avoiding trigonometric functions, relying instead on simple arithmetic and normalization, which makes it computationally cheaper per sample but potentially slower in wall-clock time due to rejected iterations.

## How it works
**Inputs:** A source of independent uniform random numbers $u \in [0, 1)$.
**Outputs:** A vector $\mathbf{p}$ uniformly distributed in the unit ball ($|\mathbf{p}| \le 1$) or on the unit sphere surface ($|\mathbf{p}| = 1$).

**Data Structures:**
*   `vec3`: A 3D vector with components $(x, y, z)$.
*   `float`: 32-bit or 64-bit floating point number.

**Algorithm for Unit Ball (Solid):**
1.  Generate three independent random values $x, y, z$ uniformly in $[-1, 1]$.
2.  Compute squared length $L = x^2 + y^2 + z^2$.
3.  If $L > 1$, reject and return to step 1.
4.  Return $\mathbf{p} = (x, y, z)$.

**Algorithm for Unit Sphere (Surface):**
1.  Generate three independent random values $x, y, z$ uniformly in $[-1, 1]$.
2.  Compute squared length $L = x^2 + y^2 + z^2$.
3.  If $L \ge 1$ or $L < \epsilon$, reject and return to step 1.
4.  Normalize: $\mathbf{p} = \frac{1}{\sqrt{L}}(x, y, z)$.
5.  Return $\mathbf{p}$.

**Parameters:**
*   $\epsilon$: Lower bound for squared length to prevent division by zero. Source [S1, unofficial] uses $10^{-160}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def sample_unit_ball():
    """Generate a point uniformly distributed inside the unit ball."""
    while True:
        # Generate x, y, z in [-1, 1]
        x = random.uniform(-1.0, 1.0)
        y = random.uniform(-1.0, 1.0)
        z = random.uniform(-1.0, 1.0)
        
        # Calculate squared length
        lensq = x*x + y*y + z*z
        
        # Check if inside unit sphere
        if lensq <= 1.0:
            return (x, y, z)

def sample_unit_sphere():
    """Generate a point uniformly distributed on the unit sphere surface."""
    while True:
        # Generate x, y, z in [-1, 1]
        x = random.uniform(-1.0, 1.0)
        y = random.uniform(-1.0, 1.0)
        z = random.uniform(-1.0, 1.0)
        
        # Calculate squared length
        lensq = x*x + y*y + z*z
        
        # Check bounds: must be strictly inside to normalize, 
        # and not too close to zero to avoid division by zero.
        # Source [S1, unofficial] uses 1e-160 as lower bound.
        if 1e-160 < lensq < 1.0:
            # Normalize
            inv_sqrt = 1.0 / math.sqrt(lensq)
            return (x * inv_sqrt, y * inv_sqrt, z * inv_sqrt)
```

**Subtopics:**
*   [Random Hemisphere Sampling](2034-random-hemisphere-sampling.md): Samples uniformly on a hemisphere. Choose when surface normals are constrained to one side.
*   [Spherical Background Sampling](3991-spherical-background-sampling.md): Samples directions for environment maps. Choose when sampling sky/ambient lighting.

## Operating characteristics
*   **Time Complexity:** Expected $O(1)$ iterations. The volume of the unit ball is $\frac{4}{3}\pi \approx 4.19$ and the cube is $2^3=8$. The acceptance rate for the ball is $\frac{4.19}{8} \approx 52\%$. For the sphere surface, the acceptance rate is effectively 100% once inside the ball, provided the rejection of $L \ge 1$ is handled. However, the normalization step adds cost.
*   **Memory Complexity:** $O(1)$.
*   **Strengths:** No trigonometric functions; simple to implement; parallelizable.
*   **Limitations:** Rejection loop can be unpredictable; less efficient than direct methods for high-dimensional spheres.
*   **When to use:** When simplicity and speed of implementation are prioritized, or when generating points for volumetric density sampling where the ball distribution is acceptable.

## Background
Originating from basic Monte Carlo integration techniques, this method is a standard primitive in computer graphics for generating random directions. It is widely used in path tracers and ray marchers due to its robustness and lack of singularities compared to spherical coordinate sampling.

## Sources
[S1] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S2] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
[S3] ollituohenmaa/raymarching-pathtracer, https://github.com/ollituohenmaa/raymarching-pathtracer, fetched 2026-10-26 (unofficial)
