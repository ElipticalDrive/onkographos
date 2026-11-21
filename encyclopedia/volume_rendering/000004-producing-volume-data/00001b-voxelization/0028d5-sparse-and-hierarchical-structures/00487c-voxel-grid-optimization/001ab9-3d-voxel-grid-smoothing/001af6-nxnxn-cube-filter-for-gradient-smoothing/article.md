# NxNxN Cube Filter for Gradient Smoothing

Also called: 3D Box Filter for Gradients, Cubic Gradient Smoothing, N x N x N Gradient Filtering

## Overview
This method smooths gradient vectors in a 3D voxel grid by averaging the gradients of neighboring voxels within a cubic neighborhood [S2, unofficial]. It solves the problem of high-frequency irregularities and noise in gradient fields, which can cause artifacts during isosurface extraction or volume rendering [S2, unofficial]. It differs from its sibling [3D Box Filter for Voxel Averaging](1513-3d-box-filter-for-voxel-averaging.md) by operating on vector-valued gradient data rather than scalar voxel intensities, and from [Iterative Convolution and Thresholding for Noise Removal](1516-iterative-convolution-and-thresholding-for-noise-removal.md) by using a single-pass box filter without iterative refinement or thresholding [G].

## How it works
**Inputs:**
- A 3D grid of gradient vectors with dimensions $W \times H \times D$ (width, height, depth) [S2, unofficial].
- An integer filter size $n$, which must be an odd number [S2, unofficial].

**Outputs:**
- A smoothed 3D grid of gradient vectors with the same dimensions as the input [S2, unofficial].

**Data Structures:**
- `gradients`: A 3D array of size $W \times H \times D$, where each element is a 3-component floating-point vector $(g_x, g_y, g_z)$ [G].
- `temp_average`: A temporary 3-component floating-point vector used for accumulation [G].
- `count`: An integer accumulator tracking the number of valid neighbors within bounds [G].

**Parameters:**
- $n$: The dimension of the cubic filter. Must be an odd number. Default value is 3 [S2, unofficial].

**Steps:**
1. For each voxel at coordinates $(x, y, z)$ where $0 \le x < W$, $0 \le y < H$, $0 \le z < D$:
   a. Calculate the radius $r = (n - 1) / 2$ [S2, unofficial].
   b. Initialize `temp_average` to $(0, 0, 0)$ and `count` to 0 [G].
   c. Iterate over $k$ from $z - r$ to $z + r$ (inclusive).
   d. Iterate over $j$ from $y - r$ to $y + r$ (inclusive).
   e. Iterate over $i$ from $x - r$ to $x + r$ (inclusive).
   f. If the coordinates $(i, j, k)$ are within the grid bounds ($0 \le i < W$, $0 \le j < H$, $0 \le k < D$):
      i. Add `gradients[i][j][k]` to `temp_average`.
      ii. Increment `count`.
   g. If `count` > 0:
      i. Divide each component of `temp_average` by `count` to get the averaged vector [S2, unofficial].
      ii. If the averaged vector is non-zero, normalize it to unit length; otherwise, set it to $(0, 0, 0)$ [S2, unofficial].
   h. Store the resulting vector in the output grid at $(x, y, z)$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def filter_nxnxn_gradients(gradients, width, height, depth, n):
    """
    Applies an NxNxN cube filter to gradient vectors.
    
    Args:
        gradients: 3D list of tuples (gx, gy, gz) of size width x height x depth.
        width, height, depth: Dimensions of the grid.
        n: Filter size (must be odd).
        
    Returns:
        Smoothed gradient grid as a 3D list of tuples.
    """
    if n % 2 == 0:
        raise ValueError("Filter size n must be an odd number.")
        
    radius = (n - 1) // 2
    smoothed = [[[None for _ in range(depth)] for _ in range(height)] for _ in range(width)]

    for z in range(depth):
        for y in range(height):
            for x in range(width):
                avg_x, avg_y, avg_z = 0.0, 0.0, 0.0
                count = 0
                
                for k in range(z - radius, z + radius + 1):
                    for j in range(y - radius, y + radius + 1):
                        for i in range(x - radius, x + radius + 1):
                            if 0 <= i < width and 0 <= j < height and 0 <= k < depth:
                                gx, gy, gz = gradients[i][j][k]
                                avg_x += gx
                                avg_y += gy
                                avg_z += gz
                                count += 1
                
                if count > 0:
                    avg_x /= count
                    avg_y /= count
                    avg_z /= count
                    
                    length = (avg_x**2 + avg_y**2 + avg_z**2)**0.5
                    if length > 0:
                        avg_x /= length
                        avg_y /= length
                        avg_z /= length
                    else:
                        avg_x, avg_y, avg_z = 0.0, 0.0, 0.0
                        
                smoothed[x][y][z] = (avg_x, avg_y, avg_z)
                
    return smoothed
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Voxels near the grid boundaries have fewer neighbors. The method handles this by only summing valid neighbors and dividing by the actual count, effectively using a smaller averaging kernel at boundaries [S2, unofficial].
- **Zero Vector:** If the averaged gradient vector is zero (e.g., all neighbors are zero or cancel out), normalization is undefined. The pseudocode handles this by checking the vector length and setting the result to a zero vector when the length is zero [S2, unofficial].
- **Return Value Bug:** In the original source code, the `sampleNxNxN` function returns a reference to a local variable `average`, which is undefined behavior and likely causes garbage values. The pseudocode avoids this by storing the computed result directly into the output array [G].
- **Filter Size:** The filter size $n$ must be odd to ensure a well-defined center voxel [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(W \cdot H \cdot D \cdot n^3)$ [G]. Each voxel requires $n^3$ vector additions and one normalization [G].
- **Memory Complexity:** $O(W \cdot H \cdot D)$ for the input and output gradient grids. Additional $O(1)$ space per voxel for temporary accumulation [G].
- **Strengths:** Simple to implement; effectively reduces high-frequency noise in gradients; preserves the general direction of gradients [G].
- **Limitations:** Blurs gradient details; computational cost increases cubically with $n$; boundary effects may cause slight inaccuracies near edges [G].
- **When to use:** When gradient noise causes artifacts in isosurface extraction or volume rendering, and computational resources allow for $O(n^3)$ per voxel [G].
- **When not to use:** When preserving fine gradient details is critical, or when performance is a major constraint with large $n$ [G].

## Background
The NxNxN cube filter for gradient smoothing was described in the context of volume rendering transfer functions and gradient calculation [S2, unofficial]. It is part of a pipeline that calculates gradients using central differences and then smooths them to prevent high irregularities [S2, unofficial].

## Sources
[S1] jose-villegas/IllustrativeVolumeRendering, https://github.com/jose-villegas/IllustrativeVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering 102: Transfer Functions, http://graphicsrunner.blogspot.com/2009/01/volume-rendering-102-transfer-functions.html, fetched 2026-10-28 (unofficial)
