# Grid Density Data Structure

Also called: 1D Array Density Layout, 3D Grid, 3D Voxel Grid Density Field, Dense Voxel Grid Storage, Dense structure, Linearized 3D Grid, Volume Grid Data Structure

## Overview
The Grid Density Data Structure is a dense, axis-aligned 3D grid representation used to store scalar density values (or coupled fields like temperature) for volume rendering [S1, unofficial][S2, unofficial]. It solves the problem of spatially organizing volumetric data to allow efficient, direct sampling during ray marching or isosurface extraction [S4, unofficial]. Unlike sparse representations such as [Octree](../octree.md) or [BitGrid3D](../bitgrid3d.md), this structure guarantees $O(1)$ access time by storing data in a contiguous linear array, trading higher memory consumption for faster sampling speeds and the ability to store precomputed auxiliary data like lighting [S4, unofficial]. It builds on the spatial locality principles of [Spatial Indexing and Ordering](1797-spatial-indexing-and-ordering.md) by mapping 3D coordinates to a 1D index via row-major strides.

## How it works
**Inputs:**
- Grid resolution $N$ (assumed equal in all dimensions, typically a power of 2: 8, 16, 32, 64, 128, etc.) [S1, unofficial].
- World-space bounding box defined by minimum point $P_{min}$ and maximum point $P_{max}$, where the extent $P_{max} - P_{min}$ is equal in all three dimensions [S1, unofficial].
- Optional: A transformation matrix $W$ mapping local grid coordinates to world coordinates, and its inverse $D = (W \times M)^{-1}$ for ray coordinate conversion [S3, unofficial].

**Outputs:**
- A contiguous 1D array of floating-point values representing the density field.
- Metadata defining the grid dimensions and bounds.

**Data Structures:**
The core structure `Grid` contains:
1.  `float *density`: A pointer to a contiguous memory block of size $N^3$.
2.  `size_t dimension`: The resolution $N$ in one dimension.
3.  `Point bounds[2]`: An array of two 3D points representing $P_{min}$ and $P_{max}$ in world space [S1, unofficial].

In more complex implementations (e.g., participating media), the structure may also include handles for auxiliary fields:
-   `GridHandleT m_density_handle`: Handle for the density grid.
-   `std::optional<GridHandleT> m_temperature_handle`: Optional handle for temperature data [S2, unofficial].

**Indexing Formula:**
Given integer grid coordinates $(x, y, z)$ where $0 \le x, y, z < N$, the 1D index $I$ in the linear array is calculated using row-major ordering [S3, unofficial]:
$$ I = (z \cdot N + y) \cdot N + x $$

**Coordinate Transformation:**
To sample the grid from a world-space ray, world coordinates must be converted to local grid coordinates. Let $P_{world}$ be a point in world space. The local coordinate $P_{local}$ is computed as [S3, unofficial]:
$$ P_{local} = D \times P_{world} $$
where $D$ is the transformation matrix from world to local space. The grid indices are then derived by scaling and clamping $P_{local}$ to the range $[0, N-1]$.

**Initialization and Population:**
1.  Allocate `density` array of size $N^3$.
2.  Initialize all elements to 0.
3.  Populate the grid with density values. For example, generating a spherical smoke distribution involves iterating through the interior voxels and computing distance from the center [S3, unofficial]:
    -   Center $C = (\frac{N-1}{2}, \frac{N-1}{2}, \frac{N-1}{2})$.
    -   For $z$ from $1$ to $N-2$, $y$ from $1$ to $N-2$, $x$ from $1$ to $N-2$:
        $$ dist = \sqrt{(x - C_x)^2 + (y - C_y)^2 + (z - C_z)^2} $$
        $$ density[I] = \frac{1}{2^{dist+2}} $$
    -   Note: Edges are often kept at 0 to reduce boundary artifacts [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Grid:
    def __init__(self, resolution, bounds_min, bounds_max):
        self.dimension = resolution
        self.bounds = [bounds_min, bounds_max]
        # Allocate contiguous 1D array
        self.density = [0.0] * (resolution ** 3)

    def set_density(self, x, y, z, value):
        """Set density at integer grid coordinates."""
        if 0 <= x < self.dimension and 0 <= y < self.dimension and 0 <= z < self.dimension:
            index = (z * self.dimension + y) * self.dimension + x
            self.density[index] = value

    def get_density(self, x, y, z):
        """Get density at integer grid coordinates."""
        if 0 <= x < self.dimension and 0 <= y < self.dimension and 0 <= z < self.dimension:
            index = (z * self.dimension + y) * self.dimension + x
            return self.density[index]
        return 0.0

    def generate_spherical_smoke(self):
        """Populate grid with a spherical smoke density field."""
        n = self.dimension
        center = (n - 1) / 2.0
        # Iterate interior to avoid boundary artifacts
        for z in range(1, n - 1):
            for y in range(1, n - 1):
                for x in range(1, n - 1):
                    dist = math.sqrt(
                        (x - center)**2 + 
                        (y - center)**2 + 
                        (z - center)**2
                    )
                    index = (z * n + y) * n + x
                    self.density[index] = 1.0 / (2**(dist + 2))

    def save_to_binary(self, filepath):
        """Save grid data to a binary file."""
        with open(filepath, 'wb') as f:
            for z in range(self.dimension):
                for y in range(self.dimension):
                    for x in range(self.dimension):
                        index = (z * self.dimension + y) * self.dimension + x
                        f.write(self.density[index].to_bytes(4, 'little'))
```

**Edge Cases and Pitfalls:**
-   **Memory Usage:** Memory scales as $O(N^3)$. High resolutions (e.g., 256+) become prohibitive for dense storage [S4, unofficial].
-   **Boundary Artifacts:** Sharp transitions at grid edges can cause visual artifacts. Setting edge voxels to 0 or using smooth falloffs mitigates this [S3, unofficial].
-   **Coordinate Precision:** When converting world rays to grid indices, floating-point precision errors can lead to sampling outside the grid bounds. Clamping is required.
-   **Non-Axis-Aligned Grids:** The basic structure assumes an axis-aligned bounding box (AABB). Non-axis-aligned grids require transformation matrices to map rays to grid space [S1, unofficial].

## Subtopics

### [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md)
Estimates spatial gradients using discrete neighbor sampling for shading. Chosen when surface normals are needed for lighting without the overhead of analytic derivatives.

### [EdgeMap Data Layout](1109-edgemap-data-layout.md)
Stores geometric intersection data for isosurface extraction. Chosen when explicit surface geometry is required rather than volumetric shading.

### [VoxelGrid Data Structure](1216-voxelgrid-data-structure.md)
A dense 3D grid storing per-voxel payloads (occupancy, labels). Chosen when richer per-voxel data beyond scalar density is needed.

### [Interpolated Discrete Values & Gradients](1268-interpolated-discrete-values-gradients.md)
Provides continuous sampling via tri-linear/cubic interpolation. Chosen when smooth density estimates and derivatives are required for ray marching.

### [Euclidean Distance Field (EDF) Data Structure](3220-euclidean-distance-field-edf-data-structure.md)
Stores distance magnitudes for space-skipping traversal. Chosen to accelerate rendering by skipping empty space.

### [Grid Resolution Heuristic](4871-grid-resolution-heuristic.md)
Determines optimal leaf-grid resolution. Chosen to balance traversal cost and primitive intersection cost in hierarchical structures.

## Operating characteristics
-   **Time Complexity:** Sampling is $O(1)$ per voxel access [S4, unofficial]. Ray marching requires $O(K)$ accesses where $K$ is the number of steps.
-   **Memory Complexity:** $O(N^3)$ where $N$ is the resolution per dimension. For $N=128$, this requires $128^3 \times 4$ bytes $\approx 8$ MB for single-precision floats [S1, unofficial].
-   **Performance:** Dense grids are faster to sample than sparse grids because they avoid indirection overhead (tree traversal or hash lookups) [S4, unofficial].
-   **Strengths:** Simple implementation, cache-friendly contiguous memory layout, fast random access, supports precomputed data (e.g., lighting) [S4, unofficial].
-   **Limitations:** High memory consumption limits resolution, especially for large scenes. Inefficient for sparse data (large empty regions waste memory) [S4, unofficial].
-   **When to Use:** Small to medium scenes, uniform density distribution, or when sampling speed is critical and memory is sufficient [S4, unofficial].
-   **When Not to Use:** Large scenes with sparse details, where sparse structures like [Octree](../octree.md) or [BitGrid3D](../bitgrid3d.md) are more appropriate [S4, unofficial].

## Background
The Grid Density Data Structure is a fundamental representation in volume rendering, originating from the need to discretize continuous volumetric fields for digital processing [S1, unofficial]. It is widely used in fluid simulation visualization, where results are stored in 3D grids of voxels [S1, unofficial]. The approach is detailed in educational resources on voxel-based volume rendering, which emphasize the simplicity of AABB grids and their integration with ray-box intersection routines [S1, unofficial]. It is also used in participating media rendering, where density grids model heterogeneous media like smoke or fog [S3, unofficial]. The structure is often loaded from sparse formats like OpenVDB and converted to dense grids for rendering [S4, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
[S3] PDF Volumetric Path Tracing, https://jugoodma.github.io/docs/740-paper.pdf, fetched 2026-10-27 (unofficial)
[S4] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
