# Grid Density Data Structure

Also called: 1D Array Density Layout, 3D Grid, 3D Voxel Grid Density Field, Dense Voxel Grid Storage, Dense structure, Linearized 3D Grid, Volume Grid Data Structure

## Overview
The Grid Density Data Structure is a dense, axis-aligned 3D grid used to store scalar density values for volume rendering [S1, S4, unofficial]. It solves the problem of representing heterogeneous media (such as smoke, clouds, or fluids) by quantizing the continuous density field into discrete voxels within a fixed bounding box [S1, unofficial]. It sets itself apart from sparse representations like [Octree](../octree.md) or [Sparse Voxel Octrees](../sparse-voxel-octree.md) by guaranteeing $O(1)$ access time at the cost of memory proportional to the volume of the bounding box, making it faster to sample but limited in resolution for large scenes [S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Grid resolution $N$ (assumed equal in all dimensions, typically a power of 2), world-space bounds (min/max extents), and density data (scalar floats) [S1, S3, unofficial].
*   **Output:** A contiguous 1D array of floats representing the 3D density field, along with metadata for coordinate transformation [S1, S3, unofficial].

**Data Structures:**
The structure consists of a header/metadata block and the data payload:
1.  **Metadata:**
    *   `dimension` ($size\_t$): The resolution $N$ in one dimension (e.g., 128). Total voxels = $N^3$ [S1, unofficial].
    *   `bounds` ($Point[2]$): Two 3D vectors defining the Axis-Aligned Bounding Box (AABB) in world space, `min` and `max` [S1, unofficial]. The physical size must be equal in all three dimensions (e.g., a cube) [S1, unofficial].
    *   `transform` ($Matrix$): A transformation matrix $D$ mapping world coordinates to local grid coordinates [S3, unofficial].
2.  **Data Payload:**
    *   `density` ($float*$): A contiguous 1D array of size $N^3$ storing the density value for each voxel [S1, S3, unofficial].

**Coordinate Transformation:**
To sample the grid, world-space coordinates must be converted to local grid indices. This is achieved using a change-of-coordinate transformation [S3, unofficial]:
1.  Define local points $p_0$ (min bound) and $p_1$ (max bound).
2.  Compute the scaling matrix $S(p_1 - p_0)$ and translation matrix $T(p_0)$.
3.  Compute the world-to-local transformation matrix $D$:
    $$ M = T(p_0) \times S(p_1 - p_0) $$
    $$ D = (W \times M)^{-1} $$
    where $W$ is the world transformation matrix of the grid [S3, unofficial].

**Linearization:**
Voxels are stored in row-major order in the 1D array. The index $I$ for a voxel at local integer coordinates $(x, y, z)$ is calculated as [S3, unofficial]:
$$ I = (z \cdot n_y + y) \cdot n_x + x $$
where $n_x, n_y, n_z$ are the grid dimensions. For a cubic grid of resolution $N$, this simplifies to $I = z \cdot N^2 + y \cdot N + x$ [S1, S3, unofficial].

**Initialization and Population:**
The density array is typically initialized to 0. Values can be populated via simulation, file loading, or procedural generation [S2, S3, unofficial]. For example, a procedural smoke sphere can be generated using a central distance function [S3, unofficial]:
1.  Initialize `density` array of size $n_x \cdot n_y \cdot n_z$ to 0.
2.  Iterate $z$ from 1 to $n_z-2$, $y$ from 1 to $n_y-2$, $x$ from 1 to $n_x-2$ [S3, unofficial].
3.  Compute distance from center:
    $$ dist = \sqrt{(x - \frac{n_x-1}{2})^2 + (y - \frac{n_y-1}{2})^2 + (z - \frac{n_z-1}{2})^2} $$
4.  Assign density:
    $$ density[index] = \frac{1}{2^{dist+2}} $$
    [S3, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class GridDensityDataStructure:
    def __init__(self, resolution: int, bounds_min: tuple, bounds_max: tuple):
        """
        Initialize the grid density data structure.
        
        Args:
            resolution (int): Number of voxels per dimension (N).
            bounds_min (tuple): World-space minimum extent (x, y, z).
            bounds_max (tuple): World-space maximum extent (x, y, z).
        """
        self.resolution = resolution
        self.bounds_min = np.array(bounds_min)
        self.bounds_max = np.array(bounds_max)
        
        # Calculate physical size to ensure cubic grid if required by specific implementations
        # Note: General implementation allows non-cubic, but S1 assumes equal size in all dims
        self.size = self.bounds_max - self.bounds_min
        
        # Compute transformation matrix D (World -> Local)
        # M = T(p0) * S(p1 - p0)
        # D = (W * M)^-1. Assuming W is identity for local grid definition, D = M^-1
        # S is scaling by 1/size, T is translation by -p0
        # Local coord = (World - p0) / size * (N-1) ? 
        # S3 uses homogeneous coordinates. Simplified linear mapping for sampling:
        # local_x = (world_x - bounds_min.x) / (bounds_max.x - bounds_min.x) * (N - 1)
        
        self.density = np.zeros(resolution ** 3, dtype=np.float32)

    def set_density_procedural_sphere(self, center_local: tuple = None):
        """
        Populate the grid with a procedural smoke sphere density field.
        Based on S3.
        """
        n = self.resolution
        if center_local is None:
            center_local = (n - 1) / 2.0
            
        # Precompute constants
        half_n = (n - 1) / 2.0
        inv_2 = 0.5
        
        # Iterate over valid interior voxels to avoid boundary artifacts
        # S3 iterates 1 to n-2
        for z in range(1, n - 1):
            for y in range(1, n - 1):
                for x in range(1, n - 1):
                    # Compute distance from center in grid units
                    dx = x - half_n
                    dy = y - half_n
                    dz = z - half_n
                    dist = np.sqrt(dx*dx + dy*dy + dz*dz)
                    
                    # Calculate index
                    idx = (z * n + y) * n + x
                    
                    # Assign density
                    self.density[idx] = inv_2 ** (dist + 2)

    def get_index(self, x: int, y: int, z: int) -> int:
        """
        Compute 1D linear index from 3D coordinates.
        Row-major layout.
        """
        return (z * self.resolution + y) * self.resolution + x

    def sample_density(self, world_pos: np.ndarray) -> float:
        """
        Sample density at a world-space position.
        Requires interpolation (see Interpolated Discrete Values & Gradients).
        This function returns the raw nearest-neighbor value for demonstration.
        """
        # 1. Transform world to local grid coordinates
        # local = (world - bounds_min) / (bounds_max - bounds_min) * (resolution - 1)
        local_pos = (world_pos - self.bounds_min) / (self.bounds_max - self.bounds_min) * (self.resolution - 1)
        
        # 2. Clamp to grid bounds
        local_pos = np.clip(local_pos, 0, self.resolution - 1)
        
        # 3. Convert to integer indices
        ix, iy, iz = np.floor(local_pos).astype(int)
        
        # 4. Retrieve from 1D array
        idx = self.get_index(ix, iy, iz)
        return self.density[idx]
```

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Hard edges at the grid boundary can cause visual artifacts. S3 suggests restricting the sphere radius to $9/2$ and enforcing edges to 0 to reduce harsh edges [S3, unofficial].
*   **Memory Usage:** Dense grids require $O(N^3)$ memory. High resolutions quickly exhaust available RAM, limiting the detail that can be stored compared to sparse structures [S4, unofficial].
*   **Coordinate Mismatch:** The grid bounds in world space must match the transformation matrix used during ray marching. Mismatches cause the ray to sample outside the density field or sample incorrect regions [S1, S3, unofficial].

## Subtopics

### [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md)
Estimates surface normals using central differences on the density grid.

### [EdgeMap Data Layout](1109-edgemap-data-layout.md)
Stores geometric intersection data for isosurface extraction within the grid structure.

### [VoxelGrid Data Structure](1216-voxelgrid-data-structure.md)
A dense 3D grid representation for occupancy or labels, serving as the foundational memory layout.

### [Interpolated Discrete Values & Gradients](1268-interpolated-discrete-values-gradients.md)
Provides smooth sampling of the grid values using tri-linear or tri-cubic interpolation.

### [Euclidean Distance Field (EDF) Data Structure](3220-euclidean-distance-field-edf-data-structure.md)
Accelerates traversal by storing distance to the nearest occupied voxel.

### [Grid Resolution Heuristic](4871-grid-resolution-heuristic.md)
Determines optimal grid resolution to balance traversal and intersection costs.

## Operating characteristics
*   **Time Complexity:** Sampling is $O(1)$ because it involves a simple arithmetic index calculation and array access [S4, unofficial]. Ray-box intersection is also efficient due to the axis-aligned nature [S1, unofficial].
*   **Memory Complexity:** $O(N^3)$, where $N$ is the resolution per dimension. For $N=128$, this is $2,097,152$ floats (~8 MB) [S1, unofficial].
*   **Performance:** Faster to sample than sparse grids (like [Sparse Voxel Octrees](../sparse-voxel-octree.md)) because it avoids pointer indirection and tree traversal overhead [S4, unofficial].
*   **Limitations:** Memory consumption limits the maximum resolution, resulting in lower detail compared to sparse representations for large scenes [S4, unofficial]. Not suitable for highly sparse or extremely large domains without tiling or multi-resolution techniques.
*   **When to Use:** Suitable for scenes with moderate density, where sampling speed is critical, and the domain size is manageable (e.g., small-scale fluid sims, clouds within a defined volume) [S1, S4, unofficial].

## Background
The 3D grid density structure is a fundamental representation in volume rendering, originating from the need to store simulation results (such as fluid dynamics or smoke) in a quantized format [S1, unofficial]. It is analogous to how bitmaps represent 2D images [S1, unofficial]. The approach is widely used in educational ray tracers and real-time rendering for clouds and smoke [S1, S4, unofficial]. The specific transformation matrix formulation for world-to-local coordinate conversion is detailed in the context of volumetric path tracing [S3, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
[S3] PDF Volumetric Path Tracing, https://jugoodma.github.io/docs/740-paper.pdf, fetched 2026-10-27 (unofficial)
[S4] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
