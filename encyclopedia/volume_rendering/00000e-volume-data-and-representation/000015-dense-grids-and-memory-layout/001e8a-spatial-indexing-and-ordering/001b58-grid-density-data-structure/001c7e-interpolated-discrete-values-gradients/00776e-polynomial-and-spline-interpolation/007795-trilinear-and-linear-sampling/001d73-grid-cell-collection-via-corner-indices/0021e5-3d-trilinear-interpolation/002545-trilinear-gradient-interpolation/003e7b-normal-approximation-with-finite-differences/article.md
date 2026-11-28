# Normal Approximation with Finite Differences
Also called: Smooth normal sampling
## Overview
Normal Approximation with Finite Differences computes the surface normal at a ray-scalar field intersection point by approximating the gradient via finite differences of the scalar field values [S1, unofficial]. It sets itself apart from sibling methods like [Trilinear Normal Interpolation](4892-trilinear-normal-interpolation.md) and [Gradient Volume Interpolation (General)](16060-gradient-volume-interpolation-general.md) by evaluating the scalar field on-the-fly at displaced points rather than relying on pre-computed or stored gradient volumes [S1, unofficial].
## How it works
**Inputs:**
- Intersection point $P = (P_x, P_y, P_z)$ within the current voxel [S1, unofficial].
- Scalar field function $T(\cdot)$, implemented as trilinear interpolation over the voxel grid [S1, unofficial].
- Epsilon $\epsilon$, set to a value slightly less than half the voxel size [S1, unofficial].

**Outputs:**
- Normal vector $\mathbf{N} = (N_x, N_y, N_z)$, representing the unnormalized gradient approximation [S1, unofficial].

**Data Structures:**
- The scalar field grid and the trilinear interpolation function $T$ [S1, unofficial]. No additional data structures are required beyond the grid bounds and voxel dimensions [G].

**Steps:**
1. **Initialize Epsilon:** Set $\epsilon$ to a value slightly less than half the voxel size [S1, unofficial].
2. **Compute Gradient Components:** For each axis $i \in \{x, y, z\}$, calculate the gradient component $N_i$ using the central difference formula:
   $$ N_i = T(P_i + \epsilon) - T(P_i - \epsilon) $$
   where $T$ evaluates the scalar field via trilinear interpolation [S1, unofficial].
3. **Handle Boundary Conditions:** If any query point $(P_i \pm \epsilon)$ falls outside the current voxel, query the respective neighboring voxel and sample the distance inside that neighbor using trilinear interpolation [S1, unofficial]. If the neighbor query falls outside the entire grid, return the extrapolated distance from the current voxel [S1, unofficial].
4. **Return Gradient:** The resulting vector $\mathbf{N}$ is the unnormalized normal [S1, unofficial]. It may be normalized to unit length for shading calculations [G].

**Pitfalls:**
- The approximation is continuous inside the voxel but not on the edges, causing artifacts [S1, unofficial].
- Precision issues may arise if $\epsilon$ is too small or too large relative to the voxel size and field variation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal(P, epsilon, voxel_grid, voxel_size, voxel_min, grid_dims):
    """
    Compute surface normal at point P using finite differences.
    P: (Px, Py, Pz) intersection point
    epsilon: step size
    voxel_grid: 3D array of scalar values
    voxel_size: (sx, sy, sz) size of the current voxel
    voxel_min: (vx0, vy0, vz0) min corner of current voxel
    grid_dims: (gx, gy, gz) total voxel dimensions of the grid
    """
    def trilinear(pt, v_min, v_max):
        # Standard trilinear interpolation [G]
        t = [(pt[i] - v_min[i]) / (v_max[i] - v_min[i]) for i in range(3)]
        t = [max(0.0, min(1.0, ti)) for ti in t]
        # Grid indices for the 8 corners of the voxel containing pt
        ix = [int((pt[0] - grid_dims[0][0]) / voxel_size[0]), int((pt[0] - grid_dims[0][0]) / voxel_size[0]) + 1]
        iy = [int((pt[1] - grid_dims[1][0]) / voxel_size[1]), int((pt[1] - grid_dims[1][0]) / voxel_size[1]) + 1]
        iz = [int((pt[2] - grid_dims[2][0]) / voxel_size[2]), int((pt[2] - grid_dims[2][0]) / voxel_size[2]) + 1]
        # Clamp indices to grid bounds
        ix = [max(0, min(grid_dims[0]-1, i)) for i in ix]
        iy = [max(0, min(grid_dims[1]-1, i)) for i in iy]
        iz = [max(0, min(grid_dims[2]-1, i)) for i in iz]
        # Bilinear interpolation across each axis
        def lerp(a, b, t): return a + (b - a) * t
        v00 = lerp(voxel_grid[ix[0]][iy[0]][iz[0]], voxel_grid[ix[1]][iy[0]][iz[0]], t[0])
        v10 = lerp(voxel_grid[ix[0]][iy[1]][iz[0]], voxel_grid[ix[1]][iy[1]][iz[0]], t[0])
        v01 = lerp(voxel_grid[ix[0]][iy[0]][iz[1]], voxel_grid[ix[1]][iy[0]][iz[1]], t[0])
        v11 = lerp(voxel_grid[ix[0]][iy[1]][iz[1]], voxel_grid[ix[1]][iy[1]][iz[1]], t[0])
        v0 = lerp(v00, v10, t[1])
        v1 = lerp(v01, v11, t[1])
        return lerp(v0, v1, t[2])

    def sample_scalar(point):
        # Check if point is inside current voxel
        if all(voxel_min[i] <= point[i] <= voxel_min[i] + voxel_size[i] for i in range(3)):
            return trilinear(point, voxel_min, tuple(voxel_min[i] + voxel_size[i] for i in range(3)))
        # Check if point is inside grid bounds to find neighbor
        if all(grid_dims[i][0] <= point[i] <= grid_dims[i][1] for i in range(3)):
            # Compute neighbor voxel min by shifting one voxel size in the out-of-bounds axis
            neighbor_min = list(voxel_min)
            for i in range(3):
                if point[i] < voxel_min[i]: neighbor_min[i] -= voxel_size[i]
                elif point[i] > voxel_min[i] + voxel_size[i]: neighbor_min[i] += voxel_size[i]
            return trilinear(point, neighbor_min, tuple(neighbor_min[i] + voxel_size[i] for i in range(3)))
        # Outside grid: extrapolate by clamping point to current voxel bounds
        clamped = [max(voxel_min[i], min(point[i], voxel_min[i] + voxel_size[i])) for i in range(3)]
        return trilinear(clamped, voxel_min, tuple(voxel_min[i] + voxel_size[i] for i in range(3)))

    Nx = sample_scalar((P[0] + epsilon, P[1], P[2])) - sample_scalar((P[0] - epsilon, P[1], P[2]))
    Ny = sample_scalar((P[0], P[1] + epsilon, P[2])) - sample_scalar((P[0], P[1] - epsilon, P[2]))
    Nz = sample_scalar((P[0], P[1], P[2] + epsilon)) - sample_scalar((P[0], P[1], P[2] - epsilon))
    return (Nx, Ny, Nz)
```
## Subtopics

### [Surface Normal Calculation via Central Difference](13157-surface-normal-calculation-via-central-difference.md)
Calculates the surface normal $\hat{n}$ at an intersection point $P$ within a scalar field by approximating the gradient $\nabla \rho$ using a central difference scheme. It differs from its parent method by explicitly requiring trilinear interpolation calls to evaluate the scalar field at displaced points, rather than relying on pre-computed or stored gradients.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Provides smooth normals by handling voxel boundaries gracefully, reducing artifacts [S1, unofficial].
- **Limitations:** Computationally more expensive than using pre-computed gradients due to multiple interpolation calls [G]. Continuity issues at voxel edges if not handled with neighbors [S1, unofficial].
- **When to Use:** When smooth shading is required and pre-computed gradients are not available or too expensive to store [G].
- **When Not to Use:** When performance is critical and pre-computed gradients are available, or when the scalar field is already smooth enough that finite differences are unnecessary [G].
## Background
Not given in the sources.
## Sources
[S1] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
