# Eikonal SDF Refinement

Also called: Eikonal Distance Field Solver, Fast Marching SDF Correction

## Overview
Eikonal SDF Refinement corrects inaccurate Signed Distance Field (SDF) values near the zero-level set by solving the Eikonal equation $|\nabla \phi| = 1$ using a fast-marching style approach over neighboring voxels [S1, unofficial]. It differs from simple relaxation methods by computing an analytical Eikonal solution in 3D and 2D planes to enforce the exact gradient magnitude constraint [S1, unofficial]. It builds on [SDF Validity Check with Relaxation](2399-sdf-validity-check-with-relaxation.md) [G].

## How it works
**Inputs:** A 3D grid of SDF values $\phi$ and grid spacing $g$ [S1, unofficial].
**Outputs:** Updated SDF values $\phi'$ where the Eikonal constraint is satisfied [S1, unofficial].

**Data Structures:**
- `phi`: 3D array of floats, indexed by $(x, y, z)$ [S1, unofficial].
- `g`: Scalar float representing the distance between adjacent voxel centers [S1, unofficial].

**Parameters:**
- `g`: Grid spacing, calculated as `g = _sdfRadius * 2.0 / _sdfTexSizeX` [S1, unofficial].

**Steps:**
1. Iterate over every voxel $(x, y, z)$ in the grid [S1, unofficial].
2. Compute distances to immediate neighbors along the X, Y, and Z axes using `neighborMin`, which returns `min(abs(a), abs(b))` where $a$ and $b$ are the SDF values at `kernel + delta` and `kernel - delta` for that axis [S1, unofficial].
3. Pass the three neighbor distances to `eikonal3d` [S1, unofficial]. If it returns true, use the result as the candidate distance $e$ [S1, unofficial]. Otherwise, fallback to 2D combinations (`eikonal2d` for XY, XZ, and YZ planes) and take the minimum valid result [S1, unofficial].
4. If the computed Eikonal distance $e$ is less than the current voxel value `current` and `current > g`, update `current = e` [S1, unofficial].

**Formulas:**

**3D Eikonal Solution:**
Given neighbor distances $u_1, u_2, u_3$ and grid spacing $g$:
$$ e = \frac{1}{n} \left( \sum_{i=1}^{n} u_i + \sqrt{ \left(\sum_{i=1}^{n} u_i\right)^2 - n \left( \sum_{i=1}^{n} u_i^2 - g^2 \right) } \right) $$
where $n=3$ [S1, unofficial].
Returns false if the discriminant $d = \left(\sum u_i\right)^2 - n \left( \sum u_i^2 - g^2 \right) < 0$ [S1, unofficial].

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
import math

def eikonal_2d(h, v, g):
    if abs(h - v) >= g:
        return min(h, v) + g
    sum_hv = h + v
    sum_sq = h**2 + v**2
    discriminant = sum_hv**2 - 2 * (sum_sq - g**2)
    if discriminant < 0:
        return None
    return 0.5 * (sum_hv + math.sqrt(discriminant))

def eikonal_3d(u, g):
    n = 3
    sum_u = sum(u)
    sum_u_sq = sum(val**2 for val in u)
    discriminant = sum_u**2 - n * (sum_u_sq - g**2)
    if discriminant < 0:
        return None, False
    e = (1.0 / n) * (sum_u + math.sqrt(discriminant))
    return e, True

def neighbor_min(phi, x, y, z, axis):
    if axis == 0:
        n1, n2 = phi[x+1, y, z], phi[x-1, y, z]
    elif axis == 1:
        n1, n2 = phi[x, y+1, z], phi[x, y-1, z]
    else:
        n1, n2 = phi[x, y, z+1], phi[x, y, z-1]
    return min(abs(n1), abs(n2))

def refine_sdf(phi, g):
    height, width, depth = phi.shape
    for x in range(1, width - 1):
        for y in range(1, height - 1):
            for z in range(1, depth - 1):
                current = phi[x, y, z]
                if current <= g:
                    continue
                u = [
                    neighbor_min(phi, x, y, z, 0),
                    neighbor_min(phi, x, y, z, 1),
                    neighbor_min(phi, x, y, z, 2)
                ]
                best_e = current
                e3d, valid3d = eikonal_3d(u, g)
                if valid3d:
                    best_e = min(best_e, e3d)
                else:
                    candidates = []
                    e_xy = eikonal_2d(u[0], u[1], g)
                    if e_xy is not None: candidates.append(e_xy)
                    e_xz = eikonal_2d(u[0], u[2], g)
                    if e_xz is not None: candidates.append(e_xz)
                    e_yz = eikonal_2d(u[1], u[2], g)
                    if e_yz is not None: candidates.append(e_yz)
                    if candidates:
                        best_e = min(best_e, min(candidates))
                if best_e < current:
                    phi[x, y, z] = best_e
```

**Edge Cases and Pitfalls:**
- The 3D Eikonal solver returns false if the discriminant $d < 0$, forcing a fallback to 2D planes [S1, unofficial]. This handles degenerate cases where the gradient direction does not align with a single axis or plane [S1, unofficial].
- Boundary conditions: The pseudocode assumes interior voxels and requires padding or clamping for grid edges [G].
- Precision: The square root and discriminant calculations require floating-point stability [G].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
