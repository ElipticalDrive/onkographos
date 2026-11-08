# Voxel Grid Storage and Voxelization

Also called: Voxel Grid Storage

## Overview
Voxel Grid Storage and Voxelization converts continuous geometric representations into a discrete, dense 3D grid of scalar values, typically signed distances [S1, unofficial]. It establishes the initial data layout for volume rendering, differing from [Voxel Grid Construction](3085-voxel-grid-construction.md) by defining the memory layout and per-voxel evaluation logic, and from [Grid Background Change](3210-grid-background-change.md) by populating the grid rather than modifying it [G].

## How it works
**Inputs:**
- A geometric shape defined by a Signed Distance Function (SDF) or a mesh [S1, unofficial].
- A target voxel size (physical units per voxel) [S1, unofficial].
- Optional: An existing grid structure to fit or a shared grid for multi-mesh workflows [S3, unofficial].

**Outputs:**
- A 1D array of floats (`voxels`) containing the SDF values [S1, unofficial].
- Grid metadata: origin position, dimensions (width, height, depth), and voxel size [S1, unofficial].

**Data Structures:**
The scalar field is stored in a single 1D float array named `voxels` [S1, unofficial]. The grid dimensions are `width`, `height`, and `depth` [S1, unofficial]. Coordinates $(x, y, z)$ map to the array index using the formula:
$$ \text{index} = (z \times \text{width} \times \text{height}) + (y \times \text{width}) + x $$
This layout treats Z as the slowest-varying dimension, Y as the middle, and X as the fastest [S1, unofficial].

**Algorithm Steps:**

1.  **Determine Grid Bounds:**
    Calculate the world-space bounding box of the shape [S1, unofficial]. Let the bounding box minimum be $(b_{min_x}, b_{min_y}, b_{min_z})$ and maximum be $(b_{max_x}, b_{max_y}, b_{max_z})$ [G]. Compute the grid origin $(pos_x, pos_y, pos_z)$ by flooring the bounding box minimum coordinates scaled by the voxel size reciprocal, then subtracting one voxel size [S1, unofficial].
    $$ \text{voxel\_size\_reciprocal} = \frac{1}{\text{voxel\_size}} $$
    $$ pos_x = \lfloor b_{min_x} \times \text{voxel\_size\_reciprocal} \rfloor - \text{voxel\_size} $$
    (Similarly for $pos_y$ and $pos_z$).

2.  **Compute Grid Dimensions:**
    Calculate the grid dimensions by ceiling the bounding box extents scaled by the voxel size reciprocal, then adding 2 voxels to each dimension for padding [S1, unofficial].
    $$ \text{width} = \lceil (b_{max_x} - b_{min_x}) \times \text{voxel\_size\_reciprocal} \rceil + 2 $$
    $$ \text{height} = \lceil (b_{max_y} - b_{min_y}) \times \text{voxel\_size\_reciprocal} \rceil + 2 $$
    $$ \text{depth} = \lceil (b_{max_z} - b_{min_z}) \times \text{voxel\_size\_reciprocal} \rceil + 2 $$

3.  **Allocate Memory:**
    Calculate the required number of voxels: $N = \text{width} \times \text{height} \times \text{depth}$ [G]. If the required number of voxels exceeds the current array length, resize the `voxels` array to the nearest power of 2 greater than or equal to $N$ using a bitwise rounding-up algorithm [S1, unofficial].

4.  **Voxelize (Evaluate SDF):**
    Iterate through all voxel coordinates in nested loops: $z$ from 0 to `depth`, $y$ from 0 to `height`, $x$ from 0 to `width` [S1, unofficial]. For each coordinate $(x, y, z)$:
    a. Compute the world-space center position using the grid origin and voxel size [S1, unofficial].
    b. Transform this point to the shape's local coordinate system [S1, unofficial].
    c. Evaluate the shape's signed distance function (SDF) at the local point [S1, unofficial].
    d. Store the resulting distance value in the `voxels` array at the index calculated by the layout formula [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def next_power_of_2(n: int) -> int:
    """Returns the smallest power of 2 greater than or equal to n."""
    if n <= 0:
        return 1
    return 1 << (n - 1).bit_length()

def compute_grid_dimensions(bmin: np.ndarray, bmax: np.ndarray, voxel_size: float) -> tuple:
    """
    Computes grid origin and dimensions based on bounding box.
    bmin, bmax: numpy arrays of shape (3,)
    voxel_size: float
    Returns: (origin, width, height, depth)
    """
    voxel_size_reciprocal = 1.0 / voxel_size
    
    # Compute origin: floor(min * reciprocal) - voxel_size
    origin = np.floor(bmin * voxel_size_reciprocal) - voxel_size
    
    # Compute dimensions: ceil(extent * reciprocal) + 2
    extent = bmax - bmin
    width = int(math.ceil(extent[0] * voxel_size_reciprocal)) + 2
    height = int(math.ceil(extent[1] * voxel_size_reciprocal)) + 2
    depth = int(math.ceil(extent[2] * voxel_size_reciprocal)) + 2
    
    return origin, width, height, depth

def voxelize_shape(sdf_func, bmin: np.ndarray, bmax: np.ndarray, voxel_size: float) -> dict:
    """
    Voxelize a shape defined by an SDF into a 1D array.
    sdf_func: callable taking a local 3D point and returning a float distance.
    """
    origin, width, height, depth = compute_grid_dimensions(bmin, bmax, voxel_size)
    
    required_voxels = width * height * depth
    array_size = next_power_of_2(required_voxels)
    
    # Initialize with a large default value for out-of-bounds safety
    voxels = np.full(array_size, 10000.0, dtype=np.float32)
    
    # Pre-calculate constants
    voxel_size_reciprocal = 1.0 / voxel_size
    half_voxel = 0.5 * voxel_size
    
    # Nested loops: z (slowest), y, x (fastest)
    for z in range(depth):
        for y in range(height):
            for x in range(width):
                # Compute world-space center
                world_x = origin[0] + (x + 0.5) * voxel_size
                world_y = origin[1] + (y + 0.5) * voxel_size
                world_z = origin[2] + (z + 0.5) * voxel_size
                
                # Transform to local space (identity assumed if shape is at origin)
                local_point = np.array([world_x, world_y, world_z])
                
                # Evaluate SDF
                distance = sdf_func(local_point)
                
                # Store in 1D array
                index = (z * width * height) + (y * width) + x
                voxels[index] = distance
                
    return {
        "voxels": voxels,
        "origin": origin,
        "width": width,
        "height": height,
        "depth": depth,
        "voxel_size": voxel_size
    }

def get_voxel(voxels: np.ndarray, x: int, y: int, z: int, width: int, height: int, depth: int) -> float:
    """
    Retrieves a voxel value. Returns 10000.0 if out of bounds.
    """
    if x < 0 or x >= width or y < 0 or y >= height or z < 0 or z >= depth:
        return 10000.0
    
    index = (z * width * height) + (y * width) + x
    return voxels[index]
```

**Parameters:**
- `DEFAULT_VOXEL_SIZE`: A static float constant set to `0.1f`, representing the default physical size of each voxel in world space [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Out-of-Bounds Access:** Accessing a voxel coordinate outside the grid bounds does not throw an error [S1, unofficial]. Instead, the `getVoxel` method returns a large default value of `10_000.0f` to indicate out-of-bounds access [S1, unofficial].
- **Memory Allocation:** If the required number of voxels is not a power of 2, the array is resized to the next power of 2 [S1, unofficial].

## Subtopics

### [VoxelizationGrid](6601-voxelizationgrid.md)
A data structure representing an explicit voxelization grid, defining bounds, grid dimensions, and voxel sizes in 3D space. It solves the problem of converting arbitrary mesh bounding boxes into a discrete, aligned 3D grid suitable for dense volume storage, ensuring that grid indices map consistently to world coordinates.

### Shared-grid Voxelization
A workflow where multiple meshes are voxelized onto a single, explicitly defined voxel grid, typically defined by the largest or outer mesh [S3, unofficial].
- **Definition:** Multiple meshes are aligned and stored in the same grid space [S3, unofficial].
- **Steps:** Build shared grid by loading the outer mesh and calling `build_voxelization_grid(outer_mesh, voxel_size=voxel_size, pixel_size=pixel_size)` to create `shared_grid` [S3, unofficial]. Calculate translation vector `translation_xyz = (outer_mesh.center - inner_mesh.center).astype(np.float32)` [S3, unofficial]. Voxelize outer mesh by calling `voxelize_on_grid(outer_mesh, grid=shared_grid, mode="solid")` to produce `outer_grid` [S3, unofficial]. Voxelize inner mesh by calling `voxelize_on_grid(inner_mesh, grid=shared_grid, mode="solid", translation_xyz=translation_xyz)` to produce `inner_grid`, applying the translation to align it with the outer mesh's coordinate system [S3, unofficial].
- **Parameters:** `mode` (str, default "solid"): Passed to `voxelize_on_grid`, specifies solid voxelization [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(W \times H \times D)$, where $W, H, D$ are the grid dimensions [G]. Each voxel requires one SDF evaluation and coordinate transformation [S1, unofficial].
- **Memory Complexity:** $O(W \times H \times D)$ for the 1D float array [S1, unofficial]. The array size is rounded up to the nearest power of 2, potentially increasing memory usage [S1, unofficial]. Alternative implementations store occupancy as a 3D array of booleans (`bool *** data`), which is memory inefficient because each boolean typically occupies a full byte rather than a single bit [S2, unofficial].
- **Strengths:** Simple memory layout, easy to implement, supports arbitrary shapes via SDFs [S1, unofficial].
- **Limitations:** Dense storage is memory-intensive for large volumes [G]. Padding adds overhead [S1, unofficial]. Out-of-bounds handling relies on a sentinel value rather than bounds checking, which can mask logic errors if not handled carefully [S1, unofficial].

## Background
This method is derived from implementations in open-source projects for isosurface extraction and voxelization [S1, unofficial]. The core algorithm for fitting the grid and evaluating the SDF is based on the `IsosurfaceExtraction` project [S1, unofficial]. The shared-grid workflow is drawn from the `Voxelerate` project [S3, unofficial].

## Sources
[S1] Chickenkeeper/IsosurfaceExtraction, https://github.com/Chickenkeeper/IsosurfaceExtraction, fetched 2026-10-27 (unofficial)
[S2] antonmartynov/OBJ-Solidizer-And-Voxelizer, https://github.com/antonmartynov/OBJ-Solidizer-And-Voxelizer, fetched 2026-10-27 (unofficial)
[S3] hsafari68/Voxelerate, https://github.com/hsafari68/Voxelerate, fetched 2026-10-26 (unofficial)
