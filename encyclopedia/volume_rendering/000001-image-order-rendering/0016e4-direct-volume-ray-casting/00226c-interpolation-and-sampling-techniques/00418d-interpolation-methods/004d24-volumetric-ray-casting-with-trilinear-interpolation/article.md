# Volumetric Ray Casting with Trilinear Interpolation

Also called: Basic Volumetric Ray Casting, Voxel Region Interpolation

## Overview
Volumetric Ray Casting with Trilinear Interpolation is a direct volume rendering technique that visualizes scalar field data by casting rays through a voxel grid and sampling values at discrete intervals [S1, unofficial]. It approximates the continuous volume function $V(\mathbf{p})$ by linearly interpolating the scalar values of the eight corners of the enclosing voxel cell [S1, unofficial]. It is distinguished from sibling methods by its use of simple distance-based weight normalization rather than standard geometric trilinear blending or higher-order kernels, resulting in a basic, computationally inexpensive implementation suitable for rapid prototyping [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A 3D scalar field stored in a flat array (e.g., RAW format) with dimensions $W \times H \times D$ (width, height, depth). A camera ray defined by origin $\mathbf{o}$, direction $\mathbf{d}$, and interval $[t_{min}, t_{max}]$.
*   **Output**: A scalar value $V(\mathbf{p})$ at a query point $\mathbf{p}$ along the ray.

**Data Structures**
The volume data is stored in a 1D array `voxels` of size $W \times H \times D$. Each element is a struct:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Voxel {
    Vector3 location; // 3D coordinates (x, y, z)
    float scalardata; // Scalar value at this location
};
```
The `location` field is pre-computed during initialization. The index $i$ for coordinates $(x, y, z)$ is calculated as:
$$ i = x + y \cdot W + z \cdot W \cdot H $$

**Numbered Steps**

1.  **Preparation**: Iterate through all voxels in $z, y, x$ order. For each voxel at index `idx` corresponding to coordinates $(x, y, z)$, set `voxels[idx].location` to $(x, y, z)$ and `voxels[idx].scalardata` to the input data value at `idx` [S1, unofficial].

2.  **Ray-Box Intersection**: Determine the entry and exit points of the ray within the volume bounding box defined by $\mathbf{b}_{min} = (0,0,0)$ and $\mathbf{b}_{max} = (W-1, H-1, D-1)$ (or physical bounds if scaled).
    *   Compute inverse direction: $\mathbf{inv\_dir} = 1.0 / \mathbf{d}$ [S1, unofficial].
    *   Compute intersection parameters for each axis:
        $$ \mathbf{minTs} = (\mathbf{b}_{min} - \mathbf{o}) \odot \mathbf{inv\_dir} $$
        $$ \mathbf{maxTs} = (\mathbf{b}_{max} - \mathbf{o}) \odot \mathbf{inv\_dir} $$
        where $\odot$ denotes component-wise multiplication [S1, unofficial].
    *   Calculate near and far $t$ values:
        $$ t_{near} = \max(\min(\mathbf{minTs}_x, \mathbf{maxTs}_x), \min(\mathbf{minTs}_y, \mathbf{maxTs}_y), \min(\mathbf{minTs}_z, \mathbf{maxTs}_z)) $$
        $$ t_{far} = \min(\max(\mathbf{minTs}_x, \mathbf{maxTs}_x), \max(\mathbf{minTs}_y, \mathbf{maxTs}_y), \max(\mathbf{minTs}_z, \mathbf{maxTs}_z)) $$
    *   Intersection exists if $t_{near} \le t_{far}$ and $t_{far} > 0$ [S1, unofficial]. Default $t_{min} = 0.0$, $t_{max} = \infty$ [S1, unofficial].

3.  **Sampling Loop**: For each sample point $\mathbf{p}$ along the ray within $[t_{near}, t_{far}]$:
    *   Identify the enclosing voxel cell by taking the floor of the coordinates:
        $$ x = \lfloor p_x \rfloor, \quad y = \lfloor p_y \rfloor, \quad z = \lfloor p_z \rfloor $$
        [S1, unofficial].
    *   Retrieve the 8 corner voxel indices using a boundary-clamping function `getCell(x, y, z)` [S1, unofficial]. The corners are:
        *   $v_{00} = \text{getCell}(x, y, z)$
        *   $v_{01} = \text{getCell}(x+1, y, z)$
        *   $v_{02} = \text{getCell}(x+1, y+1, z)$
        *   $v_{03} = \text{getCell}(x, y+1, z)$
        *   $v_{04} = \text{getCell}(x, y, z+1)$
        *   $v_{05} = \text{getCell}(x+1, y, z+1)$
        *   $v_{06} = \text{getCell}(x+1, y+1, z+1)$
        *   $v_{07} = \text{getCell}(x, y+1, z+1)$
        [S1, unofficial].

4.  **Weight Calculation**: Compute the Euclidean distance from the query point $\mathbf{p}$ to each corner location $\mathbf{l}_i$ of the retrieved voxels:
    $$ w_i = \| \mathbf{p} - \mathbf{l}_i \| $$
    for $i \in \{00, \dots, 07\}$ [S1, unofficial].
    Sum the weights:
    $$ W_{total} = \sum_{i=00}^{07} w_i $$
    Normalize the weights:
    $$ \hat{w}_i = \frac{w_i}{W_{total}} $$
    [S1, unofficial].

5.  **Interpolation**: Compute the final scalar value:
    $$ V(\mathbf{p}) = \sum_{i=00}^{07} \hat{w}_i \cdot \text{voxels}[v_i].\text{scalardata} $$
    [S1, unofficial].

**Boundary Handling (`getCell`)**
The `getCell` function clamps coordinates to the valid range to prevent out-of-bounds access. Note that the source implementation clamps $x$ to $[0, W]$, $y$ to $[0, H]$, and $z$ to $[0, D]$ [S1, unofficial].
*   If $x < 0$, set $x = 0$.
*   If $x > W$, set $x = W$.
*   (Same for $y$ and $H$, $z$ and $D$).
*   Compute index: $idx = x + y \cdot W + z \cdot W \cdot H$ [S1, unofficial].

**Pitfalls and Edge Cases**
*   **Out-of-Bounds Access**: The source code clamps coordinates up to $W, H, D$ inclusive. Since valid indices are $0 \dots W-1$, clamping to $W$ results in an index out of the array bounds $[0, W \cdot H \cdot D - 1]$ [S1, unofficial]. An implementer must clamp to $W-1, H-1, D-1$ instead [G].
*   **Zero Weight Sum**: If the query point $\mathbf{p}$ coincides exactly with a voxel corner, some weights become zero. If $\mathbf{p}$ is equidistant to all corners (e.g., center of the voxel), weights are equal. The sum $W_{total}$ is non-zero unless $\mathbf{p}$ is undefined. Division by zero is not expected in standard usage but should be guarded against [G].
*   **Precision**: Using `float` for coordinates and distances may introduce precision errors near boundaries.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def get_cell(x, y, z, width, height, depth):
    # Clamp coordinates to valid index range [0, dim-1]
    # Note: Source [S1, unofficial] clamps to dim, which is out-of-bounds. 
    # Corrected to dim-1 for valid implementation [G].
    x = max(0, min(x, width - 1))
    y = max(0, min(y, height - 1))
    z = max(0, min(z, depth - 1))
    return x + y * width + z * width * height

def trilinear_interpolation(ray, voxels, width, height, depth):
    # 1. Ray-Box Intersection
    # Bbox: bmin=(0,0,0), bmax=(width-1, height-1, depth-1)
    bmin = (0.0, 0.0, 0.0)
    bmax = (float(width - 1), float(height - 1), float(depth - 1))
    
    inv_dir = (1.0/ray.d.x, 1.0/ray.d.y, 1.0/ray.d.z)
    
    min_ts = ((bmin[0] - ray.o.x) * inv_dir[0],
              (bmin[1] - ray.o.y) * inv_dir[1],
              (bmin[2] - ray.o.z) * inv_dir[2])
    max_ts = ((bmax[0] - ray.o.x) * inv_dir[0],
              (bmax[1] - ray.o.y) * inv_dir[1],
              (bmax[2] - ray.o.z) * inv_dir[2])
              
    t_near = max(min(min_ts[0], max_ts[0]),
                 min(min_ts[1], max_ts[1]),
                 min(min_ts[2], max_ts[2]))
    t_far = min(max(min_ts[0], max_ts[0]),
                max(min_ts[1], max_ts[1]),
                max(min_ts[2], max_ts[2]))
                
    if t_near > t_far or t_far < 0:
        return None # No intersection
        
    t_start = max(t_near, ray.t_min)
    t_end = min(t_far, ray.t_max)
    
    # 2. Sampling Loop
    # Step size depends on desired resolution; not specified in source [G]
    step = 1.0 # Example step size in voxel units
    
    t = t_start
    while t <= t_end:
        p = ray.origin + ray.direction * t
        
        # 3. Identify enclosing voxel
        x = math.floor(p.x)
        y = math.floor(p.y)
        z = math.floor(p.z)
        
        # 4. Retrieve 8 corners
        v00 = get_cell(x, y, z, width, height, depth)
        v01 = get_cell(x+1, y, z, width, height, depth)
        v02 = get_cell(x+1, y+1, z, width, height, depth)
        v03 = get_cell(x, y+1, z, width, height, depth)
        v04 = get_cell(x, y, z+1, width, height, depth)
        v05 = get_cell(x+1, y, z+1, width, height, depth)
        v06 = get_cell(x+1, y+1, z+1, width, height, depth)
        v07 = get_cell(x, y+1, z+1, width, height, depth)
        
        corners = [v00, v01, v02, v03, v04, v05, v06, v07]
        
        # 5. Calculate weights (Euclidean distance)
        weights = []
        for v_idx in corners:
            loc = voxels[v_idx].location
            dist = math.sqrt((p.x - loc.x)**2 + (p.y - loc.y)**2 + (p.z - loc.z)**2)
            weights.append(dist)
            
        total_weight = sum(weights)
        
        if total_weight == 0:
            # Fallback if point is exactly at a corner and all dists are 0?
            # In practice, this only happens if p == loc for all 8, which is impossible.
            # Guard against division by zero [G]
            interpolated_value = 0.0
        else:
            # 6. Compute interpolated value
            interpolated_value = 0.0
            for i, v_idx in enumerate(corners):
                normalized_w = weights[i] / total_weight
                interpolated_value += normalized_w * voxels[v_idx].scalardata
                
        t += step
        
    return interpolated_value
```

## Subtopics

### Interpolation Schemes for Volume Rendering
A broader category comparing various kernels (linear, cubic, etc.). Trilinear is the simplest linear scheme. [Link](817-interpolation-schemes-for-volume-rendering.md)

### Gradient-Based Interpolation by Sample Splatting
Uses gradient information for higher fidelity. More complex than basic trilinear. [Link](9072-gradient-based-interpolation-by-sample-splatting.md)

### Shape-Based Interpolation
Preserves sharp features better than linear interpolation. [Link](10499-shape-based-interpolation.md)

### Interblock Interpolation
Optimizes for tiled rendering. [Link](11155-interblock-interpolation.md)

### Lanczos Resampling
Uses sinc-based kernels for high-quality reconstruction. [Link](11246-lanczos-resampling.md)

### Nearest Neighbor Sampling
Uses the value of the closest voxel. Faster but aliasing-prone. [Link](11694-nearest-neighbor-sampling.md)

## Operating characteristics
*   **Time Complexity**: $O(N)$ per ray, where $N$ is the number of samples along the ray. Each sample requires 8 memory lookups and basic arithmetic.
*   **Memory Complexity**: $O(W \cdot H \cdot D)$ to store the voxel grid with pre-computed locations.
*   **Performance**: Very fast due to simple arithmetic. Suitable for real-time visualization if resolution is low or hardware acceleration is used [S1, unofficial].
*   **Strengths**: Simple to implement; low computational overhead; good for quick prototyping [S1, unofficial].
*   **Limitations**: Low accuracy; blurs sharp features; the specific weight calculation (distance-based normalization) is non-standard for trilinear interpolation and may produce artifacts compared to standard geometric trilinear blending [S1, unofficial].
*   **When to use**: Basic visualization, educational purposes, or when performance is critical and quality is secondary [S1, unofficial].
*   **When not to use**: High-fidelity medical imaging or scientific visualization requiring sharp feature preservation.

## Background
The method originates from a basic volumetric ray casting application intended as starter code for visualization purposes [S1, unofficial]. It was developed by Srinath Ravichandran in 2014 [S1, unofficial]. The implementation supports GPU rendering and multithreading via OpenMP [S1, unofficial]. It is cited as a "quick and dirty" solution for setting up volume rendering [S1, unofficial].

## Sources
[S1] sriravic/volumeraycast, https://github.com/sriravic/volumeraycast, fetched 2026-10-26 (unofficial)
