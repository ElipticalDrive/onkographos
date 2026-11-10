# Gradient Magnitude Computation

Also called: Magnitude Grid Computation, Magnitude de gradiente

## Overview
Gradient Magnitude Computation derives a scalar volumetric attribute that measures the local rate of intensity variation by associating a value with the neighborhood of volumetric cells [S1, unofficial]. It serves as a critical input for Transfer Functions (TFs), enabling the identification of edges and surfaces in addition to monotonic regions [S1, unofficial]. Unlike the parent method which outputs a vector field, this method outputs a scalar field representing the Euclidean norm of the gradient, allowing TFs to distinguish high-contrast boundaries from smooth interior regions [S1, unofficial].

## How it works
**Inputs:**
1.  A scalar volume $f$ (intensity distribution) or a precomputed 3D gradient vector field $\nabla f$ from [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md).
2.  Grid topology information (active voxels, voxel size).

**Outputs:**
1.  A scalar grid (magnitude grid) where each active voxel contains the computed gradient magnitude.
2.  Background value of zero.

**Data Structures:**
*   **Input Grid:** A sparse or dense grid structure containing active voxels. Each active voxel $(c, v)$ holds a 3D vector $\vec{v}$ (the gradient) or scalar $f$ (if computing from scratch, though typically this step follows gradient computation).
*   **Output Grid:** A dictionary or sparse grid keyed by coordinate $(x, y, z)$, storing a single scalar value (float/double).

**Steps:**
1.  **Iterate:** Traverse all active voxels in the input grid. Let the current voxel coordinate be $ijk$.
2.  **Retrieve Gradient:** Obtain the gradient vector $\vec{v} = (v_x, v_y, v_z)$ for voxel $ijk$. If the input is a scalar field $f$, compute $\vec{v}$ using finite differences as defined in [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) [S1, unofficial].
3.  **Compute Norm:** Calculate the Euclidean norm of $\vec{v}$:
    $$ M(ijk) = \|\vec{v}\| = \sqrt{v_x^2 + v_y^2 + v_z^2} $$
    This corresponds to the formal expression $f' = \|\nabla f\|$ [S1, unofficial].
4.  **Store:** Insert the scalar magnitude $M(ijk)$ into the result grid at key $ijk$.
5.  **Finalize:** Construct the final magnitude grid using the computed data, the original grid's voxel size, and a background value of 0.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_magnitude_grid(input_gradient_grid):
    """
    Computes the gradient magnitude scalar grid from a vector gradient grid.
    
    Args:
        input_gradient_grid: A grid structure where active voxels contain Vec3 gradient vectors.
        
    Returns:
        magnitude_grid: A scalar grid with the same topology, containing Euclidean norms.
    """
    magnitude_grid = {} # Dictionary or sparse grid structure
    
    # Iterate over all active voxels in the input grid
    for voxel_coord, gradient_vector in input_gradient_grid.active_voxels():
        # Extract components
        vx = gradient_vector.x
        vy = gradient_vector.y
        vz = gradient_vector.z
        
        # Compute Euclidean norm: sqrt(vx^2 + vy^2 + vz^2)
        magnitude = (vx * vx + vy * vy + vz * vz) ** 0.5
        
        # Store in result grid
        magnitude_grid[voxel_coord] = magnitude
        
    # Build the final grid object with background value 0
    # and original voxel size from input_gradient_grid
    result = build_scalar_grid(magnitude_grid, 
                               background_value=0.0, 
                               voxel_size=input_gradient_grid.voxel_size)
    return result
```

**Edge Cases and Pitfalls:**
*   **Discrete Approximation:** Analytical calculation of the gradient is not always possible because volumes are formed by discrete elements; values are approximated using finite difference operators [S1, unofficial]. Errors in the underlying gradient computation propagate directly into the magnitude.
*   **Insufficiency for Overlap:** Gradient magnitude alone is not always sufficient to distinguish edges from overlapping volumetric regions [S1, unofficial]. In such cases, higher-order derivatives (e.g., second derivative) may be required.
*   **Zero Gradient:** Voxels in uniform regions yield a magnitude of 0. Ensure the transfer function handles this correctly to avoid artifacts in smooth areas.

## Subtopics

### Godunov Norm Squared Computation
Computes the squared Godunov norm for gradient components, often used in specific grid-based solvers or VDB implementations [S3]. It distinguishes itself by handling negative and positive gradient sides separately and checking for outside regions.
*   **Definition:** Computes $\|\nabla f\|_{Godunov}^2$ using signed gradient components.
*   **Parameters:**
    *   `isOutside`: Boolean flag indicating if the voxel is in an outside region [S3].
    *   `gradient_m`, `gradient_p`: Vec3 gradient components for negative and positive sides [S3].
    *   `dP_xm`, `dP_xp`, `dP_ym`, `dP_yp`, `dP_zm`, `dP_zp`: Scalar gradient components for negative and positive sides in x, y, z directions [S3].
*   **When to choose:** Use when implementing Godunov-type schemes or when working with OpenVDB's `nanovdb::math` utilities [S3]. It provides a robust norm estimation that accounts for directional asymmetry in the gradient field.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of active voxels. The operation is a simple vector norm calculation per voxel.
*   **Memory Complexity:** $O(N)$ for the output scalar grid.
*   **Performance:** Extremely fast, typically limited by memory bandwidth rather than computation. Can be parallelized trivially across voxels.
*   **Strengths:** Simple to implement; provides essential edge information for transfer functions; low computational overhead.
*   **Limitations:** Sensitive to noise in the underlying gradient data; may not distinguish overlapping structures without additional attributes (like second derivatives) [S1, unofficial].
*   **When to use:** As a standard step in volume rendering pipelines to enhance edge visibility in transfer functions.
*   **When not to use:** When the volume is extremely noisy and smoothing is required first; or when overlapping structures need to be resolved without higher-order derivatives.

## Background
Gradient magnitude is a fundamental concept in volumetric data analysis, derived from the directional derivative of the intensity function [S1, unofficial]. It has been widely used in volume visualization to define transfer functions that map scalar values to opacity and color, emphasizing boundaries [S1, unofficial]. The approach is standard in scientific visualization, particularly when using sparse grid structures like VDB, where efficient computation of derived attributes is crucial [S3].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S3] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
