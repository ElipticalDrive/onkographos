# Gradient Magnitude Computation

Also called: Magnitude Grid Computation

## Overview
Computes the Euclidean norm $|\mathbf{v}|$ at every active voxel of a vector grid, producing a scalar magnitude grid [S1, unofficial].

## How it works
**Inputs:** A vector grid where each active voxel stores a 3D gradient vector $\mathbf{v} = (v_x, v_y, v_z)$ [S1, unofficial].
**Outputs:** A scalar grid where each active voxel stores the magnitude $|\mathbf{v}|$ [S1, unofficial].

**Data Structures:**
The input is a hierarchical grid tree containing active voxels [S1, unofficial]. The output is a scalar grid with the same topology, active voxel locations, voxel size, and a background value of zero [S1, unofficial].

**Steps:**
1. Iterate over all active voxels $(c, \mathbf{v})$ in the grid tree [S1, unofficial].
2. Compute the Euclidean norm $|\mathbf{v}| = \sqrt{v_x^2 + v_y^2 + v_z^2}$ for each vector [S1, unofficial].
3. Store the computed magnitude in a dictionary keyed by coordinate $c$ [S1, unofficial].
4. Build the result scalar grid using the computed dictionary, a background value of zero, and the original grid's voxel size [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class ScalarGrid:
    def __init__(self, active_values, background_value, voxel_size, topology):
        self.active_values = active_values
        self.background_value = background_value
        self.voxel_size = voxel_size
        self.topology = topology

def compute_gradient_magnitude(input_vector_grid):
    magnitudes = {}
    for coord, gradient_vector in input_vector_grid.active_voxels():
        vx, vy, vz = gradient_vector.x, gradient_vector.y, gradient_vector.z
        magnitude = math.sqrt(vx*vx + vy*vy + vz*vz)
        magnitudes[coord] = magnitude
    output_grid = ScalarGrid(
        active_values=magnitudes,
        background_value=0.0,
        voxel_size=input_vector_grid.voxel_size,
        topology=input_vector_grid.topology
    )
    return output_grid
```

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### Godunov Norm Squared Computation
Computes the squared Godunov norm for gradient components [S2]. It distinguishes itself by handling negative and positive gradient components separately to ensure monotonicity [S2]. It is chosen when solving hyperbolic conservation laws on grids [S2]. It takes a boolean `isOutside` flag, gradient vectors for negative and positive sides (`gradient_m`, `gradient_p`), or individual scalar components (`dP_xm`, `dP_xp`, `dP_ym`, `dP_yp`, `dP_zm`, `dP_zp`) [S2].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
