# Scanline Projection (Nearest Neighbor)

## Overview
Scanline Projection (Nearest Neighbor) renders a 2D projection slice of a 3D voxel grid by sampling voxel values along scanlines defined by a 3D affine transform, using nearest-neighbor interpolation [S1, unofficial]. It differs from [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md) and [Parametric Surface Voxelization](6032-parametric-surface-voxelization.md) by operating directly on discrete voxel indices rather than continuous surface representations, and from [Multi-Plane Projection Voxelization and Aggregation](12100-multi-plane-projection-voxelization-and-aggregation.md) by processing one scanline at a time without intermediate plane aggregation [S1, unofficial].

## How it works
**Inputs:**
- `voxel_data`: A 3D array of voxel values (e.g., IDs or densities) with dimensions $X \times Y \times Z$.
- `transform`: A $3 \times 4$ affine transformation matrix $T$ mapping screen coordinates to voxel space. The matrix is stored in row-major order as `t.m[row][col]`.
- `lut`: A lookup table mapping voxel values to output colors/intensities.
- `dest`: A 2D output buffer for the projection slice.
- `nx`, `ny`: The width and height of the output projection slice.

**Outputs:**
- `dest`: The rendered 2D projection slice.

**Data Structures:**
- The voxel grid is accessed using flat array indexing. For a voxel at $(x, y, z)$, the index is `int(x) + int(y)*ystride + int(z)*zstride`, where `ystride = X` and `zstride = X*Y` [S1, unofficial].
- The transform matrix $T$ provides the step sizes for the scanline. The step vector $(dx, dy, dz)$ is extracted from the first column of the matrix: $dx = T_{0,0}$, $dy = T_{1,0}$, $dz = T_{2,0}$ [S1, unofficial].

**Algorithm Steps:**
1. **Initialize Scanline:** For each row $y$ in the output image (from $0$ to $ny-1$):
   a. Calculate the start point $p_0$ in voxel space corresponding to screen coordinate $(0, y)$. The source specifies calculating $p_0$ at $(0, y, z)$ and $p_1$ at $(nx-1, y, z)$, but the core logic relies on the transform to map screen $(0, y)$ to voxel space. The start coordinates are initialized as $x_0 = p_0.x + 0.5$, $y_0 = p_0.y + 0.5$, $z_0 = p_0.z + 0.5$ to center the sample within the voxel [S1, unofficial]. Note: The specific $z$ component of the start point depends on the projection geometry (e.g., fixed $z$ for parallel projection or derived from transform).
   b. Extract step sizes: $dx = T_{0,0}$, $dy = T_{1,0}$, $dz = T_{2,0}$ [S1, unofficial].

2. **Iterate Columns:** For each column $ix$ from $0$ to $nx-1$:
   a. **Boundary Check:** Determine if the current voxel coordinates $(x_0, y_0, z_0)$ are within the valid bounds $[0, X)$, $[0, Y)$, $[0, Z)$ [S1, unofficial].
   b. **Sample Voxel:**
      - If within bounds: Compute the flat index `idx = int(x0) + int(y0)*ystride + int(z0)*zstride`. Retrieve `val = voxel_data[idx]` [S1, unofficial].
      - If out of bounds: Call a boundary handling function `pixelAt(x0, y0, z0)` which returns a default value (e.g., 0 or background) [S1, unofficial].
   c. **Apply Lookup:** Map the voxel value to the output color/intensity: `color = lut[val]` [S1, unofficial].
   d. **Write Output:** Store `color` in `dest` at the current pixel position.
   e. **Increment Coordinates:** Update $x_0 \leftarrow x_0 + dx$, $y_0 \leftarrow y_0 + dy$, $z_0 \leftarrow z_0 + dz$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scanline_projection_nearest_neighbor(voxel_data, transform, lut, dest, nx, ny, X, Y, Z):
    """
    Renders a 2D projection slice using nearest-neighbor interpolation.
    
    Args:
        voxel_data: 3D array of shape (X, Y, Z)
        transform: 3x4 matrix (list of lists or flat array)
        lut: Lookup table mapping voxel values to output colors
        dest: 2D output buffer of shape (ny, nx)
        nx, ny: Output image dimensions
        X, Y, Z: Voxel grid dimensions
    """
    ystride = X
    zstride = X * Y
    
    # Extract step sizes from the first column of the transform matrix
    dx = transform[0][0]
    dy = transform[1][0]
    dz = transform[2][0]
    
    for y in range(ny):
        # Calculate start point in voxel space for screen coordinate (0, y)
        # The transform maps screen (0, y) to voxel space.
        # Assuming the transform includes translation, we compute the initial voxel coordinates.
        # The source specifies adding 0.5 for centering.
        # Note: The exact calculation of p0 depends on the full transform application.
        # For a standard affine transform T, voxel_coords = T * [0, y, 0, 1]^T
        # However, the source simplifies this to extracting steps and initializing from p0.
        # We assume p0 is pre-calculated or derived from the transform's translation and rotation components.
        # For implementation clarity, we assume a helper or direct matrix multiplication.
        
        # Screen coordinate (0, y) mapped to voxel space
        # Using the first column for steps, we need the initial position.
        # The source implies p0 is at (0, y, z) in screen space, but the transform maps it.
        # Let's assume the transform is applied to the screen origin and y-axis vector.
        # p0_voxel = transform * [0, y, 0, 1] (assuming homogeneous coordinates)
        # This is a simplification; the actual p0 depends on the specific projection setup.
        # For the purpose of this algorithm, we assume p0_voxel is available.
        
        # Placeholder for p0 calculation based on transform and screen y
        # In a real implementation, this would be:
        # p0_voxel_x = transform[0][3] + y * transform[0][1]
        # p0_voxel_y = transform[1][3] + y * transform[1][1]
        # p0_voxel_z = transform[2][3] + y * transform[2][1]
        # But the source says "Calculate start point p0 at (0, y, z) ... using transform t"
        # and then "x0 = p0.x + .5". This implies p0 is the result of the transform.
        
        # Let's assume the transform maps screen (0, y) to voxel space directly.
        # We need the initial x0, y0, z0.
        # The source does not give the full matrix multiplication, only the steps.
        # We will assume the initial coordinates are computed as:
        x0 = transform[0][3] + y * transform[0][1] + 0.5
        y0 = transform[1][3] + y * transform[1][1] + 0.5
        z0 = transform[2][3] + y * transform[2][1] + 0.5
        
        for x in range(nx):
            # Check bounds
            ix = int(x0)
            iy = int(y0)
            iz = int(z0)
            
            if 0 <= ix < X and 0 <= iy < Y and 0 <= iz < Z:
                idx = ix + iy * ystride + iz * zstride
                val = voxel_data[idx]
            else:
                # Boundary handling: return default value (e.g., 0)
                val = 0
            
            # Apply LUT
            color = lut[val]
            
            # Write to destination
            dest[y][x] = color
            
            # Increment coordinates
            x0 += dx
            y0 += dy
            z0 += dz
```

**Parameters:**
- `dx, dy, dz`: Step sizes derived from the transform matrix. Typical values depend on the resolution and field of view.
- `lut`: Lookup table size depends on the number of unique voxel values.

**Edge Cases and Pitfalls:**
- **Out-of-Bounds Access:** If the scanline extends outside the voxel grid, direct indexing will cause segfaults. A boundary check or clamping function (`pixelAt`) is required [S1, unofficial].
- **Precision:** Floating-point accumulation of $x_0, y_0, z_0$ can lead to drift. Re-computing coordinates from integer indices or using Kahan summation may be necessary for high precision [G].
- **Degenerate Input:** If the transform results in $dx=dy=dz=0$, the scanline will not progress, causing an infinite loop or repeated sampling [G].

## Subtopics

### Scanline Trilinear Interpolation
Renders a 2D projection slice using trilinear interpolation of voxel values along scanlines, providing smoother results than nearest-neighbor [S1, unofficial]. It differs by calling `trilinear(x0, y0, z0)` instead of direct indexing, which samples the 8 surrounding voxels and weights them based on the fractional part of the coordinates [S1, unofficial]. Choose this when visual quality is prioritized over speed. The algorithm steps are identical to nearest-neighbor, except for the sampling step: `val = trilinear(x0, y0, z0)` [S1, unofficial].

### Transformed Line Sampling (scanLine)
Samples voxel IDs along a line segment defined by a 3x4 affine transformation matrix, mapping screen coordinates to voxel space with center sampling and boundary handling [S1, unofficial]. It differs by supporting thick scanlines (depth accumulation) and using a buffer to accumulate values from multiple depth offsets [S1, unofficial]. It is chosen when depth information needs to be aggregated into a single 2D projection. The algorithm involves calling `scanLine` for $dz=0$ and then iterating $dz$ from $-thick$ to $thick$ (excluding 0), accumulating results into a buffer `buf` and then into `dest` if `buf[x]>0 && dest[x]==0` [S1, unofficial]. Parameters include `dz1` (depth offset) and `thick` (half-thickness) [S1, unofficial]. A pitfall is that direct indexing assumes integer truncation, and out-of-bounds endpoints trigger a fallback to `get()` to prevent segfaults [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels in the output slice and $M$ is the number of voxels sampled per pixel (1 for nearest-neighbor, more for thick scanlines) [S1, unofficial].
- **Memory Complexity:** $O(X \cdot Y \cdot Z)$ for the voxel grid, plus $O(N \cdot M)$ for temporary buffers if thick scanlines are used [S1, unofficial].
- **Performance:** Fast for nearest-neighbor due to simple integer indexing. Slower for trilinear interpolation due to multiple lookups and floating-point operations [S1, unofficial].
- **Strengths:** Simple implementation, low memory overhead for the projection process, suitable for real-time applications [S1, unofficial].
- **Limitations:** Aliasing artifacts with nearest-neighbor, boundary handling complexity [S1, unofficial].
- **When to Use:** When speed is critical and visual quality is acceptable with nearest-neighbor, or when smooth interpolation is needed with trilinear [S1, unofficial].
- **When Not to Use:** When high-quality anti-aliasing is required, or when the voxel grid is very large and memory is constrained [G].

## Background
Originated in early volume rendering techniques for visualizing 3D scalar fields, scanline projection is a fundamental rasterization-based approach [S1, unofficial]. It is used in medical imaging and scientific visualization for fast preview and interactive exploration [S1, unofficial].

## Sources
[S1] wagenadl/gvox, https://github.com/wagenadl/gvox, fetched 2026-10-28 (unofficial)
