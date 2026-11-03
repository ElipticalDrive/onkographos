# Marmitt's Ray-Cell Intersection

Also called: Cell Intersection, Isosurface Intersection (Neubauer iterative root finder), Isosurface Intersection via Neubauer Iterative Root Finder, Marmitt et al. 03 method, Marmitt et al. root finder, Neubauer iterative root finder, Ray-Isosurface Intersection

## Overview
Marmitt's Ray-Cell Intersection computes the intersection point of a ray with an isosurface defined by trilinear interpolation inside a 3D voxel cell [S1, S2]. It solves for the first positive root of the cubic polynomial derived from the cell's corner values using an iterative bisection refinement strategy [S1, S2]. Compared to analytical cubic solvers like Schwarze’s method, this approach is slightly faster and yields single-precision, numerically stable results [S1, S2]. It implements the Neubauer iterative root finder [S1, S2].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].
- Isosurface value $I_{\text{iso}}$ [S1, S2].
- Eight scalar values $v_0, \dots, v_7$ at the corners of the 3D cell [S1, S2].

**Outputs:**
- Intersection parameter $t$ along the ray [G]. If no root exists in $[0, \infty)$, the method reports no intersection [G].

**Data Structures:**
- The cell is defined by 8 scalar values [S1, S2]. No additional geometric data structures are required beyond the corner values and the ray parameters [G].

**Algorithm Steps:**
1. **Cell Reconstruction:** Reconstruct the cell from octree voxel data using forward neighbors [S3].
2. **Polynomial Formulation:** Compute the cubic polynomial $P(t)$ such that $P(t) = 0$ corresponds to the trilinear interpolation of the cell values equaling $I_{\text{iso}}$ along the ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ [S1, S2]. The coefficients are derived by substituting the ray equation into the trilinear basis functions [S1, S2].
3. **Iterative Root Finding:** Iteratively re-parameterize the ray into sub-intervals within the cell until a sign change is detected within the sub-interval and a root is found [S1, S2].
4. **Hit Position:** Evaluate the ray at the first positive root of the cubic polynomial to obtain the intersection point [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marmitt_ray_cell_intersection(ray_origin, ray_dir, corner_values, isovalue, tolerance=1e-6):
    """
    ray_origin: (ox, oy, oz)
    ray_dir: (dx, dy, dz)
    corner_values: list of 8 floats [v000, v100, v010, v110, v001, v101, v011, v111]
    isovalue: float
    tolerance: float, convergence threshold
    Returns: t (float) or None
    """
    ox, oy, oz = ray_origin
    dx, dy, dz = ray_dir
    v000, v100, v010, v110, v001, v101, v011, v111 = corner_values

    def evaluate_trilinear(t):
        x = ox + t * dx
        y = oy + t * dy
        z = oz + t * dz
        val = (v000 * (1 - x) * (1 - y) * (1 - z) +
               v100 * x * (1 - y) * (1 - z) +
               v010 * (1 - x) * y * (1 - z) +
               v110 * x * y * (1 - z) +
               v001 * (1 - x) * (1 - y) * z +
               v101 * x * (1 - y) * z +
               v011 * (1 - x) * y * z +
               v111 * x * y * z)
        return val - isovalue

    t_min = 0.0
    t_max = float('inf')
    p_min = evaluate_trilinear(t_min)

    while t_max - t_min > tolerance:
        t_mid = (t_min + t_max) / 2.0
        p_mid = evaluate_trilinear(t_mid)
        if p_min * p_mid < 0:
            t_max = t_mid
        else:
            t_min = t_mid
            p_min = p_mid
        if t_max == float('inf'):
            t_max = t_min * 2.0 + 1.0
            p_max = evaluate_trilinear(t_max)
            if p_min * p_max > 0:
                return None
    return (t_min + t_max) / 2.0
```

**Edge Cases and Pitfalls:**
- **No Sign Change:** If the polynomial does not change sign within the cell, there is no intersection [G].
- **Precision:** Single-precision arithmetic is used, which yields numerically stable results [S1, S2].
- **Gradient Computation:** Normals are computed using a forward-difference stencil on the same cell to avoid the high cost of neighbor-finding [S3].

## Operating characteristics
- **Time Complexity:** The root finding uses iterative bisection, converging logarithmically with precision. Each iteration evaluates the trilinear polynomial in constant time [G].
- **Memory Complexity:** O(1) additional memory per ray, as no cell data needs to be stored per ray [S3].
- **Performance:** Slightly faster than analytical cubic solvers [S1, S2].
- **Strengths:** Numerically stable in single precision [S1, S2]. Simple to implement.
- **Limitations:** Iterative nature may be slower than a direct analytical solution if the solver is highly optimized, but the source claims it is faster [S1, S2]. Requires cell reconstruction (neighbor finding) which is costly if not cached [S3].

## Background
The method is based on the Neubauer iterative root finder proposed by Marmitt et al. [S1, S2]. It is described in the context of isosurface ray tracing of octree volumes [S1, S2]. The approach was compared to Schwarze’s cubic solver used by Parker et al. and found to be faster and more stable in single precision [S1, S2].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://collections.lib.utah.edu/dl_files/1d/58/1d589a03a725c9d894a3866ad32f1386f0613f9d.pdf, fetched 2026-10-28
[S2] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S3] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://pdfs.semanticscholar.org/aef9/6e0dd17920d0c70cbf0dd43f494cab1c0bcc.pdf, fetched 2026-10-28
