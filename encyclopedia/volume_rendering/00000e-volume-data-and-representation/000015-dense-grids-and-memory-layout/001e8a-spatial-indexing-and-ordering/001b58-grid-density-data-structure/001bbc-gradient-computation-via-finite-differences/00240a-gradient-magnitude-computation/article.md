# Gradient Magnitude Computation

Also called: Magnitude Grid Computation, Magnitude de gradiente

## Overview
Gradient Magnitude Computation derives a scalar volumetric attribute that measures the local rate of intensity variation by associating a value with the neighborhood of volumetric cells [S1, unofficial]. It enables Transfer Functions to identify edges and surfaces in addition to monotonic regions [S1, unofficial].

## How it works
**Inputs:**
1. A vector grid containing 3D gradient vectors $\vec{v}$ at each active voxel, typically produced by [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) [S2, unofficial].
2. Grid topology information (active voxels, voxel size) [G].

**Outputs:**
1. A scalar grid (magnitude grid) where each active voxel contains the computed gradient magnitude [S2, unofficial].
2. Background value of zero [S2, unofficial].

**Data Structures:**
* Input grid tree with active voxels $(c, v)$ holding 3D vectors $\vec{v}$ [S2, unofficial].
* Output dictionary or sparse grid keyed by coordinate $(x, y, z)$, storing a single scalar value [S2, unofficial].

**Steps:**
1. Iterate over all active voxels $(c, v)$ in the grid tree [S2, unofficial].
2. Compute the Euclidean norm $|\vec{v}|$ of the vector $\vec{v}$ at each voxel [S2, unofficial].
3. Store the computed norm in a dictionary keyed by coordinate [S2, unofficial].
4. Build the result magnitude grid using the computed data, a background value of zero, and the original grid's voxel size [S2, unofficial].

**Formula:**
The gradient attribute is formally expressed as the directional derivative of the intensity function $f$:
$$f' = \|\nabla f\| \quad (3.3)$$
Here, $f$ is the intensity distribution, $\nabla f$ is the gradient vector, and $f'$ is the gradient magnitude [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z

class Grid:
    def __init__(self, voxels, voxel_size):
        self.voxels = voxels  # dict mapping coord to Vec3
        self.voxel_size = voxel_size
    def active_voxels(self):
        return self.voxels.items()

def compute_gradient_magnitude(input_grid):
    magnitude_dict = {}
    for coord, vec in input_grid.active_voxels():
        magnitude_dict[coord] = (vec.x**2 + vec.y**2 + vec.z**2) ** 0.5
    return ScalarGrid(magnitude_dict, 0.0, input_grid.voxel_size)

class ScalarGrid:
    def __init__(self, data, background_value, voxel_size):
        self.data = data
        self.background_value = background_value
        self.voxel_size = voxel_size
```

**Edge Cases and Pitfalls:**
* Analytical calculation of the gradient is not always possible because volumes are formed by discrete elements; values are approximated using finite difference operators [S1, unofficial].
* Gradient magnitude alone is not always sufficient to distinguish edges from overlapping volumetric regions [S1, unofficial]. In such cases, higher-order derivatives (e.g., second derivative) may be required [S1, unofficial].
* Voxels in uniform regions yield a magnitude of 0 [G].

## Subtopics

### Godunov Norm Squared Computation
Computes the squared Godunov norm for gradient components [S3]. It handles negative and positive gradient sides separately and checks for outside regions [S3].
*   **Definition:** Computes $\|\nabla f\|_{Godunov}^2$ using signed gradient components [S3].
*   **Parameters:** `isOutside`: Boolean flag indicating outside region [S3]. `gradient_m`, `gradient_p`: Vec3 gradient components for negative and positive sides [S3]. `dP_xm`, `dP_xp`, `dP_ym`, `dP_yp`, `dP_zm`, `dP_zp`: Scalar gradient components for negative and positive sides in x, y, z directions [S3].
*   **When to choose:** Not given in the sources.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of active voxels [G].
*   **Memory Complexity:** $O(N)$ for the output scalar grid [G].
*   **Performance:** The operation is a simple vector norm calculation per voxel [S2, unofficial].
*   **Strengths:** Simple to implement; provides essential edge information for transfer functions [S1, unofficial].
*   **Limitations:** May not distinguish overlapping structures without additional attributes like second derivatives [S1, unofficial].
*   **When to use:** As a standard step in volume rendering pipelines to enhance edge visibility in transfer functions [G].
*   **When not to use:** When overlapping structures need to be resolved without higher-order derivatives [S1, unofficial].

## Background
Gradient magnitude is a fundamental concept in volumetric data analysis, derived from the directional derivative of the intensity function [S1, unofficial]. It has been used in volume visualization to define transfer functions that map scalar values to opacity and color, emphasizing boundaries [S1, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S3] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
