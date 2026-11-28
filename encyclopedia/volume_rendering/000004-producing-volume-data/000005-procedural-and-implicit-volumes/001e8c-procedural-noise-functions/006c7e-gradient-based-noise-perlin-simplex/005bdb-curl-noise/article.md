# Curl Noise

Also called: Bridson Curl Noise

## Overview
Curl noise is a procedural noise function designed to generate time-varying, incompressible turbulent velocity fields [S2, S3]. It solves the problem of creating visually plausible fluid-like motion by constructing a divergence-free vector field from a scalar potential [S2, S3, G].

## How it works
**Inputs and Outputs:**
- Input: Query point $\mathbf{P} = (x, y)$, time $t$, noise scale $s$.
- Output: Velocity vector $\mathbf{v} = (v_x, v_y)$.

**Data Structures:**
- Permutation table $P$: Array of size 512 containing integers $0..255$ to map coordinates to gradient indices [G].
- Gradient vectors $\mathbf{g}$: For 2D Perlin noise, typically 4 diagonals: $(\pm 1, \pm 1)$ [G].

**Steps:**
1. **Generate Scalar Potential:** Compute a single scalar noise value at $\mathbf{P}$ to form the potential $\psi$.
   $$ \psi = \text{PerlinNoise}(x/s, y/s, t) $$
2. **Compute Curl:** Calculate the 2D curl of $\psi$ to obtain the velocity field $\mathbf{v}$.
   $$ \mathbf{v} = \nabla \times (0, 0, \psi) = \begin{pmatrix} \frac{\partial \psi}{\partial y} \\ -\frac{\partial \psi}{\partial x} \end{pmatrix} $$
3. **Numerical Derivatives:** Since Perlin noise is interpolated, derivatives are computed using central finite differences to approximate the partial derivatives [G].

**Parameters:**
- Scale $s$: Controls the spatial frequency of the turbulence. Typical values range from $0.1$ to $10.0$ depending on the desired feature size [G].
- Time $t$: Advances the field to create animation. Incremented per frame or per simulation step [G].

**Edge Cases and Pitfalls:**
- **Tileability:** To create seamless looping textures, the noise function must be implemented with periodic boundary conditions or tileable gradient assignments [G].
- **Precision:** Computing derivatives of interpolated noise requires careful handling of the fade function derivatives or finite difference step size to maintain $C^1$ continuity and avoid numerical noise [G].
- **Zero Direction:** If the query point lies exactly on a lattice line where the gradient contribution is flat, the derivative may be numerically unstable; clamping or adding a small epsilon mitigates this [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec2:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __add__(self, other): return Vec2(self.x + other.x, self.y + other.y)
    def __sub__(self, other): return Vec2(self.x - other.x, self.y - other.y)
    def __mul__(self, s): return Vec2(self.x * s, self.y * s)
    def __truediv__(self, s): return Vec2(self.x / s, self.y / s)

# Standard data structures [G]
perm = list(range(256)) * 2
gradients = [Vec2(1,1), Vec2(-1,1), Vec2(1,-1), Vec2(-1,-1)]

def fade(t: float) -> float:
    return 6 * t**5 - 15 * t**4 + 10 * t**3

def lerp(a: float, b: float, t: float) -> float:
    return a + t * (b - a)

def dot(v1: Vec2, v2: Vec2) -> float:
    return v1.x * v2.x + v1.y * v2.y

def perlin_noise(p: Vec2, time: float) -> float:
    ix = int(math.floor(p.x)) & 255
    iy = int(math.floor(p.y)) & 255
    fx = p.x - math.floor(p.x)
    fy = p.y - math.floor(p.y)
    ux = fade(fx)
    uy = fade(fy)
    
    A = perm[ix] + iy
    AA = perm[A]
    AB = perm[A + 1]
    B = perm[ix + 1] + iy
    BA = perm[B]
    BB = perm[B + 1]
    
    g = lambda idx: gradients[idx % len(gradients)]
    n00 = dot(g(AA), p)
    n10 = dot(g(BA), p - Vec2(1, 0))
    n01 = dot(g(AB), p - Vec2(0, 1))
    n11 = dot(g(BB), p - Vec2(1, 1))
    
    return lerp(lerp(n00, n10, ux), lerp(n01, n11, ux), uy)

def curl_noise(p: Vec2, time: float, scale: float) -> Vec2:
    s = Vec2(p.x / scale, p.y / scale)
    eps = 1e-4
    
    def dN_dx(pos):
        return (perlin_noise(pos + Vec2(eps, 0), time) - perlin_noise(pos - Vec2(eps, 0), time)) / (2 * eps)
    def dN_dy(pos):
        return (perlin_noise(pos + Vec2(0, eps), time) - perlin_noise(pos - Vec2(0, eps), time)) / (2 * eps)
        
    vx = dN_dy(s)
    vy = -dN_dx(s)
    
    return Vec2(vx, vy)
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, proportional to the number of lattice points in the cell (4 for 2D Perlin) times the number of derivative evaluations (4 for central differences) [G].
- **Memory Complexity:** $O(1)$ for the permutation table and gradient set [G].
- **Strengths:** Produces incompressible flow naturally; visually pleasing turbulence; efficient to evaluate; easily animatable via time parameter [G].
- **Limitations:** Requires careful tuning of scale to balance independence and coherence; finite difference derivatives amplify high-frequency noise if step size is too small [G].
- **Usage:** Commonly rendered using geometry shader sprites with accumulate blend to create colored density trails for fluid visualization [S1].

## Background
Curl noise was presented by Bridson et al. [S2, S3]. It is based on the mathematical property that the curl of any vector field is divergence-free [G]. The method extends earlier flow noise techniques by guaranteeing incompressibility without requiring a separate pressure solve [S2, G].

## Subtopics

### [Simplex Noise](978-simplex-noise.md)
A gradient noise variant with reduced directional artifacts. Use when isotropic noise is required. [G]

### [Improved Perlin Noise (4D)](3723-improved-perlin-noise-4d.md)
Uses 4D noise for 3D spatial variation. Use for higher quality 3D textures. [G]

### [Value Noise](4008-value-noise.md)
Interpolates scalar values at lattice points rather than gradients. Use for simpler implementations. [G]

### [Perlin Noise 2D](4211-perlin-noise-2d.md)
The 2D version of Perlin noise. Use for 2D textures. [G]

### [Perlin Gradient and Interpolation Functions](5415-perlin-gradient-and-interpolation-functions.md)
Details on the fade and gradient functions. Use for implementation details. [G]

### [Modified Noise (Dimension-Reducible)](18938-modified-noise-dimension-reducible.md)
Optimized for lower dimensions. Use when memory or performance is constrained. [G]

### [Fractal Brownian Motion](22237-fractal-brownian-motion.md)
Combines multiple octaves of noise. Use for multi-scale detail. [G]

## Sources
[S1] Part4 Chapter16 Fluid Simulation Demo, https://github.com/Razria93/Graphics_Study_Archive/issues/33, fetched 2026-10-26
[S2] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S3] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
