# Nearest Voxel Lookup with Periodic Boundary

Also called: Grid Probing, Nearest Voxel Lookup, Nearest-Neighbor Density Lookup, Periodic Boundary Voxel Lookup, Wrapped Nearest Neighbor Sampling

## Overview
Nearest Voxel Lookup retrieves the scalar value (e.g., density) of the single grid voxel containing or nearest to a continuous sample point, without interpolating neighboring values [S1, S2, unofficial]. It solves the problem of mapping continuous world-space coordinates to discrete memory indices in a dense volume grid. Unlike [Interpolated Discrete Values & Gradients](1268-interpolated-discrete-values-gradients.md), which blends adjacent voxels for smoothness, this method produces a piecewise-constant field that preserves sharp discontinuities but introduces aliasing artifacts [S2, unofficial]. It differs from [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md) by returning a single value rather than a set of corner indices, and from [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) by using zero-order approximation instead of higher-order continuity.

## How it works
**Inputs:**
- `p`: Sample point in world space (3D vector).
- `grid.bounds`: Array of two 3D vectors `[min, max]` defining the axis-aligned bounding box of the volume [S1, unofficial].
- `grid.baseResolution`: Integer resolution of the grid (assumed cubic, $N \times N \times N$) [S1, unofficial].
- `grid.density`: 1D array storing voxel values in row-major order [S1, unofficial].

**Outputs:**
- `value`: Scalar density at the nearest voxel.
- `out_of_bounds`: Boolean flag indicating if the sample point fell outside the grid bounds and was wrapped [S2, unofficial].

**Data Structures:**
- The grid is stored as a flat 1D array of size $N^3$.
- Index mapping for voxel $(x, y, z)$ is: `idx = (z * N + y) * N + x` [S1, unofficial].

**Steps:**
1. **Transform to Local Space:** Subtract the grid minimum from the sample point to shift the coordinate system origin to the grid's lower-left corner [S1, unofficial].
   $$ p_{local} = p - \text{grid.bounds}[0] $$
2. **Normalize:** Divide by the grid's physical size to map coordinates to the range $[0, 1]$ [S1, unofficial].
   $$ \text{gridSize} = \text{grid.bounds}[1] - \text{grid.bounds}[0] $$
   $$ p_{norm} = \frac{p_{local}}{\text{gridSize}} $$
3. **Convert to Voxel Space:** Multiply normalized coordinates by the grid resolution to get floating-point voxel coordinates [S1, unofficial].
   $$ p_{voxel} = p_{norm} \times \text{grid.baseResolution} $$
4. **Quantize to Integer Indices:**
   - *Standard Approach:* Use `floor` to truncate towards negative infinity. This ensures that coordinates slightly less than 0 (due to floating-point error or out-of-bounds rays) map to index 0 rather than -1, preventing invalid memory access [S1, unofficial].
     $$ x_i = \lfloor p_{voxel}.x \rfloor, \quad y_i = \lfloor p_{voxel}.y \rfloor, \quad z_i = \lfloor p_{voxel}.z \rfloor $$
   - *Periodic Boundary Approach:* If periodic boundaries are required, indices are computed by rounding to the nearest integer (or floor) and then applying modulo arithmetic with the grid resolution to wrap out-of-bounds indices [S2, unofficial].
     $$ x_i = \lfloor p_{voxel}.x \rfloor \pmod N $$
     $$ y_i = \lfloor p_{voxel}.y \rfloor \pmod N $$
     $$ z_i = \lfloor p_{voxel}.z \rfloor \pmod N $$
5. **Clamp/Check Bounds:** Ensure indices do not exceed $N-1$. In standard lookup, if $p_{norm}$ is exactly 1.0, $p_{voxel}$ becomes $N$, causing an out-of-bounds access. This must be clamped to $N-1$ [S1, unofficial]. In periodic lookup, the modulo operation handles this naturally [S2, unofficial].
6. **Lookup:** Retrieve the value from the 1D array using the flattened index [S1, unofficial].
   $$ \text{value} = \text{grid.density}[(z_i \times N + y_i) \times N + x_i] $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nearest_voxel_lookup_periodic(grid, p):
    """
    Performs nearest voxel lookup with periodic boundary conditions.
    
    Args:
        grid: Object containing bounds, baseResolution, and density array.
        p: World space point (Vec3).
        
    Returns:
        value: Density at the nearest voxel.
        out_of_bounds: Boolean, true if p was outside [0,1] normalized space before wrapping.
    """
    # 1. Transform to local space
    grid_size = grid.bounds[1] - grid.bounds[0]
    p_local = p - grid.bounds[0]
    
    # 2. Normalize
    p_norm = p_local / grid_size
    
    # 3. Convert to voxel space
    p_voxel = p_norm * grid.baseResolution
    
    # 4. Check for out-of-bounds before wrapping
    # A point is out of bounds if any coordinate is < 0 or >= 1 in normalized space
    out_of_bounds = (p_norm.x < 0) or (p_norm.x >= 1) or \
                    (p_norm.y < 0) or (p_norm.y >= 1) or \
                    (p_norm.z < 0) or (p_norm.z >= 1)
    
    # 5. Compute integer indices with floor
    xi = floor(p_voxel.x)
    yi = floor(p_voxel.y)
    zi = floor(p_voxel.z)
    
    # 6. Apply periodic boundary conditions (modulo)
    N = grid.baseResolution
    xi = xi % N
    yi = yi % N
    zi = zi % N
    
    # 7. Lookup value
    idx = (zi * N + yi) * N + xi
    return grid.density[idx], out_of_bounds
```

**Parameters:**
- `grid.baseResolution`: Typical values depend on the application (e.g., 64, 128, 256). Must be consistent with the memory layout [S1, unofficial].
- `grid.bounds`: Defines the physical extent. Must be axis-aligned [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Negative Coordinates:** Using `static_cast<int>` (truncation) on negative floating-point values returns -1, leading to invalid memory access. `floor` must be used to return 0 for values in $[-1, 0)$ [S1, unofficial].
- **Exact Boundary (Value 1.0):** If a normalized coordinate is exactly 1.0, `floor` yields $N$, which is out of bounds for an array of size $N$. The index must be clamped to $N-1$ or the input must be handled to ensure it stays within $[0, 1)$ [S1, unofficial].
- **Periodic Wrapping:** Modulo of negative numbers in some languages (e.g., C++) returns negative results. Ensure the modulo operation yields positive indices in $[0, N-1]$ (e.g., `(x % N + N) % N`) [S2, G, unofficial].

## Subtopics

### Nearest Integer Rounding (Floor)
Nearest Integer Rounding converts continuous floating-point coordinates into discrete integer grid indices by rounding to the closest integer value. It serves as the fundamental coordinate quantization step in volume rendering, mapping sample points to voxel centers or corners. Unlike floor-based indexing which truncates towards zero or negative infinity, this method minimizes spatial quantization error by selecting the nearest grid node, which is critical for accurate gradient estimation and high-fidelity reconstruction. It differs from [Ceil](1574-ceil.md) which always rounds up, and [Floor](1575-floor.md) which always rounds down, by balancing bias in both directions [Child Note].

### [Nearest Neighbor Interpolation](5891-nearest-neighbor-interpolation.md)
Nearest Neighbor Interpolation estimates the value at a continuous sample point by retrieving the value of the single grid voxel whose center is closest to that point. It serves as the simplest form of spatial sampling in volume rendering, providing a computationally efficient but low-fidelity approximation of the underlying scalar field. Unlike higher-order methods such as trilinear interpolation, it does not blend neighboring values, resulting in a piecewise-constant representation that preserves sharp discontinuities but introduces aliasing artifacts.

### Nearest Neighbor Search Interpolation
This variant uses a KD-tree to find grid vertices near a query point within a radius $w$, then solves for coefficients using a weighted least-squares approach with degree 2 basis functions. It initializes implicit function values to zeros, constructs a KD-tree from constraint points, and for each grid vertex, performs a range nearest neighbor search. If no neighbors are found, the value is set to 100.0. If neighbors exist, it extracts positions and values, generates a weight matrix $W$ and basis matrix $B$, and solves for coefficients $a$ using LLT or LDLT decomposition ($R = B^T W D$, $a = L^{-1} R$). The final value is computed as $f_x(i) = g_i \cdot a$, where $g_i$ is the basis vector for the query point [S3, unofficial]. Parameter $w$ is the search radius [S3, unofficial].

### Nearest Integer Rounding (Floor)
This variant specifically rounds a 3D vector down to the nearest integer coordinates. It takes an input vector `xyz` of type `Vec3<RealT>` and returns the floored integer coordinates. This is a specific implementation of the quantization step used to prevent negative indices [S4].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample point. The lookup involves a constant number of arithmetic operations and a single array access [S1, unofficial].
- **Memory Complexity:** $O(N^3)$ for the dense grid storage, where $N$ is the resolution [S1, unofficial].
- **Performance:** Extremely fast due to cache-friendly 1D array access and lack of interpolation logic [S1, unofficial].
- **Strengths:** Computational efficiency; preserves sharp edges and discontinuities without blurring [S2, unofficial].
- **Limitations:** Produces blocky, aliased visuals (pixelation) because it does not smooth between voxels [S2, unofficial]. Sensitive to grid alignment; shifting the grid relative to the camera can cause visible artifacts.
- **When to use:** High-performance requirements, low-resolution grids where aliasing is acceptable, or when modeling sharp boundaries (e.g., solid objects) [S1, S2, unofficial].
- **When not to use:** High-fidelity rendering requiring smooth gradients, or when the grid resolution is high enough that interpolation artifacts are negligible compared to the cost [S2, unofficial].

## Background
The method originates from basic volume rendering techniques using 3D voxel grids. It is commonly referred to as "probing" in libraries like OpenVDB, which are designed for efficient storage and retrieval of sparse or dense volume data [S1, unofficial]. The technique is foundational in ray marching algorithms where sample points are cast through a volume to accumulate opacity and color [S1, unofficial]. The use of periodic boundary conditions (wrapping) is often employed in procedural noise generation or when simulating infinite domains [S2, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S3] mwrazam/marchingtets-reconstruction, https://github.com/mwrazam/marchingtets-reconstruction, fetched 2026-10-27 (unofficial)
[S4] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
