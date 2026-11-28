# Z-list-buffer Ray Representation

Also called: Z-list-buffer, Ray-Rep, Array of Rays

## Overview
The Z-list-buffer Ray Representation is a volume data storage format that stores voxel data as an array of rays, where each ray contains a list of sample points (position and normal) defining regions [S2]. It replaces traditional scalar fields to reduce storage and preserve sharp features [S2]. It supports any cell-based triangulation method [S1]. It is distinguished from sibling structures by its specific generation method using orthographic projection and a Z-buffer with a Z-list, which is faster than modified ray tracing for axis-aligned rays [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Hermite data (position and normal) at a fine resolution grid of size $n_k \times n_k \times n_k$ [S2].
*   **Output:** Three arrays of rays (one for each axis: x, y, z), where each array has a resolution of $(n_k + 1) \times (n_k + 1)$ [S2].

**Data Structures:**
1.  **Sample:** A structure containing:
    *   `position`: 3D point $(x, y, z)$ [S2].
    *   `normal`: 3D vector $(nx, ny, nz)$ [S2].
2.  **Ray:** A list of `Sample` objects [S2].
    *   Samples are classified as **left bounding** (marks the left boundary of a region) or **right bounding** (marks the right boundary of a region) [S2].
    *   A pair of samples defines a region: the first sample is the left boundary, the second is the right boundary [S2].
3.  **Ray Array:** An array of `Ray` objects [S2]. The resolution is $(n_k + 1) \times (n_k + 1)$ [S2]. Three such arrays are maintained, one for each axis [S2].

**Algorithm Steps:**
1.  Acquire Hermite data at a uniform $n_k \times n_k \times n_k$ grid resolution [S2].
2.  Shoot axis-aligned rays along the x, y, and z axes [S2].
3.  For each ray, compute all intersection points with the volume data and their corresponding normals [S2].
4.  Store these intersection points as `Sample` objects in the corresponding `Ray` list [S2].
5.  Classify each sample as left or right bounding based on the region it defines [S2].
6.  Use orthographic projection and a Z-buffer with a Z-list for each entry to perform scan conversion, which is faster than modified ray tracing for this specific axis-aligned case [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_z_list_buffer(hermite_data, n_k):
    """
    Generates Z-list-buffer Ray Representation from Hermite data.
    
    Args:
        hermite_data: 3D grid of (position, normal) pairs at resolution n_k x n_k x n_k.
        n_k: Integer, finest resolution of the input grid.
        
    Returns:
        rays_x, rays_y, rays_z: Three 2D arrays of Ray objects, each of size (n_k + 1) x (n_k + 1).
    """
    # Initialize three 2D arrays of rays, each of size (n_k + 1) x (n_k + 1)
    rays_x = [[[] for _ in range(n_k + 1)] for _ in range(n_k + 1)]
    rays_y = [[[] for _ in range(n_k + 1)] for _ in range(n_k + 1)]
    rays_z = [[[] for _ in range(n_k + 1)] for _ in range(n_k + 1)]
    
    # The source specifies using orthographic projection and a Z-buffer with a Z-list
    # for faster scan conversion than modified ray tracing [S2].
    # We iterate over the 2D grid for each axis, project rays, and accumulate intersections.
    
    # Process X-axis rays (rays parallel to X, indexed by Y, Z)
    for y in range(n_k + 1):
        for z in range(n_k + 1):
            z_buffer = float('inf')
            z_list = []
            # Scan convert along the ray direction (X-axis)
            for x in range(n_k + 1):
                # Evaluate Hermite data to find surface crossings along X
                # The source does not specify the exact intersection routine [S2].
                point, normal = None, None 
                if point is not None:
                    # Z-buffer check: only keep intersections closer than current depth
                    if x < z_buffer:
                        z_buffer = x
                        # Classify as left or right bounding based on region alternation
                        sample_type = "left_bounding" if len(z_list) % 2 == 0 else "right_bounding"
                        z_list.append({"position": point, "normal": normal, "type": sample_type})
            rays_x[y][z] = z_list

    # Repeat for Y and Z axes similarly
    for y in range(n_k + 1):
        for z in range(n_k + 1):
            z_buffer = float('inf')
            z_list = []
            for x in range(n_k + 1):
                point, normal = None, None
                if point is not None:
                    if y < z_buffer:
                        z_buffer = y
                        sample_type = "left_bounding" if len(z_list) % 2 == 0 else "right_bounding"
                        z_list.append({"position": point, "normal": normal, "type": sample_type})
            rays_y[z][x] = z_list

    for x in range(n_k + 1):
        for y in range(n_k + 1):
            z_buffer = float('inf')
            z_list = []
            for z in range(n_k + 1):
                point, normal = None, None
                if point is not None:
                    if z < z_buffer:
                        z_buffer = z
                        sample_type = "left_bounding" if len(z_list) % 2 == 0 else "right_bounding"
                        z_list.append({"position": point, "normal": normal, "type": sample_type})
            rays_z[y][x] = z_list

    return rays_x, rays_y, rays_z
```

**Parameters:**
*   $n_k$: The finest resolution of the input Hermite data grid. Typical values are not specified, but the example uses a $512^3$ grid for comparison [S2].
*   Average number of samples per ray: ~2.86 for a dragon model, ~2.05 for a fandisk model [S2]. This is significantly lower than the 512 samples required for a $512^3$ distance field [S2].

**Edge Cases and Pitfalls:**
*   **Redundancy:** The representation is redundant and requires regularization [S1].
*   **Regularization:** Separate blocks of voxel terrain are best stored as triangular meshes and converted to Ray-Rep and back as needed [S1].
*   **Precision:** The source notes that the representation is precise, but does not specify numerical precision issues. [G] In practice, floating-point precision may affect intersection calculations, especially for thin features.

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. The generation is faster than modified ray tracing due to the use of orthographic projection and Z-buffer with Z-list [S2].
*   **Memory Complexity:** Redundant, requiring regularization [S1]. However, it reduces storage compared to scalar fields [S2]. For a $512^3$ distance field, it reduces the number of samples per ray from 512 to approximately 2 [S2].
*   **Measured Performance:** Average of ~2.86 samples per ray for a dragon model and ~2.05 for a fandisk model [S2].
*   **Strengths:** High speed for triangulation using dual methods, preserves sharp features, reduces storage compared to scalar fields [S1, S2].
*   **Limitations:** Redundant, requires regularization, higher storage than compressible formats [S1].
*   **When to use:** When fast rendering from polygonal representation and fast CSG operations are required, such as in virtual world sculpting [S2].
*   **When not to use:** When storage space is a critical constraint and compression is available for other formats [S1].

## Background
*   **Origin:** The concept was introduced in the paper "Detail sculpting using cubical marching squares" [S2]. The term "Z-list-buffer" was used in the same paper to describe the generation method [S2].
*   **History:** The GPU implementation of Ray-Rep with normals is called Layered Depth-Normal Images (LDNI) [S1]. Before GPGPU, layered depth images (LDI) used texture arrays to store depth and normal [S1]. With CUDA, OpenCL, and compute shaders, it became possible to build lists of points for each image fragment on the GPU [S1].
*   **Usage:** Used in virtual world sculpting, which involves continuous subtraction operations on target objects [S2]. It is also used for voxel terrain storage in VR system design [S1].

## Sources
[S1] REPRESENTATION AND STORAGE OF VOXEL TERRAIN FOR DESIGNING VIRTUAL REALITY SYSTEMS, https://doi.org/10.26102/2310-6018/2019.24.1.019, fetched 2026-10-26
[S2] Detail sculpting using cubical marching squares, https://doi.org/10.1145/1152399.1152402, fetched 2026-11-06
