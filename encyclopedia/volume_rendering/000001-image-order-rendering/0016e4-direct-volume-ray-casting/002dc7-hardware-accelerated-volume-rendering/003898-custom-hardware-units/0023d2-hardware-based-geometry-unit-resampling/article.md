# Hardware-based Geometry Unit Resampling

Also called: Heidelberg Raytracer geometry operations, VIRIM Geometry Unit Resampling, VIRIM basic visualization algorithm

## Overview
This method implements the geometry-intensive portion of the VIRIM basic visualization algorithm using dedicated hardware to perform perspective calculations, gradient estimation, and volume data resampling [S1]. It solves the performance bottleneck of software-based volume rendering, where geometry operations consume approximately 80% of the computation time per projection [S1]. Unlike general-purpose CPU implementations, this approach generates 26 to 36 million transformed locations per second by utilizing specialized address generation and conflict-free memory access patterns [S1]. It differs from the VOGUE architecture by replicating the entire volume memory in each parallel geometry unit to avoid data dependency issues, rather than partitioning the dataset [S2].

## How it works
**Inputs and Outputs**
*   **Input:** Ray starting point, ray vector (to next resample location), original volume dataset (voxel values), density Lookup Table (LUT).
*   **Output:** Resampled density values, X and Y gradient components for the rotated dataset.

**Data Structures**
*   **Weight Memory:** Size $256\text{K} \times 16$ bits. Stores precalculated interpolation weights. Addressed by three 6-bit fractional parts (one per coordinate: $u, v, w$) of the resampling location.
*   **Gradient Buffer:** Limited to a maximum resampled dataset size of $512^3$ voxels [S1].
*   **Memory Controller:** Returns 0 for addresses outside the data cube boundaries [S1].
*   **Cache:** 1-entry cache for previously used data voxels [S1].

**Parameters**
*   **Dataset Size Limits:**
    *   With 4Mbit ORAMs: $256^3$ voxels (16-bit).
    *   With 16Mbit DRAMs: $512 \times 512 \times 256$ voxels (16-bit) [S1].
*   **Data Precision:** 16-bit accuracy for density and gradient values [S1].
*   **Fractional Resolution:** 6 bits per coordinate for interpolation weight addressing [S1].
*   **Memory Bandwidth:** Up to 640 MBytes/second (calculated as $40\text{MHz} \times 8 \text{ neighbors} \times 2 \text{ Bytes/voxel}$) [S1].

**Algorithm Steps**
1.  **Initialization:** For each scanline of light rays, store the ray starting point and the vector to the next resample location.
2.  **Address Generation:** Use hardware to generate the positions of the sampling points. Round the X, Y, and Z components of the sampling location to obtain integer addresses.
3.  **Neighbor Selection:** Identify the 8 neighbor voxels in the original dataset corresponding to the rounded addresses. Access these neighbors conflict-free using Fast Page Mode, alternate memory bank read-out, and the 1-entry cache [S1]. If an address is outside the data cube, the controller returns 0.
4.  **Density Mapping:** Map the raw voxel values to density values using the density Lookup Table (LUT). This allows for grey value segmentation without prior data segmentation [S1].
5.  **Interpolation (Backward Mapping):** Perform weighted interpolation among the 8 mapped density neighbors.
    *   Calculate the fractional parts ($u, v, w$) of the resampling location.
    *   Retrieve precalculated weights from the Weight Memory using the 6-bit fractional parts as indices.
    *   Compute the interpolated density value.
6.  **Gradient Estimation:** Estimate the X and Y components of the gradient using 20 Sobel filters applied to the interpolated data [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometry_unit_resample(ray_start, ray_vector, volume_data, density_lut, weight_memory):
    """
    Performs hardware-based resampling for a single ray scanline.
    """
    # 1. Initialize ray parameters
    current_pos = ray_start
    step_vector = ray_vector
    
    # 2. Loop through resample locations along the ray
    while current_pos is within volume bounds:
        # 3. Generate sampling point positions
        # Hardware rounds X, Y, Z to get integer addresses
        base_x, base_y, base_z = floor(current_pos)
        
        # 4. Access 8 neighbor voxels conflict-free
        # Note: In hardware, this is parallel. Here simulated sequentially.
        # Controller returns 0 for out-of-bounds addresses.
        neighbors = []
        for dx in [0, 1]:
            for dy in [0, 1]:
                for dz in [0, 1]:
                    nx, ny, nz = base_x + dx, base_y + dy, base_z + dz
                    # Check bounds and apply cache logic if needed
                    if 0 <= nx < vol_width and 0 <= ny < vol_height and 0 <= nz < vol_depth:
                        voxel_val = volume_data[nx, ny, nz]
                    else:
                        voxel_val = 0
                    neighbors.append((dx, dy, dz, voxel_val))
        
        # 5. Map voxel values to density values using LUT
        density_neighbors = []
        for dx, dy, dz, val in neighbors:
            density_val = density_lut[val]
            density_neighbors.append((dx, dy, dz, density_val))
            
        # 6. Interpolation using precalculated weights
        # Calculate fractional parts
        fx = current_pos.x - base_x
        fy = current_pos.y - base_y
        fz = current_pos.z - base_z
        
        # Quantize to 6-bit indices (0-63)
        idx_fx = int(fx * 64) & 63
        idx_fy = int(fy * 64) & 63
        idx_fz = int(fz * 64) & 63
        
        interpolated_density = 0.0
        for dx, dy, dz, density_val in density_neighbors:
            # Retrieve weight from memory based on fractional parts and neighbor offset
            # Weight memory is indexed by (idx_fx, idx_fy, idx_fz) and potentially neighbor offset
            # The source states weights are addressed by three 6-bit fractional parts.
            # Standard trilinear interpolation weights are products of (1-f) and f.
            # The hardware precalculates these.
            
            # Simplified weight retrieval logic based on source description:
            # The weight memory stores the specific interpolation coefficient for this neighbor
            # given the fractional coordinates.
            w = weight_memory.lookup(idx_fx, idx_fy, idx_fz, dx, dy, dz)
            interpolated_density += density_val * w
            
        # 7. Gradient Estimation using Sobel filters
        # The source specifies 20 Sobel filters are used to estimate X and Y components.
        # This is applied to the rotated/resampled dataset.
        # For a single point, we assume the gradient hardware processes the local neighborhood
        # of the interpolated density values.
        grad_x, grad_y = estimate_gradient_sobel(interpolated_density, neighbors, volume_data)
        
        yield interpolated_density, grad_x, grad_y
        
        # Advance ray
        current_pos += step_vector

def estimate_gradient_sobel(density_val, neighbors, volume_data):
    """
    Placeholder for the specific 20-Sobel filter implementation.
    The source states gradient hardware estimates X and Y components using 20 Sobel filters.
    Standard Sobel kernel:
    w_ij = 1/8 * [[1, 2, 1], [0, 0, 0], [-1, -2, -1]]
    """
    # Implementation details for the 20 filters are not fully specified in the source
    # beyond the kernel definition. [G]
    return 0.0, 0.0
```

**Formulas**
The Sobel filter weights for gradient estimation are defined as [S1]:
$$w_{i,j} = \frac{1}{8} \cdot \begin{bmatrix} 1 & 2 & 1 \\ 0 & 0 & 0 \\ -1 & -2 & -1 \end{bmatrix}$$

## Subtopics

### [VIRIM Architecture](3147-virim-architecture.md)
VIRIM is an object-order volume rendering engine developed at the University of Mannheim and commercially distributed by Volume Graphics GmbH. It implements projection plane raycasting, where rays belonging to a particular scanline reside on the same projection plane and are resampled in top-to-bottom order. The architecture is characterized by its flexibility, utilizing a farm of programmable Digital Signal Processors (DSPs) for the ray-casting portion of the algorithm, allowing users to implement different visualization models and balance image quality versus rendering speed. Unlike VOGUE, which partitions the dataset, VIRIM replicates the entire volume memory in each parallel geometry unit to circumvent data dependency problems.

## Operating characteristics
*   **Performance:** A single geometry unit can generate 26 to 36 million transformed locations per second using true 3D resampling [S1].
*   **Memory Bandwidth:** The dedicated memory system achieves data access rates of up to 640 MBytes/second, calculated as $40\text{MHz} \times 8 \text{ neighbors} \times 2 \text{ Bytes/voxel}$ [S1].
*   **Memory Constraints:**
    *   Original dataset size is limited to $256^3$ (16-bit) with 4Mbit ORAMs or $512 \times 512 \times 256$ with 16Mbit DRAMs [S1].
    *   The resampled dataset size is limited to $512^3$ due to gradient buffer size constraints [S1].
*   **Precision:** All calculations (density and gradient) use 16-bit data resolution and accuracy [S1].
*   **Strengths:** High throughput for geometry operations; conflict-free memory access; flexibility in shading models via DSPs in the ray-casting unit.
*   **Limitations:** Discreteness of precalculated interpolation weights can cause artifacts, mitigated by using 6-bit resolution for fractional parts [S1]. Hardware-specific memory limits restrict maximum volume size.

## Background
The method originates from the VIRIM (Volume Rendering Image Machine) project, developed at the University of Mannheim and later commercially distributed by Volume Graphics GmbH [S1, S2]. It was designed to address the high computational cost of geometry operations in volume rendering, which traditionally consumed the majority of processing time [S1]. The architecture was presented in the mid-1990s as one of the three major volume rendering architectures of that era, alongside VOGUE and others [S1].

## Sources
[S1] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
