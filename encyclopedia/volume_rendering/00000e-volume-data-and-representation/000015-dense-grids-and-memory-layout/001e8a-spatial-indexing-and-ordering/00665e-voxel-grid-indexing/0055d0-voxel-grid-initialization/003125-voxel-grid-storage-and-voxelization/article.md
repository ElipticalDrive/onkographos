# Voxel Grid Storage and Voxelization

Also called: Voxel Grid Storage

## Overview
Voxel Grid Storage and Voxelization is the process of converting continuous geometric representations (such as meshes or signed distance fields) into a discrete, dense 3D grid of scalar or boolean values [S1, S2, unofficial]. It solves the problem of representing arbitrary 3D shapes in a format suitable for ray marching, marching cubes, or GPU-based rendering by mapping world-space coordinates to array indices [S1, S3, unofficial]. It differs from sparse representations (like [Adaptive Voxel Hashing Refined by Depth Residual](17590-adaptive-voxel-hashing-refined-by-depth-residual.md)) by allocating memory for the entire bounding volume, and from [Instanced Voxel Rendering](10592-instanced-voxel-rendering.md) by focusing on the data structure and conversion logic rather than the rendering pipeline [S1, S2, S3, unofficial].

## How it works
**Inputs:**
- A geometric primitive (mesh or SDF) [S1, S3, unofficial].
- A target voxel size (physical units per voxel) [S1, S3, unofficial].
- Optional: A pre-defined grid origin and dimensions [S3, unofficial].

**Outputs:**
- A voxel grid data structure containing the discretized representation [S1, S2, unofficial].

**Data Structures:**
Two primary layouts are documented:
1.  **1D Float Array (SDF):** A single 1D array of floats named `voxels`. The grid dimensions are `width`, `height`, and `depth`. Coordinates $(x, y, z)$ map to the array index using the formula:
    $$ \text{index} = (z \times \text{width} \times \text{height}) + (y \times \text{width}) + x $$
    This layout treats Z as the slowest-varying dimension, Y as the middle, and X as the fastest [S1, unofficial].
2.  **3D Boolean Array (Occupancy):** A 3D array of booleans `bool *** data`, allocated as `new bool ** [dimensions.x.count]` with nested loops for Y and Z [S2, unofficial].

**Parameters:**
- `DEFAULT_VOXEL_SIZE`: A static float constant set to `0.1f`, representing the default physical size of each voxel in world space [S1, unofficial].
- `mode`: A string parameter (default "solid") passed to voxelization functions to specify solid voxelization [S3, unofficial].

**Steps:**

1.  **Grid Initialization (Fit to Shape):**
    If the grid dimensions are not pre-defined, calculate them based on the shape's bounding box [S1, unofficial]:
    a.  Calculate the world-space bounding box of the shape.
    b.  Compute the grid origin $(posX, posY, posZ)$ by flooring the bounding box minimum coordinates scaled by the voxel size reciprocal, then subtracting one voxel size:
        $$ \text{origin} = \lfloor \frac{\text{bbox\_min}}{\text{voxel\_size}} \rfloor - 1 $$
    c.  Compute the grid dimensions $(\text{width}, \text{height}, \text{depth})$ by ceiling the bounding box extents scaled by the voxel size reciprocal, then adding 2 voxels to each dimension for padding:
        $$ \text{dim} = \lceil \frac{\text{bbox\_extent}}{\text{voxel\_size}} \rceil + 2 $$

2.  **Memory Allocation:**
    Before allocation, if the required number of voxels ($\text{width} \times \text{height} \times \text{depth}$) exceeds the current array length, resize the voxels array to the nearest power of 2 greater than or equal to the required size using a bitwise rounding-up algorithm [S1, unofficial]. For the boolean grid, allocate the 3D pointer structure as described in the layout section [S2, unofficial].

3.  **Voxelization Loop:**
    Iterate through all voxel coordinates in nested loops: $z$ from 0 to $\text{depth}$, $y$ from 0 to $\text{height}$, $x$ from 0 to $\text{width}$ [S1, unofficial].
    a.  Compute the world-space center position using `getVoxelCenterPos(x, y, z)` [S1, unofficial].
    b.  Transform this point to the shape's local coordinate system [S1, unofficial].
    c.  Evaluate the shape's signed distance function (SDF) at the local point [S1, unofficial].
    d.  Store the resulting distance value in the voxel array at $(x, y, z)$ [S1, unofficial].
    *Note: For boolean occupancy grids, the evaluation typically checks if the point is inside the mesh [S2, S3, unofficial].*

4.  **Shared-Grid Voxelization (Variant):**
    When voxelizing multiple meshes onto a single grid [S3, unofficial]:
    a.  Build the shared grid using the outer mesh: `shared_grid = build_voxelization_grid(outer_mesh, voxel_size, pixel_size)` [S3, unofficial].
    b.  Voxelize the outer mesh: `outer_grid = voxelize_on_grid(outer_mesh, grid=shared_grid, mode="solid")` [S3, unofficial].
    c.  Calculate the translation vector: `translation_xyz = (outer_mesh.center - inner_mesh.center).astype(np.float32)` [S3, unofficial].
    d.  Voxelize the inner mesh with translation: `inner_grid = voxelize_on_grid(inner_mesh, grid=shared_grid, mode="solid", translation_xyz=translation_xyz)` [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def get_voxel_center_pos(x: int, y: int, z: int, origin: np.ndarray, voxel_size: float) -> np.ndarray:
    """Compute world-space center of a voxel given its grid coordinates."""
    return origin + np.array([x, y, z], dtype=np.float32) * voxel_size

def resize_array_to_power_of_two(current_length: int, required_length: int) -> int:
    """Resize array to nearest power of 2 >= required_length."""
    if required_length <= current_length:
        return current_length
    # Bitwise rounding up to next power of 2
    power = 1
    while power < required_length:
        power <<= 1
    return power

def voxelize_sdf(mesh, voxel_size: float = 0.1f) -> dict:
    """
    Voxelize a mesh into a 1D float array representing an SDF.
    Returns a dict with 'voxels' (1D array), 'width', 'height', 'depth', 'origin'.
    """
    # 1. Get bounding box
    bbox_min, bbox_max = mesh.get_world_space_bounding_box()
    bbox_extent = bbox_max - bbox_min
    
    # 2. Compute grid dimensions and origin
    # origin = floor(bbox_min / voxel_size) - 1
    origin = np.floor(bbox_min / voxel_size).astype(np.float32) - np.array([voxel_size, voxel_size, voxel_size], dtype=np.float32)
    
    # dim = ceil(bbox_extent / voxel_size) + 2
    width = int(math.ceil(bbox_extent[0] / voxel_size)) + 2
    height = int(math.ceil(bbox_extent[1] / voxel_size)) + 2
    depth = int(math.ceil(bbox_extent[2] / voxel_size)) + 2
    
    # 3. Allocate memory
    required_voxels = width * height * depth
    # Assuming initial array is empty or needs resizing
    allocated_size = resize_array_to_power_of_two(0, required_voxels)
    voxels = np.full(allocated_size, 10000.0, dtype=np.float32) # Default out-of-bounds value
    
    # 4. Iterate and evaluate
    for z in range(depth):
        for y in range(height):
            for x in range(width):
                # Compute world center
                world_pos = get_voxel_center_pos(x, y, z, origin, voxel_size)
                
                # Transform to local space (inverse transform of mesh)
                local_pos = mesh.world_to_local(world_pos)
                
                # Evaluate SDF
                sdf_value = mesh.sdf(local_pos)
                
                # Store in 1D array
                index = (z * width * height) + (y * width) + x
                voxels[index] = sdf_value
                
    return {
        "voxels": voxels,
        "width": width,
        "height": height,
        "depth": depth,
        "origin": origin,
        "voxel_size": voxel_size
    }

def voxelize_boolean(mesh, voxel_size: float = 0.1f) -> dict:
    """
    Voxelize a mesh into a 3D boolean array representing occupancy.
    Returns a dict with 'data' (3D bool array), 'width', 'height', 'depth', 'origin'.
    """
    # 1. Get bounding box
    bbox_min, bbox_max = mesh.get_world_space_bounding_box()
    bbox_extent = bbox_max - bbox_min
    
    # 2. Compute grid dimensions and origin
    origin = np.floor(bbox_min / voxel_size).astype(np.float32) - np.array([voxel_size, voxel_size, voxel_size], dtype=np.float32)
    width = int(math.ceil(bbox_extent[0] / voxel_size)) + 2
    height = int(math.ceil(bbox_extent[1] / voxel_size)) + 2
    depth = int(math.ceil(bbox_extent[2] / voxel_size)) + 2
    
    # 3. Allocate 3D boolean array
    # data[x][y][z]
    data = [[[False for _ in range(depth)] for _ in range(height)] for _ in range(width)]
    
    # 4. Iterate and evaluate
    for x in range(width):
        for y in range(height):
            for z in range(depth):
                world_pos = get_voxel_center_pos(x, y, z, origin, voxel_size)
                local_pos = mesh.world_to_local(world_pos)
                
                # Check if inside mesh
                is_inside = mesh.is_inside(local_pos)
                data[x][y][z] = is_inside
                
    return {
        "data": data,
        "width": width,
        "height": height,
        "depth": depth,
        "origin": origin,
        "voxel_size": voxel_size
    }

def voxelize_shared_grid(outer_mesh, inner_mesh, voxel_size: float = 0.1f) -> dict:
    """
    Voxelize multiple meshes onto a single shared grid.
    """
    # Build shared grid from outer mesh
    shared_grid = build_voxelization_grid(outer_mesh, voxel_size=voxel_size, pixel_size=voxel_size)
    
    # Voxelize outer mesh
    outer_grid = voxelize_on_grid(outer_mesh, grid=shared_grid, mode="solid")
    
    # Calculate translation
    translation_xyz = (outer_mesh.center - inner_mesh.center).astype(np.float32)
    
    # Voxelize inner mesh with translation
    inner_grid = voxelize_on_grid(inner_mesh, grid=shared_grid, mode="solid", translation_xyz=translation_xyz)
    
    return {
        "shared_grid": shared_grid,
        "outer_grid": outer_grid,
        "inner_grid": inner_grid
    }
```

**Edge Cases and Pitfalls:**
-   **Out-of-Bounds Access:** Accessing a voxel coordinate outside the grid bounds does not throw an error. Instead, the `getVoxel` method returns a large default value of `10_000.0f` to indicate out-of-bounds access [S1, unofficial].
-   **Memory Efficiency:** Using `bool` for voxel storage is memory inefficient because each boolean typically occupies a full byte rather than a single bit [S2, unofficial].
-   **Precision:** The bitwise rounding-up algorithm for array resizing ensures power-of-two sizes but may waste memory if the required size is just above a power of two [S1, unofficial].

## Subtopics

### [VoxelizationGrid](6601-voxelizationgrid.md)
`VoxelizationGrid` is a data structure that represents an explicit voxelization grid, defining spatial bounds, grid dimensions, and voxel sizes in 3D space. It serves as a base class for managing a 3D voxel grid in world space, providing utilities for coordinate conversion, ray intersection, and mesh generation. It is distinct from sibling methods like [Instanced Voxel Rendering](10592-instanced-voxel-rendering.md) by focusing on the foundational storage and indexing logic rather than the rendering pipeline itself.

### [Instanced Voxel Rendering](10592-instanced-voxel-rendering.md)
Instanced Voxel Rendering is a technique for visualizing sparse voxel grids by generating a single unit cube mesh and drawing it multiple times at different positions using OpenGL instanced rendering, thereby reducing memory overhead compared to storing full geometry for every voxel. It differs from [VoxelizationGrid](6601-voxelizationgrid.md) by explicitly leveraging GPU instancing (`glDrawElementsInstanced`) to render discrete voxel positions, whereas VoxelizationGrid may imply a denser or differently structured grid representation without necessarily optimizing for instance count reduction via this specific API call pattern.

### [Adaptive Voxel Hashing Refined by Depth Residual](17590-adaptive-voxel-hashing-refined-by-depth-residual.md)
Adaptive Voxel Hashing Refined by Depth Residual is a hierarchical volume representation that fuses depth data into a stack of uniform voxel grids with halving voxel sizes, refining only those blocks where the depth systematically deviates from the coarser surface. It replaces the retired octree trial by using depth residuals to drive dynamic allocation, allowing finer resolution (e.g., 5 mm) only where necessary while maintaining free-space carving across all levels. Unlike uniform grids, it reduces memory usage and meshing overhead by concentrating detail, and unlike octrees, it leverages existing uniform grid infrastructure for stamps, garbage collection, and extraction.

### Shared-grid Voxelization
**Definition:** Workflow where multiple meshes are voxelized onto a single, explicitly defined voxel grid, typically defined by the largest or outer mesh [S3, unofficial].
**Steps:**
1.  Build shared grid: Load outer mesh. Call `build_voxelization_grid(outer_mesh, voxel_size=voxel_size, pixel_size=pixel_size)` to create `shared_grid`. Calculate translation vector `translation_xyz = (outer_mesh.center - inner_mesh.center).astype(np.float32)` [S3, unofficial].
2.  Voxelize outer mesh: Call `voxelize_on_grid(outer_mesh, grid=shared_grid, mode="solid")` to produce `outer_grid` [S3, unofficial].
3.  Voxelize inner mesh: Call `voxelize_on_grid(inner_mesh, grid=shared_grid, mode="solid", translation_xyz=translation_xyz)` to produce `inner_grid`, applying the translation to align it with the outer mesh's coordinate system [S3, unofficial].
**Parameters:** `mode` (str, default "solid"): Passed to `voxelize_on_grid`, specifies solid voxelization [S3, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(N)$ where $N$ is the total number of voxels in the grid ($\text{width} \times \text{height} \times \text{depth}$) [S1, unofficial]. Each voxel requires a constant-time SDF or inside/outside evaluation.
-   **Memory Complexity:** $O(N)$ for the voxel array. For float SDFs, this is $4N$ bytes. For boolean occupancy, it is $1N$ byte (inefficient) or potentially $N/8$ bytes if bit-packed (not implemented in the cited sources) [S1, S2, unofficial].
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** Linear scaling with grid resolution. Doubling the resolution in each dimension increases memory and time by a factor of 8 [S1, unofficial].
-   **Strengths:** Simple implementation; fast random access; compatible with standard ray marching and marching cubes algorithms [S1, S3, unofficial].
-   **Limitations:** High memory usage for large volumes or high resolutions; inefficient for sparse scenes compared to sparse voxel structures [S1, S2, S3, unofficial].
-   **When to use:** When the scene is dense or the bounding volume is small relative to the world; when simple, fast access is required [S1, S3, unofficial].
-   **When not to use:** For very large, sparse scenes; when memory is constrained [S2, S3, unofficial].

## Background
The method originates from implementations in Java (JavaFX) and C++ for isosurface extraction and voxelization [S1, S2, unofficial]. The Java implementation (`Chickenkeeper/IsosurfaceExtraction`) uses a 1D float array for SDF storage and a power-of-two resizing strategy [S1, unofficial]. The C++ implementation (`antonmartynov/OBJ-Solidizer-And-Voxelizer`) uses a 3D boolean array for occupancy [S2, unofficial]. The Python implementation (`hsafari68/Voxelerate`) introduces shared-grid voxelization for multi-mesh workflows [S3, unofficial]. These methods are used in 3D modeling tools, 3D printing slicers, and real-time rendering engines for converting mesh data into volumetric representations [S1, S2, S3, unofficial].

## Sources
[S1] Chickenkeeper/IsosurfaceExtraction, https://github.com/Chickenkeeper/IsosurfaceExtraction, fetched 2026-10-27 (unofficial)
[S2] antonmartynov/OBJ-Solidizer-And-Voxelizer, https://github.com/antonmartynov/OBJ-Solidizer-And-Voxelizer, fetched 2026-10-27 (unofficial)
[S3] hsafari68/Voxelerate, https://github.com/hsafari68/Voxelerate, fetched 2026-10-26 (unofficial)
