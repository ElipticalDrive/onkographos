# Precomputed Gradient Volume

Also called: Precomputed Normalized Gradient, VoxelVertex Gradient Storage

## Overview
Precomputed Gradient Volume is a volume rendering acceleration technique that trades memory for rendering speed by calculating and storing the 3D gradient vector for every voxel in a scalar volume during a preprocessing step [S2, unofficial]. This allows the rendering engine to sample surface normals directly from memory during raycasting, avoiding the computational cost of finite difference calculations per sample [S2, unofficial]. It distinguishes itself from on-the-fly gradient estimation by requiring significant additional storage (e.g., >200 MB for a 256³ dataset) but enabling faster shading and lighting calculations [S2, unofficial].

## How it works
The method involves two phases: data structure definition and gradient precomputation.

### Inputs and Outputs
*   **Input:** A 3D scalar volume grid of size $L \times L \times L$ containing density or intensity values.
*   **Output:** A flat array of structures, where each element contains the original scalar value and the precomputed normalized gradient vector.

### Data Structures
The core data structure is a union or struct that packs the scalar value and gradient components.

**Layout (Standard Implementation):**
A `VoxelVertex` structure stores the data. It is implemented as a union to allow access as either a 4-component vector or separate components [S1, unofficial].
*   `xyzw`: A `vec4` where `x, y, z` store the gradient components and `w` stores the scalar volume value.
*   `gradient`: A `vec3` storing the normalized gradient vector.
*   **Indexing:** The data is stored in a flat `std::array<VoxelVertex, L^3>`. The index for a voxel at coordinates $(x, y, z)$ is calculated as:
    $$ index = z \cdot L^2 + y \cdot L + x $$
    where $L$ is the cubic volume dimension [S1, unofficial].

**Layout (High-Quality/Sobel Variant):**
For high-resolution anatomical data, a different packing scheme is used to optimize texture storage and filtering [S3]:
*   **Scalar Data:** Stored as 16-bit unsigned integers.
*   **Gradient Data:** Stored as 32-bit values, composed of 8 bits for the X component, 8 bits for the Y component, 8 bits for the Z component, and 8 bits for the norm (magnitude) [S3].

### Precomputation Steps
The gradient is computed for each voxel $(x, y, z)$ using finite differences. Boundary conditions are handled by zero-padding [S1, unofficial].

1.  **Retrieve Neighbors:** For each voxel, retrieve the scalar values of its six immediate neighbors. Use a safe access function `m_safe_get(x, y, z)` which returns $0.0$ if the coordinates are out of bounds ($<0$ or $\ge L$) [S1, unofficial].
    *   $v_{x-} = \text{m\_safe\_get}(x-1, y, z)$
    *   $v_{x+} = \text{m\_safe\_get}(x+1, y, z)$
    *   $v_{y-} = \text{m\_safe\_get}(x, y-1, z)$
    *   $v_{y+} = \text{m\_safe\_get}(x, y+1, z)$
    *   $v_{z-} = \text{m\_safe\_get}(x, y, z-1)$
    *   $v_{z+} = \text{m\_safe\_get}(x, y, z+1)$

2.  **Compute Gradient Vector:** Calculate the central difference approximation for the gradient $\mathbf{g}$:
    $$ \mathbf{g} = (v_{x-} - v_{x+}, \quad v_{y-} - v_{y+}, \quad v_{z-} - v_{z+}) $$
    *Note: The source code uses a backward difference convention ($v_{before} - v_{after}$) [S1, unofficial].*

3.  **Normalize:** Normalize the gradient vector to unit length:
    $$ \mathbf{n} = \frac{\mathbf{g}}{\|\mathbf{g}\|} $$
    If $\|\mathbf{g}\| = 0$, the normal is undefined (typically treated as zero vector or identity depending on implementation, though not explicitly specified in sources [G]).

4.  **Store:** Store the result in the data structure.
    *   In the standard union layout: `data[index].xyzw = vec4(n.x, n.y, n.z, scalar_value)` [S1, unofficial].
    *   In the Sobel variant: The gradient is precomputed using a 5x5x5 Sobel filter instead of simple central differences to improve quality [S3]. The components are quantized to 8-bit integers [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def precompute_gradients(volume: List[float], length: int) -> List[VoxelVertex]:
    """
    Precompute normalized gradients for a cubic volume.
    volume: 1D list of floats representing the LxLxL grid.
    length: Dimension L of the cubic volume.
    Returns: List of VoxelVertex objects.
    """
    num_voxels = length * length * length
    data = [None] * num_voxels

    # Helper to safely access volume with zero-padding
    def safe_get(x, y, z):
        if 0 <= x < length and 0 <= y < length and 0 <= z < length:
            return volume[z * length * length + y * length + x]
        return 0.0

    for z in range(length):
        for y in range(length):
            for x in range(length):
                # Retrieve neighbors
                v_x_minus = safe_get(x - 1, y, z)
                v_x_plus = safe_get(x + 1, y, z)
                v_y_minus = safe_get(x, y - 1, z)
                v_y_plus = safe_get(x, y + 1, z)
                v_z_minus = safe_get(x, y, z - 1)
                v_z_plus = safe_get(x, y, z + 1)

                # Compute gradient (central difference)
                # Source uses (before - after) convention
                gx = v_x_minus - v_x_plus
                gy = v_y_minus - v_y_plus
                gz = v_z_minus - v_z_plus

                # Normalize
                norm = sqrt(gx*gx + gy*gy + gz*gz)
                if norm > 0:
                    nx = gx / norm
                    ny = gy / norm
                    nz = gz / norm
                else:
                    nx, ny, nz = 0.0, 0.0, 0.0

                # Get scalar value
                scalar = safe_get(x, y, z)

                # Store in VoxelVertex union structure
                # xyzw stores gradient in x,y,z and scalar in w
                data[z * length * length + y * length + x] = VoxelVertex(nx, ny, nz, scalar)

    return data
```

### Parameters
*   **Length ($L$):** The dimension of the cubic volume. Typical value in code examples is 60 (`CUBE_SIZE`) [S1, unofficial].
*   **Memory Overhead:** For a $256 \times 256 \times 256$ dataset, storing both the scalar data and the precomputed gradients requires more than 200 megabytes [S2, unofficial].

### Edge Cases and Pitfalls
*   **Boundary Conditions:** Out-of-bounds access during gradient computation is handled by returning $0.0$, effectively zero-padding the volume boundaries [S1, unofficial]. This can create artificial gradients at the edges of the volume if the volume is not surrounded by a border of zeros.
*   **Memory Constraints:** The memory requirement for high-resolution volumes (e.g., 256³) can exceed available RAM on standard workstations, preventing sequential rendering [S2, unofficial].
*   **Segmented Data:** For segmented datasets (discrete labels), standard averaging during LOD precomputation is replaced by selecting the most popular label [S3]. At runtime, trilinear filtering is substituted with nearest-neighbor filtering to preserve label integrity [S3].

## Subtopics

### Data Representation and Precomputation
A variant optimized for high-quality anatomical display. It uses 16-bit scalars and 32-bit gradients (8 bits per component + 8 bits for norm) [S3]. Precomputation employs a high-quality 5x5x5 Sobel filter instead of simple central differences [S3]. For segmented data, LOD precomputation selects the most popular label rather than averaging, and runtime rendering uses nearest-neighbor filtering instead of trilinear filtering [S3]. Jittering of ray start positions is employed to reduce aliasing artifacts [S3].

### Gradient Precomputation with Sobel Filtering
This specific approach defines the precomputation step using a 5x5x5 Sobel filter kernel to estimate gradients. The resulting gradient data is stored in a 32-bit format: 8 bits for X, 8 bits for Y, 8 bits for Z, and 8 bits for the norm [S3]. The rendering pipeline operates at fully floating-point precision despite the quantized storage of gradients [S3].

## Operating characteristics
*   **Time Complexity:** Precomputation is $O(N)$ where $N$ is the number of voxels, with a constant factor depending on the stencil size (3x3x3 for central difference, 5x5x5 for Sobel). Rendering time is reduced because gradient lookups are memory accesses rather than arithmetic operations.
*   **Memory Complexity:** $O(N)$ additional memory. For a $256^3$ volume, this exceeds 200 MB [S2, unofficial].
*   **Performance:** Enables faster shading by avoiding per-sample gradient calculations [S2, unofficial]. However, the high memory footprint can prevent rendering on systems with limited RAM [S2, unofficial].
*   **Strengths:** Fast rendering, high-quality normals (with Sobel variant), supports occlusion queries and out-of-core structures [S3].
*   **Limitations:** High memory usage, boundary artifacts due to zero-padding, not suitable for dynamic volumes without recomputation.

## Background
The method was developed for parallel volume rendering on massively parallel supercomputers like the CM-5, which features SIMD and MIMD programming models with vector units [S2, unofficial]. The approach trades memory for time to optimize rendering speed on these architectures [S2, unofficial]. It has also been implemented in systems for high-quality anatomical data analysis, utilizing hardware/software frameworks with OpenGL and Cg shaders, and out-of-core storage via Berkeley DB [S3].

## Sources
[S1] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19940009248: A data distributed parallel algorithm for ray-traced volume rendering, https://archive.org/details/NASA_NTRS_Archive_19940009248, fetched 2026-10-26 (unofficial)
[S3] Towards Advanced Volumetric Display of the Human Musculoskeletal System, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/021-025, fetched 2026-10-26
