# Dynamic Voxelization (Coordinate Only)
## Overview
Dynamic Voxelization (Coordinate Only) computes the discrete voxel coordinates for each point in a point cloud without assigning them to a fixed-size voxel grid structure or handling duplicate merging [S1, unofficial]. It is used when `max_points` is set to -1, effectively bypassing the aggregation and duplicate-handling stages of standard hard voxelization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Point cloud coordinates tensor `points` of shape $(N, D)$, where $N$ is the number of points and $D$ is the dimensionality (typically 3) [G]. Bounding box range $R = [x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$ and voxel sizes $v = [v_x, v_y, v_z]$ [G].
*   **Output:** Tensor `coors` of shape $(N, D)$ containing the integer voxel indices for each point [S1, unofficial].

**Data Structures:**
*   `points`: Float tensor, shape $(N, D)$ [G].
*   `coors`: Integer tensor, shape $(N, D)$ [S1, unofficial].
*   `grid_dims`: Integer tuple $(G_x, G_y, G_z)$ representing the dimensions of the bounding box in voxels [S1, unofficial].

**Parameters:**
*   `NDim`: Number of coordinate dimensions. Default is 3 [S1, unofficial].
*   `max_points`: Set to -1 to indicate coordinate-only mode [S1, unofficial].

**Steps:**
1.  **Calculate Grid Dimensions:** Compute the number of voxels along each axis using rounding:
    $$
    G_x = \text{round}\left(\frac{x_{max} - x_{min}}{v_x}\right), \quad
    G_y = \text{round}\left(\frac{y_{max} - y_{min}}{v_y}\right), \quad
    G_z = \text{round}\left(\frac{z_{max} - z_{min}}{v_z}\right)
    $$
    [S1, unofficial]
2.  **Compute Voxel Coordinates:** For each point $p_n$ with coordinates $(x_n, y_n, z_n)$, compute the voxel index $(i_n, j_n, k_n)$:
    $$
    i_n = \left\lfloor \frac{x_n - x_{min}}{v_x} \right\rfloor, \quad
    j_n = \left\lfloor \frac{y_n - y_{min}}{v_y} \right\rfloor, \quad
    k_n = \left\lfloor \frac{z_n - z_{min}}{v_z} \right\rfloor
    $$
    [S1, unofficial]
3.  **Boundary Handling:** Points falling outside the bounding box $R$ result in indices outside $[0, G_x-1] \times [0, G_y-1] \times [0, G_z-1]$ [G]. In this specific coordinate-only implementation, these points are stored as-is, as no duplicate merging or max-point limiting occurs [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dynamic_voxelize_coordinate_only(points, coors_range, voxel_size, ndim=3):
    """
    points: list of lists or tuples of shape (N, ndim)
    coors_range: [x_min, y_min, z_min, x_max, y_max, z_max]
    voxel_size: [vx, vy, vz]
    returns: list of lists of shape (N, ndim) of integer voxel coordinates
    """
    x_min, y_min, z_min, x_max, y_max, z_max = coors_range
    vx, vy, vz = voxel_size
    
    # Calculate grid dimensions (computed per source, though not strictly used for coordinate mapping)
    grid_x = round((x_max - x_min) / vx)
    grid_y = round((y_max - y_min) / vy)
    grid_z = round((z_max - z_min) / vz)
    
    # Initialize output list
    coors = []
    
    # Compute voxel coordinates for each point
    for point in points:
        x, y = point[0], point[1]
        z = point[2] if ndim == 3 else 0.0
        
        # Compute voxel indices using floor division
        i = math.floor((x - x_min) / vx)
        j = math.floor((y - y_min) / vy)
        k = math.floor((z - z_min) / vz) if ndim == 3 else 0
        
        coors.append([i, j, k])
        
    return coors
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point division and floor operations may cause points exactly on boundaries to be assigned to adjacent voxels depending on implementation details [G].
*   **Out-of-Bounds:** Points outside the range $R$ produce negative or excessive indices [G].

## Subtopics

### [Dynamic Voxelization (Coordinate and Feature)](18416-dynamic-voxelization-coordinate-and-feature.md)
Extends this method by adding feature aggregation and duplicate handling after coordinate computation. Choose when features need to be stored per voxel.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of points, as each point is processed independently [G].
*   **Memory Complexity:** $O(N)$ for the output coordinate tensor [G].
*   **Strengths:** Minimal memory overhead and fast execution, as it skips complex atomic operations and sorting required for duplicate handling [G].
*   **Limitations:** Does not reduce point count; output size equals input size [G].
*   **When to use:** When only spatial indexing is needed, or as a pre-processing step for more complex voxelization variants [G].

## Background
This method is part of the dynamic voxelization pipeline used in 3D object detection frameworks, specifically implemented in CUDA for GPU acceleration [S1, unofficial].

## Sources
[S1] Susie0909/Multiscale_Voxelization_CUDA, https://github.com/Susie0909/Multiscale_Voxelization_CUDA, fetched 2026-10-27 (unofficial)
