# Hermite Interpolation of Scalar Fields

Also called: C1-continuous Hermite Interpolation, Hermite Interpolation

## Overview
Hermite Interpolation of Scalar Fields renders scalar-valued functions (such as height fields or signed distance fields) by storing partial derivatives alongside function values and using Hermite interpolation between samples to guarantee a globally $C^1$-continuous result [S1, unofficial]. It differs from its siblings by explicitly utilizing gradient information to construct a cubic Hermite surface, allowing the exact normals of this surface to be used as shading normals, which provides a storage-efficient replacement for normal mapping with richer visual appearance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A discrete grid of scalar values $f$ and their partial derivatives $\nabla f$ (or surface normals). For full 3D $C^1$ continuity, mixed partial derivatives are also required [S1, unofficial].
*   **Output:** A continuous scalar field $u(\mathbf{p})$ and its gradient $\nabla u(\mathbf{p})$ at any query point $\mathbf{p}$ [G].

**Data Structures:**
The grid stores scalar values and derivative vectors at each vertex. When full 3D $C^1$ continuity is required, mixed partial derivatives are approximated akin to Adini's method for parametric surface patches [S1, unofficial].

**Steps:**
1.  **Coordinate Mapping:** Map the continuous query point $\mathbf{p} = (x, y, z)$ to local fractional coordinates $\tilde{\mathbf{x}} = (t_x, t_y, t_z) \in [0, 1]^3$ within the unit cell defined by the enclosing voxel [G].
2.  **Support Selection:** Select the 8 vertices at the corners of the enclosing voxel [G].
3.  **Basis Evaluation:** Compute the 1D cubic Hermite basis functions for each dimension [G].
4.  **Weighted Summation:** Evaluate the tensor product of the basis functions against the 8 support points to compute $u(\mathbf{p})$ [G].
5.  **Gradient Computation:** Differentiate the basis functions with respect to the query coordinates to compute $\nabla u(\mathbf{p})$; the exact normals of the reconstructed cubic Hermite surface are used as shading normals [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hermite_interpolate(grid, gradients, p, spacing):
    """
    3D Cubic Hermite Interpolation for a scalar field.
    grid: 3D array of scalar values
    gradients: 3D array of vectors (dx, dy, dz)
    p: (x, y, z) query point
    spacing: (dx, dy, dz) grid spacing
    """
    ix, iy, iz = int(p[0]), int(p[1]), int(p[2])
    tx, ty, tz = p[0] - ix, p[1] - iy, p[2] - iz

    def basis(t):
        t2, t3 = t*t, t*t*t
        # Basis for v0, v1, d0, d1 respectively
        return 2*t3 - 3*t2 + 1, -2*t3 + 3*t2, t3 - 2*t2 + t, t3 - t2

    hx = basis(tx)
    hy = basis(ty)
    hz = basis(tz)

    # Pre-gather 8 vertices: values and scaled derivatives (dx, dy, dz)
    verts = []
    for dx in range(2):
        for dy in range(2):
            for dz in range(2):
                v = grid[ix+dx, iy+dy, iz+dz]
                g = gradients[ix+dx, iy+dy, iz+dz]
                # Scale derivatives by spacing to match unit-cell basis formulation
                verts.append((v, g[0]*spacing[0], g[1]*spacing[1], g[2]*spacing[2]))

    # Interpolate along X
    ux = [0.0]*4
    dux_dy = [0.0]*4
    dux_dz = [0.0]*4
    for dy in range(2):
        for dz in range(2):
            idx0 = dy*2 + dz
            idx1 = idx0 + 4
            v0, d0x, d0y, d0z = verts[idx0]
            v1, d1x, d1y, d1z = verts[idx1]
            ux[idx0] = hx[0]*v0 + hx[1]*v1 + hx[2]*d0x + hx[3]*d1x
            dux_dy[idx0] = hx[0]*d0y + hx[1]*d1y
            dux_dz[idx0] = hx[0]*d0z + hx[1]*d1z

    # Interpolate along Y
    uy = [0.0]*2
    duy_dz = [0.0]*2
    for dz in range(2):
        uy[dz] = hy[0]*ux[dz] + hy[1]*ux[dz+2] + hy[2]*dux_dy[dz] + hy[3]*dux_dy[dz+2]
        duy_dz[dz] = hy[0]*dux_dz[dz] + hy[1]*dux_dz[dz+2]

    # Interpolate along Z
    val = hz[0]*uy[0] + hz[1]*uy[1] + hz[2]*duy_dz[0] + hz[3]*duy_dz[1]
    return val
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The method assumes access to derivatives at grid boundaries; if missing, they must be approximated [G].
*   **Mixed Partial Derivatives:** Full 3D $C^1$ continuity requires mixed partial derivatives ($\partial_{xy}, \partial_{xz}, \partial_{yz}, \partial_{xyz}$), which are approximated akin to Adini's method for parametric surface patches [S1, unofficial].
*   **Precision and Scaling:** Derivatives must be scaled by the grid spacing to align with the unit-cell basis formulation; non-uniform spacing requires per-dimension scaling [G].

## Subtopics

### Floating Scale Surface Reconstruction (FSSR) Integration
In FSSR, Hermite interpolation replaces the original weighting function with $w(r) = \frac{1}{12} (r - 3)^{12} (r + 1)^4$ to ensure $C^1$ continuity [S2]. This improves mean and RMS error and visual appearance compared to quadratic methods, although the maximum error can remain large [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires storing partial derivatives alongside scalar values; does not necessarily require extra storage if surface normals are already part of the discrete field [S1, unofficial].
*   **Strengths:** Globally $C^1$-continuous result; storage-efficient replacement for normal mapping with richer visual appearance [S1, unofficial].
*   **Limitations:** Requires computation or storage of derivatives and mixed partials [S1, unofficial].

## Background
The method was proposed to address the $C^0$ approximation limitations of bi- or trilinear filtering in real-time computer graphics [S1, unofficial]. It generalizes to arbitrary orders and dimensions and has been demonstrated in height field and signed distance field rendering [S1, unofficial]. It has also been implemented in Floating Scale Surface Reconstruction (FSSR) and Poisson Surface Reconstruction (PSR) for isosurface extraction [S2].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S2] Accurate Isosurface Interpolation with Hermite Data, https://ieeexplore.ieee.org/document/7335492/, fetched 2026-11-05
