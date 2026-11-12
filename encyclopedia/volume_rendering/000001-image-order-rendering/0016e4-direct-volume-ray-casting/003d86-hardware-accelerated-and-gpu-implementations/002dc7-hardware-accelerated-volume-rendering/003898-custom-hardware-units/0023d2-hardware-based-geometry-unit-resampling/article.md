# Hardware-based Geometry Unit Resampling

Also called: Heidelberg Raytracer geometry operations, VIRIM Geometry Unit Resampling, VIRIM basic visualization algorithm

## Overview
This method implements a dedicated hardware geometry unit that performs perspective calculations, gradient estimation, and volume data resampling to generate ray starting points and sample values for direct volume ray casting. It solves the problem of high computational cost in geometry operations, which typically consume 80% of the processing time in software-based volume rendering [S1]. Unlike general-purpose GPU implementations, this approach uses specialized ASIC logic for address generation and interpolation, enabling data access rates up to 640 MB/s [S1]. It is distinguished by its use of precalculated interpolation weights stored in dedicated memory and a dedicated gradient estimation hardware block using Sobel filters, operating within the VIRIM architecture [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Original volume dataset (stored in ORAMs or DRAMs), ray starting points, and ray direction vectors for each scanline.
*   **Output:** Resampled density values and estimated X/Y gradient components for the projection plane.

**Data Structures:**
*   **Weight Memory:** Size $256\text{K} \times 16$ bits. Stores precalculated interpolation weights. Addressed by three 6-bit fractional parts (one per coordinate: $u, v, w$).
*   **Gradient Buffer:** Limited to a resampled dataset size of $512^3$ voxels [S1].
*   **Density LUT:** Maps raw voxel values to density values. Accessible by the host system for real-time reloading [S1].
*   **Cache:** 1-entry cache for previously used data voxels [S1].

**Parameters:**
*   **Fractional Resolution:** 6 bits per coordinate for interpolation weight addressing [S1].
*   **Data Precision:** 16 bits for density and gradient values [S1].
*   **Dataset Limits:**
    *   Max original size: $256^3$ (16-bit voxels) with 4 Mbit ORAMs [S1].
    *   Max original size: $512 \times 512 \times 256$ with 16 Mbit DRAMs [S1].
*   **Memory Bandwidth:** Up to 640 MB/s (calculated as $40\text{MHz} \times 8 \text{ neighbors} \times 2 \text{ Bytes/voxel}$) [S1].

**Steps:**
1.  **Ray Initialization:** For each scanline, store the ray starting point and the vector to the next resample location [S1].
2.  **Address Generation:** Use hardware to generate sampling point positions. Round the X, Y, and Z components of the sampling location to obtain integer addresses [S1].
3.  **Neighbor Access:** Access the 8 neighbor voxels in the original dataset corresponding to the rounded addresses.
    *   *Boundary Handling:* A controller returns 0 for addresses outside the data cube [S1].
    *   *Optimization:* Use Fast Page Mode, alternate memory bank read-out, and the 1-entry cache to ensure conflict-free access [S1].
4.  **Density Mapping:** Map the raw voxel values of the 8 neighbors to density values using the Density Lookup Table (LUT) [S1].
5.  **Interpolation:** Perform backward mapping rotation using weighted interpolation among the 8 voxel neighbors.
    *   The weights are retrieved from the Weight Memory using the 6-bit fractional parts of the sampling coordinates [S1].
    *   Different filters (e.g., tri-linear or sinc approximation) can be used [S1].
6.  **Gradient Estimation:** Estimate the X and Y components of the gradient using 20 Sobel filters applied to the interpolated density values [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
NUM_NEIGHBORS = 8
FRACTION_BITS = 6
MAX_WEIGHTS = 256 * 1024  # 256K

# Hardware Memory Accessors (Abstracted)
def access_weight_memory(u_frac, v_frac, w_frac):
    """
    Retrieves interpolation weight from precalculated memory.
    u_frac, v_frac, w_frac are 6-bit integers (0-63).
    """
    # In hardware, this is a direct lookup.
    # The index is derived from the combination of fractional parts.
    # Returns a 16-bit weight value.
    pass

def access_volume_memory(x, y, z):
    """
    Accesses the original volume dataset.
    Returns 0 if (x,y,z) is outside the data cube.
    Uses cache and bank interleaving for speed.
    """
    if not is_inside_bounds(x, y, z):
        return 0
    # Check 1-entry cache
    if is_in_cache(x, y, z):
        return get_from_cache(x, y, z)
    # Read from DRAM/ORAM
    val = read_dram(x, y, z)
    update_cache(x, y, z, val)
    return val

def apply_density_lut(raw_value):
    """Maps raw voxel value to density using LUT."""
    return density_lut[raw_value]

def compute_sobel_gradient(density_grid_3x3x3, axis):
    """
    Computes gradient component using Sobel filters.
    Uses 20 Sobel filters total for X and Y components.
    """
    # Implementation details of the 20 filters are hardware-specific.
    # This function abstracts the hardware block.
    pass

def resample_scanline(ray_origin, ray_direction, step_size):
    """
    Processes a single scanline.
    """
    # 1. Initialize ray parameters
    current_pos = ray_origin
    next_pos = current_pos + ray_direction * step_size
    
    while current_pos is within view volume:
        # 2. Generate sampling point
        # Round to get integer addresses for neighbors
        x_int = floor(current_pos.x)
        y_int = floor(current_pos.y)
        z_int = floor(current_pos.z)
        
        # 3. Get fractional parts for weight lookup
        u_frac = int((current_pos.x - x_int) * (2**FRACTION_BITS))
        v_frac = int((current_pos.y - y_int) * (2**FRACTION_BITS))
        w_frac = int((current_pos.z - z_int) * (2**FRACTION_BITS))
        
        # 4. Access 8 neighbors and map to density
        densities = []
        for dx in [0, 1]:
            for dy in [0, 1]:
                for dz in [0, 1]:
                    nx = x_int + dx
                    ny = y_int + dy
                    nz = z_int + dz
                    raw_val = access_volume_memory(nx, ny, nz)
                    densities.append(apply_density_lut(raw_val))
        
        # 5. Interpolate
        # Retrieve weights for the 8 corners
        weights = []
        for dx in [0, 1]:
            for dy in [0, 1]:
                for dz in [0, 1]:
                    # Weight lookup depends on specific bit packing of u,v,w
                    # Assuming standard tri-linear weight generation logic
                    # stored in hardware memory
                    w = access_weight_memory(
                        u_frac if dx == 0 else (2**FRACTION_BITS - 1 - u_frac),
                        v_frac if dy == 0 else (2**FRACTION_BITS - 1 - v_frac),
                        w_frac if dz == 0 else (2**FRACTION_BITS - 1 - w_frac)
                    )
                    weights.append(w)
        
        # Compute weighted sum
        interpolated_density = 0.0
        for i in range(8):
            interpolated_density += densities[i] * weights[i]
        
        # 6. Gradient Estimation
        # The hardware block estimates X and Y gradients using 20 Sobel filters
        # on the interpolated data.
        # Note: The Sobel weights are defined as:
        # w_ij = 1/8 * [[1, 2, 1], [0, 0, 0], [-1, -2, -1]]
        # This is applied in the hardware gradient unit.
        gradient_x, gradient_y = compute_sobel_gradient(interpolated_density, current_pos)
        
        # Output density and gradients to ray-casting unit
        yield interpolated_density, gradient_x, gradient_y
        
        # Advance ray
        current_pos = next_pos
        next_pos += ray_direction * step_size
```

**Formulas:**
The Sobel filter weights for gradient estimation are defined as [S1]:
$$w_{i,j} = \frac{1}{8} \cdot \begin{bmatrix} 1 & 2 & 1 \\ 0 & 0 & 0 \\ -1 & -2 & -1 \end{bmatrix}$$

**Edge Cases and Pitfalls:**
*   **Discreteness Artifacts:** The use of 6-bit resolution for fractional parts in weight addressing can cause artifacts due to the discreteness of the precalculated weights. This resolution was chosen specifically to make these artifacts invisible to the viewer [S1].
*   **Boundary Conditions:** Addresses outside the data cube must be handled by returning 0 to prevent memory errors [S1].

## Subtopics

### [VIRIM Architecture](3147-virim-architecture.md)
VIRIM (Virtual Reality in Medicine) is an object-order volume rendering engine developed at the University of Mannheim and commercially distributed by Volume Graphics GmbH. It solves the problem of high-performance, flexible volume visualization by separating the pipeline into a dedicated geometry unit for resampling and a programmable ray-casting unit for image generation. Unlike image-order architectures that process pixels directly, VIRIM uses projection plane raycasting, where rays belonging to a scanline reside on the same projection plane, allowing for efficient hardware-based resampling and view-dependent gradient estimation. It is distinguished by its use of programmable DSPs for the shading stage, enabling high-quality illumination models like the Heidelberg algorithm, at the cost of requiring replicated volume memory in parallel configurations.

## Operating characteristics
*   **Performance:** A single geometry unit can generate 26 to 36 million transformed locations per second using true 3D resampling [S1].
*   **Memory Bandwidth:** Achieves up to 640 MB/s through conflict-free access of 8 neighbors, Fast Page Mode, and alternate bank read-out [S1].
*   **Complexity:** The hardware offloads the 80% of computation time typically spent on geometry operations in software [S1].
*   **Limitations:** The maximal resampled dataset size is limited to $512^3$ by the gradient buffer size [S1]. Original dataset size is constrained by the available memory technology (4 Mbit ORAMs vs 16 Mbit DRAMs) [S1].
*   **Precision:** All calculations (density and gradient) are performed with 16-bit precision [S1].

## Background
This method is part of the VIRIM (Virtual Reality in Medicine) architecture, developed at the University of Mannheim [S1, S2]. It was designed to address the computational bottlenecks of volume rendering by dedicating hardware to geometry operations, which are the most expensive part of the rendering pipeline in software implementations [S1]. The approach is closely associated with the Heidelberg Raytracer and the basic visualization algorithm of VIRIM [S1].

## Sources
[S1] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
