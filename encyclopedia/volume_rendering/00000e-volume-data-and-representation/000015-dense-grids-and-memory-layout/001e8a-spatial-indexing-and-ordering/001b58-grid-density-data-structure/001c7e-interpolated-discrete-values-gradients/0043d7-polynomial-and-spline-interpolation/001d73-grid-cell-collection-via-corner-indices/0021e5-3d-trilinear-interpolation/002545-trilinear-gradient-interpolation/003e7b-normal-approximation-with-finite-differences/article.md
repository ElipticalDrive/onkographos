# Normal Approximation with Finite Differences
Also called: Smooth normal sampling
## Overview
Normal Approximation with Finite Differences computes the surface normal at a ray-scalar field intersection point by approximating the gradient via finite differences of the scalar field values [S1, unofficial]. It differs from its parent method, Trilinear Gradient Interpolation, by explicitly requiring trilinear interpolation calls to evaluate the scalar field at displaced points, rather than relying on pre-computed or stored gradients [S1, unofficial]. This approach avoids discontinuities at voxel boundaries that can cause artifacts in shading.

## How it works
**Inputs:**
- Intersection point $P = (P_x, P_y, P_z)$ within the current voxel.
- Scalar field function $T(\cdot)$, implemented as trilinear interpolation over the voxel grid.
- Epsilon $\epsilon$, typically slightly less than half the voxel size.

**Outputs:**
- Normal vector $\mathbf{N} = (N_x, N_y, N_z)$, which is the unnormalized gradient approximation.

**Data Structures:**
- No specific data structures are required beyond the scalar field grid and the trilinear interpolation function $T$. The method relies on accessing neighboring voxel data if the finite difference points fall outside the current voxel.

**Steps:**
1. **Initialize Epsilon:** Set $\epsilon$ to a value slightly less than half the voxel size [S1, unofficial].
2. **Compute Gradient Components:** For each axis $i \in \{x, y, z\}$, calculate the gradient component $N_i$ using the central difference formula:
   $$ N_i = T(P_i + \epsilon) - T(P_i - \epsilon) $$
   where $T$ is the trilinear interpolation function [S1, unofficial].
3. **Handle Boundary Conditions:**
   - If any of the points $(P_i + \epsilon)$ or $(P_i - \epsilon)$ fall outside the current voxel, query the respective neighboring voxel [S1, unofficial].
   - Sample the distance inside that neighbor using trilinear interpolation relative to that neighbor's local coordinates [S1, unofficial].
   - If the neighbor query does not return a result (e.g., at the boundary of the entire grid), return the extrapolated distance from the current voxel [S1, unofficial].
4. **Normalize:** The resulting vector $\mathbf{N}$ is the surface normal. It may be normalized to unit length for shading calculations.

**Pitfalls:**
- The approximation is continuous inside the voxel but not on the edges, which can cause artifacts [S1, unofficial]. Using neighboring voxels as described helps mitigate this.
- Precision issues may arise if $\epsilon$ is too small or too large relative to the voxel size and field variation.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal(P, epsilon, voxel_grid, current_voxel):
    """
    Compute surface normal at point P using finite differences.
    
    Args:
        P: Tuple (Px, Py, Pz) representing the intersection point.
        epsilon: Float, step size for finite differences.
        voxel_grid: The scalar field grid data.
        current_voxel: The voxel containing point P.
        
    Returns:
        Tuple (Nx, Ny, Nz) representing the unnormalized normal.
    """
    # Helper function to get scalar value at a point, handling neighbors
    def get_scalar_value(point):
        # Check if point is within current voxel bounds
        if is_within_voxel(point, current_voxel):
            return trilinear_interpolate(point, current_voxel, voxel_grid)
        else:
            # Determine neighbor voxel
            neighbor = get_neighbor_voxel(point, current_voxel)
            if neighbor is not None:
                # Sample distance inside neighbor using trilinear interpolation
                return trilinear_interpolate(point, neighbor, voxel_grid)
            else:
                # Extrapolate from current voxel if no neighbor available
                return extrapolate_from_voxel(point, current_voxel, voxel_grid)

    # Compute gradient components using central differences
    Nx = get_scalar_value((P[0] + epsilon, P[1], P[2])) - get_scalar_value((P[0] - epsilon, P[1], P[2]))
    Ny = get_scalar_value((P[0], P[1] + epsilon, P[2])) - get_scalar_value((P[0], P[1] - epsilon, P[2]))
    Nz = get_scalar_value((P[0], P[1], P[2] + epsilon)) - get_scalar_value((P[0], P[1], P[2] - epsilon))
    
    return (Nx, Ny, Nz)
```

## Subtopics

### [Surface Normal Calculation via Central Difference](13157-surface-normal-calculation-via-central-difference.md)
Calculates the surface normal $\hat{n}$ at an intersection point $P$ within a scalar field by approximating the gradient $\nabla \rho$ using a central difference scheme. It differs from its parent method by explicitly requiring trilinear interpolation calls to evaluate the scalar field at displaced points, rather than relying on pre-computed or stored gradients.

## Operating characteristics
- **Time Complexity:** $O(1)$ per intersection point, as it requires a fixed number of trilinear interpolation calls (up to 6 points for the central differences, potentially involving neighbor lookups).
- **Memory Complexity:** $O(1)$ additional memory, as it does not require storing additional data structures beyond the scalar field.
- **Strengths:** Provides smooth normals by handling voxel boundaries gracefully, reducing artifacts.
- **Limitations:** Computationally more expensive than using pre-computed gradients due to multiple interpolation calls. Continuity issues at voxel edges if not handled with neighbors.
- **When to Use:** When smooth shading is required and pre-computed gradients are not available or too expensive to store.
- **When Not to Use:** When performance is critical and pre-computed gradients are available, or when the scalar field is already smooth enough that finite differences are unnecessary.

## Background
The method is described in [PSL∗98] and is used in level-of-detail independent voxel-based surface rendering [S1, unofficial]. It addresses the need for accurate surface normals in ray marching or ray casting through scalar fields, where analytical gradients are not available.

## Sources
[S1] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
