# Neubauer's Ray-Voxel Intersection Algorithm

Also called: Neubauer, Neubauer's Method, Repeated Linear Interpolation

## Overview
Neubauer's algorithm is a method for finding the intersection of a viewing ray with an iso-surface defined by trilinearly interpolated scalar data within a voxel [S1]. It solves the problem of locating the precise root of the implicit function $f(t) = \rho(R(t)) - \rho_{iso}$ by using repeated linear interpolation to refine the intersection position within the ray segment containing the root [S1]. It is distinguished from its siblings by its reliance on iterative linear approximation rather than closed-form algebraic solutions (like Schwarze's Analytic Inversion) or root isolation techniques (like Isolation and Iterative Root Finding), offering a balance of speed and moderate accuracy at the cost of potential artifacts in complex topologies [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray $R(t) = \mathbf{a} + t\mathbf{b}$, where $\mathbf{a}$ is the origin and $\mathbf{b}$ is the direction. The interval $[t_{in}, t_{out}]$ where the ray overlaps the voxel [S1]. The scalar field values $\rho_{ijk}$ at the eight vertices of the voxel [S1]. The iso-value $\rho_{iso}$. The number of iterations $N$ (typically 2-3) [S1].
*   **Output:** The hit distance $t_{hit}$ along the ray, or a "NO HIT" status if no intersection is found within the voxel [S1].

**Data Structures:**
The algorithm operates on scalar values representing the ray parameter $t$ and the interpolated density $\rho$ [S1].

**Method Steps:**

1.  **Initialization:**
    Set the search interval boundaries:
    $$t_0 := t_{in}$$
    $$t_1 := t_{out}$$
    Compute the density values at the boundaries:
    $$\rho_0 := \rho(R(t_0))$$
    $$\rho_1 := \rho(R(t_1))$$
    where $\rho(R(t))$ is computed via trilinear interpolation of the voxel's vertex data [S1].

2.  **Initial Sign Check:**
    If $\text{sign}(\rho_0 - \rho_{iso}) = \text{sign}(\rho_1 - \rho_{iso})$, return NO HIT [S1]. This indicates that the iso-surface does not cross the ray segment within the voxel, or crosses an even number of times, which the method cannot detect [S1].

3.  **Iterative Refinement (Repeated Linear Interpolation):**
    For $i = 1$ to $N$:
    a. Compute the candidate intersection parameter $t$ using linear interpolation (secant step):
    $$t := t_0 + (t_1 - t_0) \frac{\rho_{iso} - \rho_0}{\rho_1 - \rho_0}$$
    b. Evaluate the density at the candidate point:
    $$\rho_{new} := \rho(R(t))$$
    c. Determine which sub-segment contains the root by comparing signs:
    If $\text{sign}(\rho_{new} - \rho_{iso}) = \text{sign}(\rho_0 - \rho_{iso})$:
    The root lies in the upper sub-segment $[t, t_1]$ [S1]. Update:
    $$t_0 := t$$
    $$\rho_0 := \rho_{new}$$
    Else:
    The root lies in the lower sub-segment $[t_0, t]$ [S1]. Update:
    $$t_1 := t$$
    $$\rho_1 := \rho_{new}$$
    [S1]

4.  **Final Result:**
    After $N$ iterations, compute the final hit distance:
    $$t_{hit} := t_0 + (t_1 - t_0) \frac{\rho_{iso} - \rho_0}{\rho_1 - \rho_0}$$
    Return $t_{hit}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def NeubauerRayVoxelIntersection(ray_origin, ray_direction, voxel_bounds, 
                                 voxel_data, rho_iso, N=3):
    """
    ray_origin: tuple (x, y, z)
    ray_direction: tuple (dx, dy, dz)
    voxel_bounds: tuple (x0, x1, y0, y1, z0, z1)
    voxel_data: list/tuple of 8 floats representing density at voxel corners
    rho_iso: float, the iso-surface value
    N: int, number of interpolation iterations
    """
    
    # 1. Calculate ray-voxel overlap interval [tin, tout] using slab method
    x0, x1, y0, y1, z0, z1 = voxel_bounds
    tmin = -float('inf')
    tmax = float('inf')
    
    for axis in range(3):
        p = ray_origin[axis]
        d = ray_direction[axis]
        b0 = [x0, y0, z0][axis]
        b1 = [x1, y1, z1][axis]
        
        if abs(d) < 1e-12:
            if p < b0 or p > b1:
                return None
        else:
            t1 = (b0 - p) / d
            t2 = (b1 - p) / d
            if t1 > t2: t1, t2 = t2, t1
            tmin = max(tmin, t1)
            tmax = min(tmax, t2)
            
    if tmin > tmax:
        return None
        
    tin, tout = tmin, tmax
    
    # Helper to evaluate trilinear interpolation
    def evaluate_density(t):
        px = ray_origin[0] + t * ray_direction[0]
        py = ray_origin[1] + t * ray_direction[1]
        pz = ray_origin[2] + t * ray_direction[2]
        
        u = (px - x0) / (x1 - x0)
        v = (py - y0) / (y1 - y0)
        w = (pz - z0) / (z1 - z0)
        
        u = max(0.0, min(1.0, u))
        v = max(0.0, min(1.0, v))
        w = max(0.0, min(1.0, w))
        
        d000 = voxel_data[0]; d100 = voxel_data[1]
        d010 = voxel_data[2]; d110 = voxel_data[3]
        d001 = voxel_data[4]; d101 = voxel_data[5]
        d011 = voxel_data[6]; d111 = voxel_data[7]
        
        d00 = d000 * (1 - u) + d100 * u
        d10 = d010 * (1 - u) + d110 * u
        d01 = d001 * (1 - u) + d101 * u
        d11 = d011 * (1 - u) + d111 * u
        
        d0 = d00 * (1 - v) + d10 * v
        d1 = d01 * (1 - v) + d11 * v
        
        return d0 * (1 - w) + d1 * w

    # 2. Initialize interval
    t0 = tin
    t1 = tout
    rho0 = evaluate_density(t0)
    rho1 = evaluate_density(t1)
    
    # 3. Initial sign check
    def sign(val):
        if val > 0: return 1
        elif val < 0: return -1
        else: return 0
        
    if sign(rho0 - rho_iso) == sign(rho1 - rho_iso):
        return None
        
    # 4. Iterative refinement
    for i in range(N):
        if abs(rho1 - rho0) < 1e-12:
            t = (t0 + t1) / 2.0
        else:
            t = t0 + (t1 - t0) * (rho_iso - rho0) / (rho1 - rho0)
            
        rho_new = evaluate_density(t)
        
        if sign(rho_new - rho_iso) == sign(rho0 - rho_iso):
            t0 = t
            rho0 = rho_new
        else:
            t1 = t
            rho1 = rho_new
            
    # 5. Final result
    if abs(rho1 - rho0) < 1e-12:
        t_hit = (t0 + t1) / 2.0
    else:
        t_hit = t0 + (t1 - t0) * (rho_iso - rho0) / (rho1 - rho0)
        
    return t_hit
```

**Edge Cases and Pitfalls:**
*   **Multiple Intersections:** The algorithm assumes a single root within the segment defined by differing signs [S1]. It can falsely return the last intersection point if three intersections are contained within a voxel but two of them are within the first ray segment [S1].
*   **Complex Topologies:** In cases where the function has two roots, the entry and exit densities may be on the same side of $\rho_{iso}$, causing the initial sign check to return NO HIT incorrectly [S1].
*   **SIMD Compatibility:** The control flow involves conditional updates to $t_0, t_1, \rho_0, \rho_1$ based on sign comparisons [S1]. This inconsistent decision-making on which segment to handle makes the base algorithm less suitable for SIMD than methods with uniform control flow, although it is still functional and faster than Schwarze's method in SIMD contexts [S1].
*   **Precision:** As an iterative approximation, it is prone to approximation artifacts and image errors for complex voxel configurations compared to exact methods [S1].

## Subtopics

### [Schwarze's Analytic Inversion](1001-schwarze-s-analytic-inversion.md)
Schwarze's Analytic Inversion is a method for computing the exact intersection of a ray with an iso-surface defined by trilinearly interpolated scalar data within a voxel. It solves the cubic polynomial resulting from substituting the ray equation into the trilinear interpolation function to find the precise root. It is distinguished from sibling methods like Neubauer's algorithm by its reliance on closed-form algebraic solutions (Cardano's formula) rather than iterative refinement or isolation techniques, offering mathematical exactness at the cost of significant computational expense and numerical instability.

### [Isolation and Iterative Root Finding](1041-isolation-and-iterative-root-finding.md)
Isolation and Iterative Root Finding is a ray-voxel intersection algorithm for iso-surfaces that combines the speed of Neubauer's method with the correctness guarantees of Schwarze's method. It solves the problem of finding the first intersection of a ray with a cubic polynomial within a voxel by first isolating the root interval using the polynomial's extrema, then refining the location via iterative root finding. Unlike Neubauer's method, which can fail or return incorrect intersections when multiple roots exist in a segment, this method guarantees that the final interval contains exactly one root, ensuring the first valid intersection is found. It is roughly three times faster than Schwarze's method while maintaining superior numerical stability.

### [Marmitt's Ray-Cell Intersection](1606-marmitt-s-ray-cell-intersection.md)
Marmitt's Ray-Cell Intersection computes the point where a ray intersects an isosurface defined by trilinear interpolation within a 3D voxel cell. It solves for the first positive root of the cubic polynomial derived from the cell's corner values using an iterative bisection refinement strategy. Unlike analytical cubic solvers (e.g., Schwarze’s method), this approach is slightly faster and yields single-precision, numerically stable results. It builds on [Neubauer's Ray-Voxel Intersection Algorithm](975-neubauer-s-ray-voxel-intersection-algorithm.md) by applying iterative root finding specifically to the reconstructed cell geometry.

## Operating characteristics
**Time and Memory Complexity:**
The algorithm performs a fixed number of iterations $N$ (typically 2-3) [S1]. Each iteration requires one trilinear interpolation evaluation [S1]. Trilinear interpolation is $O(1)$ with a constant number of arithmetic operations [G]. Thus, the time complexity per ray-voxel intersection is $O(N)$, which is effectively constant [G]. Memory complexity is $O(1)$ as it only stores a few scalar variables [S1].

**Measured Performance:**
*   On a 1.8GHz AMD Opteron CPU, Neubauer achieves an average of 3.04 million ray-voxel intersections per second in synthetic setups [S1] (Table 1).
*   In SIMD mode on synthetic setups, Neubauer achieves an average of 3.04 million ray-cell intersections per second [S1] (Table 2).
*   On real-world datasets (Bonsai, Aneurism, Engine), Neubauer achieves approximately 2.93-2.94 million intersections per second [S1] (Table 3).
*   In SIMD mode on real-world datasets, Neubauer achieves 6.94-7.65 million intersections per second, a speedup of factor 2.3-2.6 over sequential code [S1] (Table 4).

**Strengths and Limitations:**
*   **Strengths:** Faster than the accurate but slow Schwarze algorithm [S1]. Better suited for SIMD implementation than Schwarze due to simpler control flow (though still not ideal) [S1]. Integrated into the OpenRT realtime ray tracing system [S1].
*   **Limitations:** Prone to approximation artifacts and image errors for complex voxel configurations [S1]. Can fail to locate valid intersections in cases with multiple roots or specific topological configurations [S1]. Less numerically stable than isolation-based methods [S1].

**When to Use:**
Use when performance is critical and the voxel data topology is relatively simple, or when an approximate intersection is acceptable [S1]. It is a good baseline for SIMD-accelerated volume rendering where the overhead of exact solvers is prohibitive [S1].

**When Not to Use:**
Avoid when high accuracy is required, especially for complex iso-surfaces with multiple intersections within a single voxel, or when numerical stability is paramount [S1].

## Background
Neubauer's algorithm was proposed by Neubauer et al. in 2002 [S1]. It was analyzed and compared with other ray-voxel intersection techniques (including Schwarze's method and linear interpolation) in the context of iso-surface volume rendering by Wald et al. in 2004 [S1]. The method is integrated into the OpenRT realtime ray tracing system [S1] for iso-surface volume rendering, where it serves as a fast, approximate intersection primitive [S1].

## Sources
[S1] Fast and Accurate Ray-Voxel Intersection Techniques for Iso-Surface Ray Tracing, https://www.sci.utah.edu/~wald/Publications/2004/iso/IsoIsec_VMV2004.pdf, fetched 2026-10-29
