# Voxel Interpolation (Nearest, Trilinear, Triquadratic, Tricubic)


## Overview
Voxel interpolation samples values from a dense 3D volume grid at arbitrary floating-point coordinates using polynomial basis functions of varying orders (0 to 3) [S2, unofficial]. It solves the problem of converting discrete voxel data into continuous scalar fields for ray casting or feature extraction. It sets itself apart from its parent [Voxel Trilinear Interpolation (SIMD)](2564-voxel-trilinear-interpolation-simd.md) by supporting higher-order continuity (quadratic and cubic) and providing a generic templated interface for various tree types (e.g., OpenVDB/NanoVDB) rather than being restricted to SIMD-optimized trilinear kernels [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volume grid (dense or sparse tree structure) and a set of query coordinates $\mathbf{xyz} = (x, y, z)$ in world or grid space.
- Output: The interpolated scalar value $V$ at $\mathbf{xyz}$.

**Data Structures:**
The implementation relies on an accessor object `acc` that provides random access to voxel values. For dense grids, this is a direct memory pointer or stride-based index. For sparse grids (NanoVDB), the accessor traverses the tree hierarchy to locate the leaf node containing the query point [S2, unofficial][S3].

**Interpolation Orders:**
The method supports four orders [S2, unofficial]:
- Order 0: Nearest-neighbor.
- Order 1: Trilinear (parent method).
- Order 2: Triquadratic.
- Order 3: Tricubic.

**Steps:**
1. **Coordinate Normalization (Optional):** If the input coordinates are in a bounding box $[bbox_{min}, bbox_{max}]$, normalize them to $[-1, 1]$ for compatibility with frameworks like PyTorch [S1, unofficial].
   $$
   \text{center} = \frac{bbox_{min} + bbox_{max}}{2}
   $$
   $$
   \text{extent} = \frac{bbox_{max} - bbox_{min}}{2}
   $$
   $$
   xyz_{norm} = \frac{xyz - \text{center}}{\text{extent}}
   $$
2. **Integer and Fractional Decomposition:** For a query coordinate $x$, compute the integer grid index $i = \lfloor x \rfloor$ and the fractional offset $t = x - i$.
3. **Neighbor Fetching:** Retrieve the voxel values surrounding the query point based on the interpolation order.
   - Order 0: Fetch $V(i, j, k)$.
   - Order 1: Fetch 8 neighbors $(i+\delta_x, j+\delta_y, k+\delta_z)$ where $\delta \in \{0, 1\}$.
   - Order 2: Fetch 27 neighbors ($\delta \in \{-1, 0, 1\}$).
   - Order 3: Fetch 64 neighbors ($\delta \in \{-1, 0, 1\}$ for cubic B-spline or convolution kernel support).
4. **Weight Calculation:** Compute interpolation weights based on the fractional offset $t$ and the chosen basis function (linear, quadratic, or cubic B-spline) [S2, unofficial].
5. **Accumulation:** Sum the product of fetched values and weights.
   $$
   V = \sum_{\delta_x, \delta_y, \delta_z} V(i+\delta_x, j+\delta_y, k+\delta_z) \cdot W_x(\delta_x, t_x) \cdot W_y(\delta_y, t_y) \cdot W_z(\delta_z, t_z)
   $$
6. **Boundary Handling:** If any neighbor index is outside the grid bounds, return 0 or a specified boundary value [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_voxel(grid_accessor, xyz, order):
    """
    Samples a voxel grid at xyz with specified interpolation order.
    grid_accessor: Object with method get_value(ijk) returning float.
    xyz: Tuple (x, y, z) of floats.
    order: Integer 0, 1, 2, or 3.
    """
    # 1. Decompose coordinates
    ix, fx = math.floor(xyz[0]), xyz[0] - math.floor(xyz[0])
    iy, fy = math.floor(xyz[1]), xyz[1] - math.floor(xyz[1])
    iz, fz = math.floor(xyz[2]), xyz[2] - math.floor(xyz[2])

    # 2. Determine neighbor offsets based on order
    if order == 0:
        offsets = [(0, 0, 0)]
    elif order == 1:
        offsets = [(dx, dy, dz) for dx in (0, 1) for dy in (0, 1) for dz in (0, 1)]
    elif order == 2:
        offsets = [(dx, dy, dz) for dx in (-1, 0, 1) for dy in (-1, 0, 1) for dz in (-1, 0, 1)]
    elif order == 3:
        offsets = [(dx, dy, dz) for dx in (-1, 0, 1) for dy in (-1, 0, 1) for dz in (-1, 0, 1)]
    else:
        raise ValueError("Order must be 0, 1, 2, or 3")

    # 3. Compute weights
    # Note: Specific weight functions W(t) depend on the basis (Linear, Quadratic, Cubic B-Spline).
    # For Order 1 (Trilinear), weights are simply (1-t) and t.
    # For higher orders, standard B-spline or Lagrange coefficients are used [S2, unofficial].
    
    total_value = 0.0
    
    for dx, dy, dz in offsets:
        # Calculate local fractional coordinates relative to the corner
        # For order > 1, the relationship between 't' and the basis function argument 
        # depends on the specific implementation (e.g., B-spline vs Convolution) [G].
        
        # Fetch value
        ijk = (ix + dx, iy + dy, iz + dz)
        
        # Boundary check
        if not grid_accessor.is_valid(ijk):
            value = 0.0 # [G] Default to 0 for out-of-bounds
        else:
            value = grid_accessor.get_value(ijk)
            
        # Weight calculation placeholder: In a real implementation, 
        # W_x, W_y, W_z are computed using the specific polynomial basis for the order.
        wx = compute_weight_1d(fx, dx, order)
        wy = compute_weight_1d(fy, dy, order)
        wz = compute_weight_1d(fz, dz, order)
        
        total_value += value * wx * wy * wz

    return total_value

def compute_weight_1d(t, offset, order):
    """
    Computes 1D interpolation weight.
    t: fractional part [0, 1).
    offset: integer offset from floor(t).
    order: 0, 1, 2, 3.
    """
    if order == 0:
        return 1.0 if offset == 0 else 0.0
    elif order == 1:
        # Trilinear
        if offset == 0:
            return 1.0 - t
        else:
            return t
    elif order == 2:
        # Triquadratic (Lagrange or B-spline specific)
        # Standard quadratic B-spline basis functions centered at integer knots
        # This is a simplified representation; exact coefficients depend on the specific spline definition [G]
        if offset == -1:
            return 0.5 * t * t
        elif offset == 0:
            return 0.75 - t * t
        elif offset == 1:
            return 0.5 * (1.0 - t) * (1.0 - t)
        else:
            return 0.0
    elif order == 3:
        # Tricubic B-Spline
        # Standard uniform cubic B-spline basis functions
        if offset == -1:
            return (1.0/6.0) * (-t*t*t + 3*t*t - 3*t + 1)
        elif offset == 0:
            return (1.0/6.0) * (3*t*t*t - 6*t*t + 4)
        elif offset == 1:
            return (1.0/6.0) * (-3*t*t*t + 3*t*t + 3*t + 1)
        elif offset == 2:
            return (1.0/6.0) * (t*t*t)
        else:
            return 0.0
    return 0.0
```

**Parameters:**
- `Order`: Interpolation order. 0=Nearest, 1=Trilinear, 2=Triquadratic, 3=Tricubic [S2, unofficial].
- `TreeT`: The tree type (e.g., OpenVDB/NanoVDB tree) determining the memory layout and accessor interface [S2, unofficial].
- `UseCache`: Boolean flag for caching node lookups in sparse grids, defaults to true [S3].

**Edge Cases:**
- **Out of Bounds:** If the query point lies outside the grid, the accessor must handle boundary conditions. The default behavior in many implementations is to return 0 [S2, unofficial].
- **Precision:** Floating-point precision errors in coordinate decomposition can lead to fetching incorrect neighbors if $t$ is very close to 0 or 1. Guard against this by clamping $t$ to $[0, 1)$ [G].

## Subtopics

### [Tricubic B-Spline Interpolation](8246-tricubic-b-spline-interpolation.md)
A GPU-accelerated method for sampling 3D volume textures using uniform cubic B-splines, providing smoother results than linear interpolation while being faster than full tricubic convolution. It sets itself apart from standard tricubic convolution by reducing the number of texture fetches to eight per sample through grouped weight calculations and optimized coordinate mapping, trading a moderate increase in arithmetic operations for improved smoothness over trilinear interpolation.

### [SDF Grid Sampling](12658-sdf-grid-sampling.md)
SDF Grid Sampling evaluates a Signed Distance Field (SDF) at arbitrary 3D query points by storing SDF values directly in the cells of a discretized volume grid and interpolating between them. It solves the problem of querying continuous geometric surfaces from explicit voxel representations, serving as a baseline for neural implicit surface reconstruction. Unlike [Tricubic B-Spline Interpolation](8246-tricubic-b-spline-interpolation.md) or [Multi-resolution Interpolation](13755-multi-resolution-interpolation.md), which use learned latent codes or hierarchical structures to enforce smoothness, the standard dense SDF grid stores raw SDF values independently per cell, leading to potential noise without additional regularization. It builds on [Voxel Interpolation (Nearest, Trilinear, Triquadratic, Tricubic)](6486-voxel-interpolation-nearest-trilinear-triquadratic-tricubic.md) by applying these interpolation primitives specifically to signed distance values.

### [Bilinear Sampler](12826-bilinear-sampler.md)
The Bilinear Sampler is a differentiable operation that samples values from a dense grid (image or volume) at arbitrary floating-point coordinates using bilinear (2D) or trilinear (3D) interpolation. It serves as the bridge between continuous coordinate spaces (such as ray-casting outputs or feature point locations) and discrete tensor representations. It differs from its parent [Voxel Interpolation](6486-voxel-interpolation-nearest-trilinear-triquadratic-tricubic.md) by being a specific, optimized implementation of order-1 interpolation, typically exposed via deep learning frameworks (e.g., PyTorch's `F.grid_sample`) with specific coordinate normalization conventions (`align_corners`) and padding modes.

### [Multi-resolution Interpolation](13755-multi-resolution-interpolation.md)
Multi-resolution interpolation is a technique for sampling volumetric data stored in hierarchical structures (such as Sparse Voxel Octrees) where adjacent blocks may have different resolutions. It solves the problem of interpolating values across block boundaries where simple trilinear interpolation fails due to non-uniform voxel spacing. It sets itself apart from standard [Voxel Interpolation](6486-voxel-interpolation-nearest-trilinear-triquadratic-tricubic.md) by dynamically adjusting the sampling grid to the finest common resolution among neighbors, ensuring smooth transitions between Level-of-Detail (LoD) boundaries.

### Voxel Sampler Factory
Factory function that creates a voxel sampler of a specific polynomial order [S3]. It allows for compact syntax in C++ implementations like NanoVDB.
- **Parameters:**
  - `Order`: Polynomial order for the sampler [S3].
  - `UseCache`: Boolean flag, defaults to true [S3].
- **Code Example:**
  ```cpp
  auto acc = grid.getAccessor();
  auto smp = nanovdb::math::createSampler<1>( acc );
  ```
  [S3]

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample for dense grids. For sparse grids, $O(\log N)$ or $O(depth)$ depending on tree traversal efficiency [S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory per sample.
- **Performance:** Trilinear is the fastest and most common. Triquadratic and Tricubic offer smoother gradients but require more memory fetches and arithmetic operations. GPU acceleration (e.g., via B-Spline optimizations) can mitigate the cost of higher orders [S2, unofficial].
- **Strengths:** Supports multiple continuity orders; generic interface for various grid types (dense/sparse).
- **Limitations:** Higher orders increase computational cost. Boundary handling must be explicit.
- **When to use:** Use Nearest for speed/low quality. Use Trilinear for standard volume rendering. Use Triquadratic/Tricubic for high-quality surface reconstruction or smooth field visualization.

## Background
The concept of voxel interpolation is fundamental to volume rendering and scientific visualization. The specific templated class implementation supporting orders 0-3 for NanoVDB/OpenVDB trees originates from the NanoVDB library developed for high-performance volume rendering [S2, unofficial]. The PyTorch integration (`F.grid_sample`) popularized the normalization conventions and differentiable sampling interface for deep learning applications [S1, unofficial]. The factory pattern for creating samplers is part of the NanoVDB math namespace [S3].

## Sources
[S1] zzilch/Mesh2Volume, https://github.com/zzilch/Mesh2Volume, fetched 2026-10-26 (unofficial)
[S2] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S3] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
