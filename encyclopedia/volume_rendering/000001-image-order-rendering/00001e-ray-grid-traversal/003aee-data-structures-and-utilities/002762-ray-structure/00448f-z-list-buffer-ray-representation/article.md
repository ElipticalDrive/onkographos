# Z-list-buffer Ray Representation

Also called: Z-list-buffer, Ray Representation, Ray-Rep, Array of Rays

## Overview
The Z-list-buffer Ray Representation is a volume data storage format that stores voxel data as an array of rays, where each ray contains a list of sample points (position and normal) defining regions [S2]. It replaces traditional scalar fields to reduce storage and preserve sharp features, supporting any cell-based triangulation method [S1, S2]. It is distinguished from sibling structures by its specific generation method using orthographic projection and a Z-buffer with a Z-list, which is faster than modified ray tracing for axis-aligned rays [S2].

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
3.  **Ray Array:** An array of `Ray` objects. The resolution is $(n_k + 1) \times (n_k + 1)$ [S2]. Three such arrays are maintained, one for each axis [S2].

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
    # Initialize arrays of rays. Each ray is a list of samples.
    # Resolution is (n_k + 1) x (n_k + 1)
    rays_x = [[[] for _ in range(n_k + 1)] for _ in range(n_k + 1)]
    rays_y = [[[] for _ in range(n_k + 1)] for _ in range(n_k + 1)]
    rays_z = [[[] for _ in range(n_k + 1)] for _ in range(n_k + 1)]
    
    # Step 1 & 2: Shoot axis-aligned rays and compute intersections
    # Note: The source specifies using orthographic projection and Z-buffer with Z-list
    # for faster scan conversion than modified ray tracing.
    # This implies a rasterization-like approach rather than analytical ray marching.
    
    # For each axis, we iterate through the grid cells to determine ray intersections.
    # Since the input is Hermite data (implicit surface), we must find where rays intersect the surface.
    
    # Example for X-axis rays:
    # Rays are parallel to X-axis. For each (y, z) coordinate in the (n_k + 1) x (n_k + 1) grid:
    for y in range(n_k + 1):
        for z in range(n_k + 1):
            # Define the ray: origin (x_min, y, z), direction (1, 0, 0)
            # x_min and x_max are the bounds of the volume
            ray_origin = (0.0, y, z) # Assuming normalized coordinates or grid indices
            ray_direction = (1.0, 0.0, 0.0)
            
            # Find all intersection points of this ray with the volume defined by hermite_data
            # This step typically involves evaluating the implicit function or using the Hermite data
            # to determine surface crossings.
            intersections = find_intersections(ray_origin, ray_direction, hermite_data)
            
            # Step 3 & 4: Store intersection points and normals as samples
            ray_samples = []
            for i, point in enumerate(intersections):
                # point is (x, y, z)
                # normal is computed from Hermite data at the intersection
                normal = compute_normal(point, hermite_data)
                
                # Step 5: Classify as left or right bounding
                # In 1D, samples alternate: left, right, left, right...
                # The first sample in the list is the left boundary of the first region.
                if i % 2 == 0:
                    sample_type = "left_bounding"
                else:
                    sample_type = "right_bounding"
                    
                sample = {
                    "position": point,
                    "normal": normal,
                    "type": sample_type
                }
                ray_samples.append(sample)
            
            # Store in the corresponding ray array
            rays_x[y][z] = ray_samples
            
    # Repeat for Y and Z axes similarly
    # ... (code for Y and Z axes omitted for brevity, following same logic)
    
    return rays_x, rays_y, rays_z

def find_intersections(ray_origin, ray_direction, hermite_data):
    """
    Finds all intersection points of a ray with the volume defined by hermite_data.
    This is a placeholder for the specific intersection algorithm, which may involve
    evaluating the implicit function derived from Hermite data.
    """
    # Implementation detail: This function would evaluate the implicit surface
    # defined by the Hermite data along the ray and find roots.
    # The source notes that this is done via scan conversion using orthographic projection
    # and a Z-buffer with a Z-list, which suggests a rasterization-based approach
    # rather than analytical root finding.
    # However, the core concept is finding the intersection points.
    return [] # Placeholder

def compute_normal(point, hermite_data):
    """
    Computes the normal vector at a given point using the Hermite data.
    """
    # Implementation detail: The normal is typically the gradient of the implicit function.
    # Hermite data provides both position and normal, so interpolation might be used.
    return (0.0, 0.0, 0.0) # Placeholder
```

**Parameters:**
*   $n_k$: The finest resolution of the input Hermite data grid. Typical values are not specified, but the example uses a $512^3$ grid for comparison [S2].
*   Average number of samples per ray: ~2.86 for a dragon model, ~2.05 for a fandisk model [S2]. This is significantly lower than the 512 samples required for a $512^3$ distance field [S2].

**Edge Cases and Pitfalls:**
*   **Redundancy:** The representation is redundant and requires regularization [S1].
*   **Regularization:** Separate blocks of voxel terrain are best stored as triangular meshes and converted to Ray-Rep and back as needed [S1].
*   **Precision:** The source notes that the representation is precise, but does not specify numerical precision issues. [G] In practice, floating-point precision may affect intersection calculations, especially for thin features.

## Operating characteristics
*   **Time Complexity:** Not explicitly given. The generation is faster than modified ray tracing due to the use of orthographic projection and Z-buffer with Z-list [S2].
*   **Memory Complexity:** Redundant, requiring regularization [S1]. However, it reduces storage compared to scalar fields. For a $512^3$ distance field, it reduces the number of samples per ray from 512 to approximately 2 [S2].
*   **Measured Performance:** Average of ~2.86 samples per ray for a dragon model and ~2.05 for a fandisk model [S2].
*   **Strengths:** High speed for triangulation using dual methods, preserves sharp features, reduces storage compared to scalar fields [S1, S2].
*   **Limitations:** Redundant, requires regularization, higher storage than compressible formats [S1].
*   **When to use:** When fast rendering from polygonal representation and fast CSG operations are required, such as in virtual world sculpting [S2].
*   **When not to use:** When storage space is a critical constraint and compression is available for other formats [S1].

## Background
*   **Origin:** The concept was introduced in the paper "Detail sculpting using cubical marching squares" [S2]. The term "Z-list-buffer" was used in the same paper to describe the generation method.
*   **History:** The GPU implementation of Ray-Rep with normals is called Layered Depth-Normal Images (LDNI) [S1]. Before GPGPU, layered depth images (LDI) used texture arrays to store depth and normal [S1]. With CUDA, OpenCL, and compute shaders, it became possible to build lists of points for each image fragment on the GPU [S1].
*   **Usage:** Used in virtual world sculpting, which involves continuous subtraction operations on target objects [S2]. It is also used for voxel terrain storage in VR system design [S1].

## Sources
[S1] REPRESENTATION AND STORAGE OF VOXEL TERRAIN FOR DESIGNING VIRTUAL REALITY SYSTEMS, https://doi.org/10.26102/2310-6018/2019.24.1.019, fetched 2026-10-26
[S2] Detail sculpting using cubical marching squares, https://doi.org/10.1145/1152399.1152402, fetched 2026-11-06
