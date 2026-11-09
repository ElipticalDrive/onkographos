# Schwarze's Analytic Inversion
Also called: Accurate Ray-Isosurface Intersection via Cubic Polynomial, Cardano's formula method, Cubic Intersection Search, Exact Ray-Surface Intersection, Schwarze approach

## Overview
Schwarze's Analytic Inversion is a method for computing the exact intersection of a viewing ray with an iso-surface defined by trilinearly interpolated scalar data within a voxel [S1, S3, unofficial]. It models the scalar value along the ray segment as a cubic polynomial and solves for the precise root using Cardano's formula [S3, unofficial]. The method is mathematically the most accurate solution for iso-surface intersection but is computationally expensive and numerically unstable in single precision, making it unsuitable for interactive volume rendering [S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, voxel corner scalar values $\rho_{ijk}$ for $i,j,k \in \{0,1\}$, iso-value $\rho_{iso}$, voxel dimensions $\Delta x, \Delta y, \Delta z$, and traversal interval $[t_{in}, t_{out}]$ [S1, S3, unofficial].
- Output: The smallest ray parameter $t \in [t_{in}, t_{out}]$ where the interpolated density equals $\rho_{iso}$, if such a root exists [S3, unofficial].

**Data Structures:**
The method operates on the coefficients of the cubic polynomial $f(t) = \rho(t) - \rho_{iso}$ [S1, S3, unofficial]. Let the ray be $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ [G]. The density $\rho(t)$ is a cubic polynomial in $t$:
$$ \rho(t) = a t^3 + b t^2 + c t + d $$
The coefficients $a, b, c, d$ are estimated from four scalar values sampled along the ray segment within the voxel [S1, unofficial]. The target polynomial is $f(t) = a t^3 + b t^2 + c t + (d - \rho_{iso})$ [G].

**Steps:**
1. **Sample Values:** Identify $p_0$ and $p_3$ as the ray entry and exit points of the voxel. Define $p_1$ and $p_2$ as the trisection points of the segment $p_0 p_3$. Fetch the scalar values $\rho(p_0), \rho(p_1), \rho(p_2), \rho(p_3)$ using trilinear interpolation [S1, unofficial].
2. **Estimate Coefficients:** Compute $a, b, c, d$ such that the cubic polynomial passes through the four sampled values [S1, unofficial].
3. **Check Special Cases:** If $|a|$ is negligible, the polynomial reduces to quadratic or lower degree [S3, unofficial]. If $|a|$ and $|b|$ are negligible, it reduces to linear [S3, unofficial]. Solve these directly using the quadratic or linear formula [S3, unofficial]. Note that even the quadratic case requires a costly square root operation [S3, unofficial].
4. **Solve Cubic (Cardano's Formula):** For the full cubic case ($a \neq 0$), normalize the polynomial to $x^3 + px + q = 0$ by substituting $t = x - \frac{b}{3a}$ [G]. Compute the discriminant $D = (q/2)^2 + (p/3)^3$ [G]. Apply Cardano's formula, which involves computing cosines and inverse cosines to handle the trigonometric solution for three real roots when $D \le 0$ [S3, unofficial].
5. **Filter Roots:** Compute all roots of the polynomial and identify those lying within $[t_{in}, t_{out}]$ [S3, unofficial].
6. **Return Result:** Return the smallest valid $t$ as the intersection point [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def schwarze_analytic_inversion(ray_origin, ray_dir, voxel_corners, rho_iso, t_in, t_out, voxel_size):
    # voxel_corners: [rho_000, rho_100, rho_010, rho_110, rho_001, rho_101, rho_011, rho_111]
    # voxel_size: [dx, dy, dz]
    
    def cbrt(x):
        return math.copysign(abs(x)**(1/3), x)

    def trilinear_value(p, corners, vs):
        u, v, w = p[0]/vs[0], p[1]/vs[1], p[2]/vs[2]
        u = max(0.0, min(1.0, u))
        v = max(0.0, min(1.0, v))
        w = max(0.0, min(1.0, w))
        c00 = corners[0]
        c10 = corners[1] - corners[0]
        c01 = corners[2] - corners[0]
        c001 = corners[4] - corners[0]
        c11 = corners[3] - corners[1] - corners[2] + corners[0]
        c101 = corners[5] - corners[1] - corners[4] + corners[0]
        c011 = corners[6] - corners[2] - corners[4] + corners[0]
        c111 = corners[7] - corners[3] - corners[5] - corners[6] + corners[1] + corners[2] + corners[4] - corners[0]
        return (c00 + c10*u + c01*v + c001*w +
                c11*u*v + c101*u*w + c011*v*w +
                c111*u*v*w)

    EPSILON = 1e-8
    # 1. Sample four values along the ray segment [S1, unofficial]
    t0, t1, t2, t3 = t_in, t_in + (t_out - t_in)/3.0, t_in + 2*(t_out - t_in)/3.0, t_out
    p0 = ray_origin + t0 * ray_dir
    p1 = ray_origin + t1 * ray_dir
    p2 = ray_origin + t2 * ray_dir
    p3 = ray_origin + t3 * ray_dir
    v0 = trilinear_value(p0, voxel_corners, voxel_size)
    v1 = trilinear_value(p1, voxel_corners, voxel_size)
    v2 = trilinear_value(p2, voxel_corners, voxel_size)
    v3 = trilinear_value(p3, voxel_corners, voxel_size)

    # 2. Estimate cubic coefficients f(t) = a*t^3 + b*t^2 + c*t + d [G]
    # Solving the linear system for the four points
    A = [[t0**3, t0**2, t0, 1],
         [t1**3, t1**2, t1, 1],
         [t2**3, t2**2, t2, 1],
         [t3**3, t3**2, t3, 1]]
    # Gaussian elimination to solve A * [a,b,c,d]^T = [v0,v1,v2,v3]^T
    M = [A[i] + [v0, v1, v2, v3][i] for i in range(4)]
    for i in range(4):
        pivot = M[i][i]
        for j in range(i+1, 4):
            factor = M[j][i] / pivot
            for k in range(i, 5):
                M[j][k] -= factor * M[i][k]
    coeffs = [0.0]*4
    for i in range(3, -1, -1):
        coeffs[i] = M[i][4]
        for j in range(i+1, 4):
            coeffs[i] -= M[i][j] * coeffs[j]
        coeffs[i] /= M[i][i]
    a, b, c, d = coeffs
    d -= rho_iso
    
    # 3. Handle special cases [S3, unofficial]
    if abs(a) < EPSILON:
        if abs(b) < EPSILON:
            # Linear: c*t + d = 0
            if abs(c) < EPSILON:
                return None
            t = -d / c
            return t if t_in <= t <= t_out else None
        else:
            # Quadratic: b*t^2 + c*t + d = 0
            disc = c*c - 4*b*d
            if disc < 0:
                return None
            sqrt_disc = math.sqrt(disc)
            t1 = (-c + sqrt_disc) / (2*b)
            t2 = (-c - sqrt_disc) / (2*b)
            valid = [t for t in (t1, t2) if t_in <= t <= t_out]
            return min(valid) if valid else None
    
    # 4. Full Cubic Case: a*t^3 + b*t^2 + c*t + d = 0
    shift = b / (3*a)
    p = (3*a*c - b*b) / (3*a*a)
    q = (2*b*b*b - 9*a*b*c + 27*a*a*d) / (27*a*a*a)
    
    roots = []
    D = (q/2)**2 + (p/3)**3
    
    if D > 0:
        # One real root using Cardano's radical formula
        sqrt_D = math.sqrt(D)
        u = cbrt(-q/2 + sqrt_D)
        v = cbrt(-q/2 - sqrt_D)
        roots.append(u + v + shift)
    else:
        # Three real roots using trigonometric formula [S3, unofficial]
        r = math.sqrt(-p**3 / 27.0)
        cos_theta = -q / (2 * r) if abs(r) > EPSILON else 0.0
        cos_theta = max(-1.0, min(1.0, cos_theta))
        theta = math.acos(cos_theta)
        for k in range(3):
            angle = (theta + 2 * math.pi * k) / 3.0
            x = 2 * math.sqrt(-p/3.0) * math.cos(angle)
            roots.append(x + shift)
        
    # 5. Filter and return smallest valid t
    valid = [t for t in roots if t_in <= t <= t_out]
    return min(valid) if valid else None
```

**Parameters:**
- `EPSILON`: A small threshold for determining if coefficients $a$ or $b$ are negligible. Value not given in the sources [G].
- `rho_iso`: The iso-surface density value.

**Edge Cases and Pitfalls:**
- **Numerical Stability:** The algebraic solution is prone to numerical problems, especially in single-precision floating point, making it difficult to tune to avoid incorrectly computed intersections [S3, unofficial].
- **Special Cases:** Small coefficients $A$ and/or $B$ indicate the polynomial does not have full degree and must be solved more directly (quadratic or linear) [S3, unofficial].
- **Cost:** The method involves costly square root, cosine, and inverse cosine operations [S3, unofficial].
- **Boundary Handling:** Roots must be strictly checked against $[t_{in}, t_{out}]$ to avoid returning intersections outside the current voxel or behind the ray origin [S1, S3, unofficial].

## Subtopics

### Marmitt Voxel Intersection
A voxel intersection method that solves the cubic polynomial for ray-SDF intersection by first splitting the polynomial using the method proposed by Marmitt et al. [2004] and then using repeated linear interpolation for iteration [S4, unofficial]. It uses a maximum of 50 iterations and an epsilon threshold of $\epsilon = 4.0 \cdot 10^{-3}$ for convergence, as larger iteration counts or smaller epsilon values never visually improved rendered images [S4, unofficial]. Newton-Raphson can take longer to converge in harder cases where the starting point is on the cubic curve and the derivative magnitude is small, making Marmitt's method often faster than Newton-Raphson despite Newton-Raphson's faster theoretical convergence rates [S4, unofficial].

### Analytic Iso-Surface Intersection (Marmitt et al.)
A method to find the intersection of a ray with an iso-surface defined by trilinear interpolation within a cell by isolating roots via extrema, avoiding the numerical instability and cost of Cardano's formula [S5]. It substitutes the ray equation into the trilinear interpolation to form a cubic polynomial, computes the extrema to split the ray segment into at most three parts, and steps through these segments from front to back to find the interval containing zero [S5]. The root in the identified interval is calculated using recursive bisection until desired accuracy is reached or 3 to 4 steps are performed, which is sufficient for no visual differences [S5]. This method is approximately three times faster than Schwarze's algorithm for solving Cardano's formula, while being more numerically stable and using only single precision [S5].

## Operating characteristics
- **Time Complexity:** High. The method requires computing cube roots, square roots, and trigonometric functions (arccos, cos) for every voxel intersection test [S3, unofficial].
- **Memory Complexity:** Low. It requires storing only the polynomial coefficients and a few temporary variables [G].
- **Performance:** Computationally very expensive [S2, unofficial]. It is considered too slow for interactive volume rendering due to speed and numerical stability issues compared to isolation methods [S3, unofficial].
- **Strengths:** Mathematically the most accurate solution for computing intersection in volume ray tracing [S3, unofficial]. It provides exact intersection points.
- **Limitations:** Prone to numerical instability, particularly in single precision. The computational cost is prohibitive for real-time applications [S3, unofficial].
- **When to use:** When exact intersection accuracy is critical and performance is not a constraint [S3, unofficial].
- **When not to use:** For interactive volume rendering or when numerical stability in single precision is required [S3, unofficial].

## Background
- **Origin:** Introduced by Lin et al. and Parker et al. [S1, unofficial]. The method is also referred to as the Schwarze approach, citing [Sch90] and [PSL+98] [S3, unofficial].
- **History:** It was developed as a highly accurate alternative to approximate methods like the simple midpoint algorithm or linear interpolation refinements [S1, S2, unofficial].
- **Usage:** It is commonly applied in volume ray tracing where exactness is prioritized over speed, though it has largely been superseded by more stable iterative or isolation-based methods for interactive applications [S3, unofficial].

## Sources
[S1] Efficient and Effective Volume Visualization with Enhanced Isosurface Rendering, https://doi.org/10.48550/arxiv.1202.5360, fetched 2026-10-26 (unofficial)
[S2] (PDF) Cell-Based First-Hit Ray Casting - ResearchGate, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S3] PDF Fast and Accurate Ray-Voxel Intersection Techniques for Iso-Surface Ray ..., https://www.sci.utah.edu/~wald/Publications/2004/iso/IsoIsec_VMV2004.pdf, fetched 2026-10-29 (unofficial)
[S4] Ray tracing of signed distance function grids, https://www.jcgt.org/published/0011/03/06/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
[S5] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
