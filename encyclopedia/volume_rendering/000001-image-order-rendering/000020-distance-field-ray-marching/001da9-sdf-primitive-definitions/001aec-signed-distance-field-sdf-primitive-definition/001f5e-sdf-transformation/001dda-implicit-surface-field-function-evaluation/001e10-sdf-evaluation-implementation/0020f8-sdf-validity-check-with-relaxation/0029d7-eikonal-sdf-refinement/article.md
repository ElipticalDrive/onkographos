# Eikonal SDF Refinement

Also called: Eikonal Distance Field Solver, Fast Marching SDF Correction

## Overview
Eikonal SDF Refinement corrects inaccurate Signed Distance Field (SDF) values near the zero-level set by solving the Eikonal equation $|\nabla \phi| = 1$ over neighboring voxels [S1, unofficial]. It sets itself apart from simple relaxation methods by using an analytical solution to the Eikonal equation in 3D and 2D planes, ensuring the gradient magnitude is exactly 1 where possible, rather than just smoothing values [S1, unofficial]. It builds on [SDF Validity Check with Relaxation](2399-sdf-validity-check-with-relaxation.md).

## How it works
**Inputs:** A 3D grid of SDF values $\phi$, grid spacing $g$.
**Outputs:** Updated SDF values $\phi'$ where the Eikonal constraint is satisfied.

**Data Structures:**
- `phi`: 3D array of floats, indexed by $(x, y, z)$.
- `g`: Scalar float representing the distance between adjacent voxel centers.

**Parameters:**
- `g`: Grid spacing. Calculated as `g = _sdfRadius * 2.0 / _sdfTexSizeX` [S1, unofficial].

**Steps:**
1. Iterate over every voxel $(x, y, z)$ in the grid.
2. Compute distances to immediate neighbors along the X, Y, and Z axes. For each axis, identify the minimum distance among the two opposing neighbors.
   - Let $u_x, u_y, u_z$ be these minimum neighbor distances.
   - Specifically, for axis $i$, $u_i = \min(\phi_{neighbor+}, \phi_{neighbor-})$. Note: The source describes `neighborMin` as `min(abs(a), abs(b))` where `a` and `b` are SDF values at `kernel + delta` and `kernel - delta` [S1, unofficial]. In the context of distance fields, we typically look at the values of the neighbors themselves to propagate distance.
3. Attempt to solve the 3D Eikonal equation using $u_x, u_y, u_z$.
   - If the 3D solution is valid (discriminant $\ge 0$), use it as the new value $e$.
   - If the 3D solution is invalid, fallback to 2D solutions for XY, XZ, and YZ planes.
   - The new value $e$ is the minimum of all valid solutions (3D and valid 2D planes).
4. Update the voxel: If $e < \phi_{current}$ and $\phi_{current} > g$, set $\phi_{current} = e$.

**Formulas:**

**3D Eikonal Solution:**
Given neighbor distances $u_1, u_2, u_3$ (corresponding to X, Y, Z axes) and grid spacing $g$:
$$ e = \frac{1}{n} \left( \sum_{i=1}^{n} u_i + \sqrt{ \left(\sum_{i=1}^{n} u_i\right)^2 - n \left( \sum_{i=1}^{n} u_i^2 - g^2 \right) } \right) $$
where $n=3$ [S1, unofficial].
Returns false (invalid) if the discriminant $d = \left(\sum u_i\right)^2 - n \left( \sum u_i^2 - g^2 \right) < 0$ [S1, unofficial].

**2D Eikonal Solution:**
Given two neighbor distances $h, v$ and grid spacing $g$:
- If $|h - v| \ge g$:
  $$ e = \min(h, v) + g $$
- Else:
  $$ e = \frac{1}{2} \left( (h+v) + \sqrt{(h+v)^2 - 2(h^2 + v^2 - g^2)} \right) $$
[S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def eikonal_2d(h, v, g):
    """
    Solves the 2D Eikonal equation for two neighbor distances h, v and grid spacing g.
    Returns the computed distance e.
    """
    if abs(h - v) >= g:
        return min(h, v) + g
    else:
        sum_hv = h + v
        sum_sq = h**2 + v**2
        discriminant = sum_hv**2 - 2 * (sum_sq - g**2)
        if discriminant < 0:
            return None # Invalid
        return 0.5 * (sum_hv + sqrt(discriminant))

def eikonal_3d(u, g):
    """
    Solves the 3D Eikonal equation for three neighbor distances u[0..2] and grid spacing g.
    Returns (e, valid).
    """
    n = 3
    sum_u = sum(u)
    sum_u_sq = sum(val**2 for val in u)
    
    discriminant = sum_u**2 - n * (sum_u_sq - g**2)
    
    if discriminant < 0:
        return None, False
    
    e = (1.0 / n) * (sum_u + sqrt(discriminant))
    return e, True

def neighbor_min(phi, x, y, z, axis, g):
    """
    Returns the minimum SDF value among the two neighbors along the given axis.
    Assumes boundary checks are handled by the caller or padding.
    """
    # Get neighbors
    if axis == 0: # X
        n1 = phi[x+1, y, z]
        n2 = phi[x-1, y, z]
    elif axis == 1: # Y
        n1 = phi[x, y+1, z]
        n2 = phi[x, y-1, z]
    else: # Z
        n1 = phi[x, y, z+1]
        n2 = phi[x, y, z-1]
    
    # The source says min(abs(a), abs(b)). In SDF context, we usually take the value.
    # However, if the source explicitly says abs, we follow it.
    # Note: Standard Eikonal solvers use the values directly. 
    # The source snippet: `min(abs(a), abs(b))` where a,b are SDF values.
    return min(abs(n1), abs(n2))

def refine_sdf(phi, g):
    """
    Refines the SDF grid using the Eikonal solver.
    """
    # Copy phi to avoid in-place issues if needed, or update in place carefully.
    # The source implies a single pass update: "If e < current ... update".
    # This suggests an in-place or single-buffer update.
    
    height, width, depth = phi.shape
    
    for x in range(1, width - 1):
        for y in range(1, height - 1):
            for z in range(1, depth - 1):
                current = phi[x, y, z]
                
                # If current value is already small enough, skip?
                # Source: "If e < current and current > g"
                if current <= g:
                    continue
                
                # Get neighbor mins
                u = [
                    neighbor_min(phi, x, y, z, 0, g),
                    neighbor_min(phi, x, y, z, 1, g),
                    neighbor_min(phi, x, y, z, 2, g)
                ]
                
                best_e = current
                
                # Try 3D
                e3d, valid3d = eikonal_3d(u, g)
                if valid3d:
                    best_e = min(best_e, e3d)
                
                # Try 2D planes
                # XY plane: u[0], u[1]
                e_xy = eikonal_2d(u[0], u[1], g)
                if e_xy is not None:
                    best_e = min(best_e, e_xy)
                
                # XZ plane: u[0], u[2]
                e_xz = eikonal_2d(u[0], u[2], g)
                if e_xz is not None:
                    best_e = min(best_e, e_xz)
                
                # YZ plane: u[1], u[2]
                e_yz = eikonal_2d(u[1], u[2], g)
                if e_yz is not None:
                    best_e = min(best_e, e_yz)
                
                # Update
                if best_e < current:
                    phi[x, y, z] = best_e
```

**Edge Cases and Pitfalls:**
- The 3D Eikonal solver returns false if the discriminant $d < 0$. This forces a fallback to 2D planes [S1, unofficial]. This handles degenerate cases where the gradient direction doesn't align with a single axis or plane [S1, unofficial].
- Boundary conditions: The pseudocode assumes interior voxels. Boundary handling (padding, clamping) is not detailed in the source [G].
- Precision: The square root and discriminant calculations require floating-point stability.

## Operating characteristics
**Time Complexity:** $O(N)$ where $N$ is the number of voxels, as it performs a single pass over the grid. Each voxel involves a constant number of arithmetic operations.
**Memory Complexity:** $O(1)$ auxiliary space if updated in-place, or $O(N)$ if a copy is required.
**Strengths:** Provides analytically correct gradient magnitudes near the surface. Faster than iterative relaxation for achieving the Eikonal constraint.
**Limitations:** Single-pass nature may not propagate corrections globally in one iteration. Requires fallback logic for degenerate cases.
**When to use:** When accurate normals and distances are critical near the zero-level set, and performance is better than full iterative solvers.
**When not to use:** If the grid is very large and a single pass is insufficient to propagate distances from distant sources, or if boundary conditions are complex.

## Background
Originated from the implementation of Signed Distance Fields in Unity, specifically in the context of rendering boids and planet textures using SDF primitives [S1, unofficial]. It is part of a pipeline that includes SDF primitive definitions, transformations, and validity checks.

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
