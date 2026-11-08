# Voxel Interpolation (Nearest, Trilinear, Triquadratic, Tricubic)
Also called: Nearest Neighbor Sampler, SampleFromVoxels, Tricubic Sampler, Trilinear Sampler, Triquadratic Sampler

## Overview
This method provides a unified interface for sampling scalar values from a dense or sparse voxel grid using interpolation orders ranging from 0 (nearest neighbor) to 3 (tricubic). It solves the problem of retrieving continuous values from discrete grid data by supporting integer and floating-point coordinate access. It sets itself apart from its siblings by offering a templated, order-polymorphic API that abstracts the underlying tree structure (OpenVDB/NanoVDB), allowing the same code path to switch between interpolation precisions without changing the caller's logic [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
- Input: A voxel grid (represented as a tree or accessor) and a sample coordinate (integer `ijk` or float `xyz`).
- Output: The interpolated scalar value at the sample coordinate.

**Data Structures:**
The implementation relies on the underlying tree structure (e.g., NanoVDB). The sampler internally manages a cache for active nodes to optimize repeated accesses [S1, unofficial]. The core data access is handled via a read accessor (`cnanovdb_readaccessor`) which tracks the current node level and child pointers [S1, unofficial].

**Parameters:**
- `Order`: Integer specifying the interpolation polynomial degree.
  - 0: Nearest Neighbor
  - 1: Trilinear
  - 2: Triquadratic
  - 3: Tricubic
- `UseCache`: Boolean flag (default `true`) enabling caching of active nodes [S2].

**Steps:**
1. **Initialization**: Create a sampler instance using the factory function `createSampler<Order, TreeT, UseCache>(grid.tree())` or `createSampler<Order>(acc)` where `acc` is a grid accessor [S1, unofficial][S2].
2. **Coordinate Handling**:
   - If the input is an integer coordinate `ijk`, the sampler directly maps it to the grid cell.
   - If the input is a floating-point coordinate `xyz`, the sampler calculates the fractional offsets within the voxel.
3. **Node Traversal**: For sparse grids, the sampler traverses the tree hierarchy. It checks child masks (`mChildMask`) and value masks (`mValueMask`) to locate the active leaf node containing the sample point [S1, unofficial].
4. **Interpolation**:
   - **Order 0 (Nearest)**: Returns the value of the active voxel closest to the coordinate.
   - **Order 1 (Trilinear)**: Computes the weighted average of the 8 corner values of the enclosing voxel using linear weights based on fractional distances [S1, unofficial].
   - **Order 2 (Triquadratic)**: Uses 27 points (3x3x3 grid) to fit a quadratic polynomial, providing smoother gradients than trilinear [S1, unofficial].
   - **Order 3 (Tricubic)**: Uses 64 points (4x4x4 grid) to fit a cubic polynomial, offering the highest smoothness [S1, unofficial].
5. **Boundary Handling**: If the sample point lies outside the defined volume bounds, the method returns 0 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the sampling interface
# Note: The actual interpolation math (weights, fetches) is encapsulated in the library
# based on the Order template parameter.

class VoxelSampler:
    def __init__(self, grid, order, use_cache=True):
        self.grid = grid
        self.order = order
        self.use_cache = use_cache
        self.accessor = grid.get_accessor()
        # Internal cache structure for active nodes
        self.cache = {} if use_cache else None

    def sample_int(self, ijk):
        """Sample at integer coordinate ijk."""
        # Map ijk to grid value
        # If sparse, traverse tree to find active leaf
        # Return value or 0 if inactive/out of bounds
        return self._get_value_at_ijk(ijk)

    def sample_float(self, xyz):
        """Sample at floating-point coordinate xyz."""
        # Determine enclosing voxel indices
        ijk_base = floor(xyz)
        # Calculate fractional offsets
        frac = xyz - ijk_base
        
        # Fetch neighbor values based on Order
        if self.order == 0:
            return self._get_value_at_ijk(ijk_base)
        elif self.order == 1:
            # Trilinear: 8 neighbors
            v000 = self._get_value_at_ijk(ijk_base)
            v100 = self._get_value_at_ijk(ijk_base + (1,0,0))
            v010 = self._get_value_at_ijk(ijk_base + (0,1,0))
            v001 = self._get_value_at_ijk(ijk_base + (0,0,1))
            v110 = self._get_value_at_ijk(ijk_base + (1,1,0))
            v101 = self._get_value_at_ijk(ijk_base + (1,0,1))
            v011 = self._get_value_at_ijk(ijk_base + (0,1,1))
            v111 = self._get_value_at_ijk(ijk_base + (1,1,1))
            # Linear interpolation weights
            w = frac
            # Trilinear formula
            return self._trilinear(v000, v100, v010, v001, v110, v101, v011, v111, w)
        elif self.order == 2:
            # Triquadratic: 27 neighbors
            # Fetch 3x3x3 grid of values
            # Fit quadratic polynomial
            return self._triquadratic(...) # Implementation details omitted [G]
        elif self.order == 3:
            # Tricubic: 64 neighbors
            # Fetch 4x4x4 grid of values
            # Fit cubic polynomial
            return self._tricubic(...) # Implementation details omitted [G]

    def _get_value_at_ijk(self, ijk):
        # Handle boundary checks
        if not self.grid.is_inside(ijk):
            return 0.0
        # Access tree node
        # Check active mask
        # Return value
        pass

    def _trilinear(self, v000, v100, v010, v001, v110, v101, v011, v111, w):
        # Standard trilinear interpolation
        # w is Vec3f (fx, fy, fz)
        # ...
        pass
```

**Edge Cases:**
- **Out of Bounds**: Coordinates outside the grid dimensions return 0 [S1, unofficial].
- **Inactive Voxels**: In sparse grids, if the coordinate falls within an inactive region, the result is 0 [S1, unofficial].
- **Precision**: Floating-point sampling requires careful handling of fractional offsets to avoid artifacts at voxel boundaries [S1, unofficial].

## Subtopics

### [Tricubic B-Spline Interpolation](8246-tricubic-b-spline-interpolation.md)
A GPU-accelerated variant using uniform cubic B-splines. It provides smoother results than linear interpolation while being faster than full tricubic convolution. It sets itself apart by using optimized weight grouping and coordinate transformation to reduce texture fetches and arithmetic operations, making it suitable for real-time volume rendering on WebGL-compatible hardware.

### Voxel Sampler Factory
A factory function that creates a voxel sampler of a specific polynomial order. It simplifies instantiation by allowing compact syntax like `auto smp = nanovdb::math::createSampler<1>(acc)`. It supports a `UseCache` boolean flag (default `true`) to enable caching of active nodes for performance [S2].

## Operating characteristics
- **Time Complexity**: Depends on the interpolation order and grid sparsity. Nearest neighbor is $O(1)$ for dense grids. Trilinear is $O(1)$ with 8 fetches. Triquadratic and Tricubic involve more fetches (27 and 64 respectively) and more arithmetic operations. Sparse grids add tree traversal overhead.
- **Memory Complexity**: The sampler itself has minimal memory footprint. The `UseCache` option may increase memory usage by storing references to active nodes [S2].
- **Performance**: Trilinear is the standard for real-time rendering due to its balance of speed and smoothness. Tricubic offers higher quality but is computationally expensive. B-spline variants offer a middle ground optimized for GPU hardware [S1, unofficial].
- **Strengths**: Unified API for multiple interpolation orders; supports both integer and floating-point coordinates; integrates with sparse grid structures (OpenVDB/NanoVDB) [S1, unofficial][S2].
- **Limitations**: Higher-order interpolations are computationally intensive; boundary handling returns 0, which may cause artifacts at edges [S1, unofficial].

## Background
The method is part of the NanoVDB/OpenVDB ecosystem, which is widely used in visual effects and scientific visualization for storing large sparse voxel grids. The templated sampler design allows for flexible integration into various rendering pipelines [S1, unofficial][S2].

## Sources
[S1] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S2] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
