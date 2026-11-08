# Brute-Force QEF Solver

Also called: Grid Search QEF Solver, Iterative Approximation via Grid Search

## Overview
A method for solving the Quadratic Error Function (QEF) by performing a brute-force grid search over the unit cube $[0,1]^3$ to find the point with the minimum squared distance to a set of planes, rather than using a closed-form linear algebra solution [S1, unofficial]. It serves as a robust, non-linear solver alternative for Dual Contouring when analytical solutions are unstable or when the QEF data is sparse [S1, unofficial]. It differs from [QEF Solver Initialization](4722-qef-solver-initialization.md) by providing the full optimization routine, and from [Quadric Error Function (QEF) for Sharp Features](1604-quadric-error-function-qef-for-sharp-features.md) by ignoring feature preservation in favor of exhaustive spatial sampling.

## How it works
**Inputs:**
- `data`: A struct containing:
  - `positions`: Array of 3D vectors ($P_i$) representing zero-crossing points on voxel edges.
  - `normals`: Array of 3D vectors ($n_i$) representing surface normals at those points.
  - `error`: Float, initialized to `float.MaxValue` to track the minimum error found.
  - `point`: 3D vector, initialized to store the best candidate point.
- `steps`: Integer, the number of divisions along each axis of the search grid. Default is 8 [S1, unofficial].

**Outputs:**
- `data.point`: The 3D vector $(x, y, z)$ within $[0,1]^3$ that minimizes the sum of squared distances to all planes defined by `data.positions` and `data.normals`.
- `data.error`: The minimum squared error value.

**Data Structures:**
The search space is discretized into a uniform 3D grid. The grid spans the unit cube $[0,1]^3$. The resolution is controlled by `steps`.

**Algorithm Steps:**
1. Initialize `data.error` to `float.MaxValue` [S1, unofficial].
2. Calculate the grid increment: `increment = 1.0 / steps` [S1, unofficial].
3. Iterate $x$ from $0$ to $1$ inclusive, with step size `increment` [S1, unofficial].
4. Nested iterate $y$ from $0$ to $1$ inclusive, with step size `increment` [S1, unofficial].
5. Nested iterate $z$ from $0$ to $1$ inclusive, with step size `increment` [S1, unofficial].
6. Construct candidate point: $P_{cand} = (x, y, z)$ [S1, unofficial].
7. Calculate error for $P_{cand}$:
   - Initialize `current_error` to $0$ [S1, unofficial].
   - For each plane $i$ from $0$ to `data.positions.Length - 1`:
     - Let $P_i = \text{data.positions}[i]$ and $n_i = \text{data.normals}[i]$ [S1, unofficial].
     - Calculate the signed distance from $P_{cand}$ to the plane: $d_i = n_i \cdot (P_{cand} - P_i)$ [S1, unofficial].
     - Add the squared distance to the total: `current_error` += $d_i^2$ [S1, unofficial].
8. Update minimum: If `current_error` < `data.error`, set `data.point` = $P_{cand}$ and `data.error` = `current_error` [S1, unofficial].
9. Return `data` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_brute_force_qef(positions, normals, steps=8):
    # Initialize tracking variables
    min_error = float('inf')
    best_point = None
    
    # Calculate grid resolution
    increment = 1.0 / steps
    
    # Iterate over the unit cube [0, 1]^3
    x = 0.0
    while x <= 1.0:
        y = 0.0
        while y <= 1.0:
            z = 0.0
            while z <= 1.0:
                # Construct candidate point
                candidate = (x, y, z)
                
                # Calculate QEF error for this candidate
                current_error = 0.0
                for i in range(len(positions)):
                    p_i = positions[i]
                    n_i = normals[i]
                    
                    # Vector from plane point to candidate
                    diff = (candidate[0] - p_i[0], 
                            candidate[1] - p_i[1], 
                            candidate[2] - p_i[2])
                    
                    # Dot product for signed distance
                    dist = n_i[0]*diff[0] + n_i[1]*diff[1] + n_i[2]*diff[2]
                    
                    # Add squared distance
                    current_error += dist * dist
                
                # Update minimum if better
                if current_error < min_error:
                    min_error = current_error
                    best_point = candidate
                
                z += increment
            y += increment
        x += increment
        
    return best_point, min_error
```

**Parameters:**
- `steps`: Determines the resolution of the grid search. Default value is 8 [S1, unofficial]. Higher values increase accuracy but exponentially increase computation time ($O(steps^3)$).

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point accumulation errors in `current_error` may affect tie-breaking, though unlikely to change the global minimum significantly given the discrete search space [S1, unofficial].
- **Boundaries:** The loop includes the upper bound ($1.0$) to ensure the entire unit cube is covered [S1, unofficial].
- **Degenerate Input:** If `positions` or `normals` are empty, the error remains 0 and the best point is the first grid point $(0,0,0)$ [G].

## Subtopics

### QEF Solver with Least Squares Approximation
A method to determine a single point approximating the isosurface by solving a linear system of intersecting planes derived from Hermite data (Zero Crossing positions and normals) [S2]. It is chosen when the number of intersecting planes is sufficient to form a determined or overdetermined system, offering higher precision than grid search for well-sampled voxels [S2]. The linear system for finding intersection point $x$ of three planes (defined by points $P_i$ and normals $n_i$) is:
$$n_1 \cdot (x - P_1) = 0$$
$$n_2 \cdot (x - P_2) = 0$$
$$n_3 \cdot (x - P_3) = 0$$
The exact intersection point is approximated using the equation:
$$E(x) = x - n_i \cdot P_i$$
The obtained point minimizes the distances to all involved planes [S2]. If the system is underdetermined (e.g., only two or one plane described), a least squares solution is computed using accumulated QEF data [S2]. A key pitfall is that if the implicit surface intersects a voxel such that only two or one plane(s) are described, the system is underdetermined (intersection of two planes is a line, not a point) [S2]. Additionally, if the QEF solver computes a point outside the voxel cell, the solution falls back to the mass point (average of intersection positions) [S2].

## Operating characteristics
- **Time Complexity:** $O(steps^3 \cdot N)$, where $N$ is the number of planes (edges intersecting the voxel) [S1, unofficial]. With default `steps=8`, this is $512 \cdot N$ operations per voxel [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, as it iterates through the grid without storing intermediate candidates [S1, unofficial].
- **Performance:** Significantly slower than closed-form solvers for dense QEF data, but robust against ill-conditioned matrices [S1, unofficial].
- **Strengths:** Simple to implement; does not require matrix inversion or decomposition; guaranteed to find the global minimum within the discretized grid [S1, unofficial].
- **Limitations:** Resolution is limited by `steps`; cannot find sub-grid precision minima; computationally expensive for high-resolution searches [S1, unofficial].
- **When to use:** When the QEF system is ill-conditioned, when a closed-form solver fails, or when implementing a quick prototype [S1, unofficial].
- **When not to use:** For real-time rendering with high `steps` counts or large numbers of intersecting planes [S1, unofficial].

## Background
The brute-force grid search approach is documented in the context of Dual Contouring implementations using Octrees, specifically in the `DualContouring-3` repository [S1, unofficial]. It is presented as a straightforward alternative to solving the linear system analytically, particularly useful in educational or simplified rendering pipelines [S1, unofficial]. The concept of using QEF data to approximate isosurfaces via linear systems is further detailed in volumetric terrain rendering literature, which discusses handling underdetermined systems via least squares [S2].

## Sources
[S1] Alan-Baylis/DualContouring-3, https://github.com/Alan-Baylis/DualContouring-3, fetched 2026-10-26 (unofficial)
[S2] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
