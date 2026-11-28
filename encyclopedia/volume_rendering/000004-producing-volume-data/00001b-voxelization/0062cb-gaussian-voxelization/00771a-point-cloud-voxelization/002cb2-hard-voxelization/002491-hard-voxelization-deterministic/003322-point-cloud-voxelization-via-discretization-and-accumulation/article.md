# Point Cloud Voxelization via Discretization and Accumulation

Also called: Coverage Estimation and View Frustum Construction, LiDAR Point Cloud Voxelization, MATLAB voxel grid conversion, Sparse SfM Point Voxelization, Voxel-based Urban Vegetation Volume Analysis, binary occupancy grid from point cloud

## Overview
This method converts an irregular 3D point cloud into a structured, binary 3D volume model (occupancy grid) by discretizing point coordinates into a regular grid and marking voxels as occupied if they contain at least one point [S1][S2, unofficial]. It is distinguished from sibling methods by its reliance on direct coordinate binning and accumulation to produce a binary occupancy map, rather than storing feature vectors or using sorting-based parallelization [S2, unofficial]. It serves as a foundational step for spatial analysis and volume calculation, such as urban vegetation modeling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An $N \times 3$ matrix `xyz` containing $(X, Y, Z)$ coordinates for $N$ points [S1][S2, unofficial].
*   **Input:** A scalar `voxelsize` defining the cubic voxel edge length (e.g., 0.5 or 1.0) [S1][S2, unofficial].
*   **Output:** A 3D binary logical array `V` where `V(i,j,k) = 1` if occupied and `0` otherwise [S2, unofficial].

**Data Structures:**
*   `xyz`: $N \times 3$ floating-point matrix [S1][S2, unofficial].
*   `V`: 3D logical array of dimensions $D_x \times D_y \times D_z$ derived from coordinate ranges and `voxelsize` [S2, unofficial].

**Parameters:**
*   `voxelsize`: Physical voxel edge length; typical values are 0.5 [S2, unofficial] or 1.0 [S1].
*   $(X_{min}, Y_{min}, Z_{min})$: Minimum coordinates of the point cloud along each axis, establishing the voxel coordinate system origin [S1].

**Steps:**
1.  **Establish Coordinate System:** Determine the minimum coordinates of the point cloud: $X_{min} = \min(X)$, $Y_{min} = \min(Y)$, $Z_{min} = \min(Z)$ [S1]. The method ensures resulting voxel indices are non-negative [S1].
2.  **Discretize Coordinates:** For each dimension $d \in \{X, Y, Z\}$: extract the coordinate column `tmp` [S2, unofficial]; define bin edges starting at $\min(tmp)$, stepping by `voxelsize`, and ending at $\max(tmp) + \text{voxelsize}$ [S2, unofficial]; convert continuous coordinates to discrete bin indices using $i = \text{Int}((X - X_{min}) / \Delta i)$, $j = \text{Int}((Y - Y_{min}) / \Delta j)$, $k = \text{Int}((Z - Z_{min}) / \Delta k)$ [S1]. Here $\Delta i = \Delta j = \Delta k = \text{voxelsize}$, and $\text{Int}(\cdot)$ performs truncation towards zero (floor for non-negative values) [S1][G].
3.  **Accumulate and Threshold:** Count the number of points assigned to each discrete bin index [S2, unofficial]; threshold the counts to create a binary occupancy grid where `V = 1` if count $> 0$ and `0` otherwise [S2, unofficial].
4.  **Visualization (Optional):** Display the resulting binary volume `V` using a volume viewer [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def point_cloud_voxelization(xyz: np.ndarray, voxelsize: float) -> np.ndarray:
    """
    Convert a 3D point cloud to a binary voxel grid via discretization and accumulation.
    
    Args:
        xyz: (N, 3) array of point coordinates.
        voxelsize: Side length of the cubic voxels.
        
    Returns:
        V: 3D binary numpy array representing the voxel grid.
    """
    if voxelsize <= 0:
        raise ValueError("voxelsize must be positive")
        
    min_coords = np.min(xyz, axis=0)
    indices = np.floor((xyz - min_coords) / voxelsize).astype(int)
    grid_shape = np.max(indices, axis=0) + 1
    V = np.zeros(grid_shape, dtype=bool)
    V[tuple(indices.T)] = True
    return V
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point errors in coordinate calculations can cause points on voxel boundaries to be assigned to incorrect bins [G].
*   **Memory Usage:** The output grid size scales with the bounding box range divided by `voxelsize` [S1].
*   **Resolution Trade-off:** High-resolution models (small `voxelsize`) provide detailed information but are slower for calculation and visual processing, whereas low-resolution models have fast response times but less detail [S1].
*   **Negative Coordinates:** The formula $(X - X_{min})$ ensures non-negative numerators, guaranteeing non-negative voxel indices [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for discretization and accumulation, where $N$ is the number of points [S2, unofficial].
*   **Memory Complexity:** $O(V)$ for the output grid, where $V = D_x \times D_y \times D_z$ [S1].
*   **Strengths:** Deterministic; produces a clean binary occupancy grid suitable for volume calculations [S1].
*   **Limitations:** Does not preserve point attributes (e.g., intensity, color) [S2, unofficial]; memory usage scales with the bounding box size and inverse voxel size [S1].
*   **When to use:** When a binary occupancy representation is needed for volume analysis or as input for further voxel-based processing [S1].
*   **When not to use:** When preserving point features is required or when the scene is extremely large and sparse [S1][G].

## Background
This method originates from applications in urban vegetation analysis using LiDAR point clouds, where converting irregular point data into a structured 3D volume model facilitates spatial analysis and volume calculation [S1]. It is also commonly implemented in MATLAB environments for converting laser scanner point clouds into voxel grids for visualization and further processing [S2, unofficial]. The approach relies on basic discretization and accumulation techniques, which are standard in computational geometry for converting continuous data to discrete grids [G].

## Sources
[S1] Voxel-based Urban Vegetation Volume Analysis with LiDAR Point Cloud, https://openpublishing.library.umass.edu/fabos/article/803/galley/753/download/, fetched 2026-10-27
[S2] 3d point cloud into voxel grid - MATLAB Answers - MathWorks, https://www.mathworks.com/matlabcentral/answers/1621610-3d-point-cloud-into-voxel-grid, fetched 2026-10-28 (unofficial)
