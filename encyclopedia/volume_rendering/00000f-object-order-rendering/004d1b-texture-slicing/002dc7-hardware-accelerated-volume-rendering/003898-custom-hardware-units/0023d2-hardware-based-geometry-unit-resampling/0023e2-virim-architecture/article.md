# VIRIM Architecture

Also called: VIRIM, Virtual Reality in Medicine

## Overview
VIRIM (Virtual Reality in Medicine) is an object-order volume rendering engine developed at the University of Mannheim and commercially distributed by Volume Graphics GmbH [S1][S2, unofficial]. It solves the problem of high-performance, flexible volume visualization by separating the pipeline into a dedicated geometry unit for resampling and a programmable ray-casting unit for image generation [S1, S3]. Unlike image-order architectures that process pixels directly, VIRIM uses projection plane raycasting, where rays belonging to a scanline reside on the same projection plane, allowing for efficient hardware-based resampling and view-dependent gradient estimation [S1][S2, unofficial]. It is distinguished by its use of programmable DSPs for the shading stage, enabling high-quality illumination models like the Heidelberg algorithm, at the cost of requiring replicated volume memory in parallel configurations [S4, unofficial][S5].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume dataset (up to $256^3$ voxels, 16-bit), viewing parameters (orientation, projection type), and user-defined classification/lighting parameters [S1][S2, S4, unofficial].
*   **Output:** Final 2D image frame [S1].

**Data Structures:**
1.  **Volume Memory:** 8-way interleaved memory system [S1][S2, unofficial].
2.  **Density Lookup Tables (LUTs):** 8 LUTs (one per memory bank) mapping raw voxel values to classified density values [S2, unofficial].
3.  **Interpolation Weight Memory:** Stores arbitrary interpolation weights (e.g., Gaussian) for the 8-voxel neighborhood [S2, unofficial].
4.  **Geometry Bus:** Transmits sample data from geometry unit to ray-casting unit. Width: 48 bits; Clock: 40 MHz; Peak Rate: 240 MB/s [S1][S2, unofficial].
5.  **FIFOs:** First-In-First-Out memories on DSP boards to decouple the high-speed geometry unit from the slower DSPs [S2, unofficial].
6.  **DSP Memory:** Local memory on DSP boards storing scan lines of the final image [S1].

**Algorithm Steps:**

1.  **Rotation and Resampling (Geometry Unit):**
    The Rotator Board aligns the volume dataset with the viewing position using backward mapping [S2, unofficial].
    *   The geometry processor generates addresses for the 8 memory banks using a rotation matrix [S2, unofficial].
    *   Voxel values are retrieved from the 8-way interleaved memory [S1].
    *   Values are mapped to density values using the Density LUTs [S1][S2, unofficial].
    *   Interpolation is performed on the classified density values using a weighted interpolation mask over the 8-voxel neighborhood [S2, unofficial]. The weights are stored in the Interpolation Weight Memory [S2, unofficial].
    *   *Note:* While tri-linear interpolation is the default, arbitrary weights (e.g., Gaussian) can be used [S2, unofficial].

2.  **Gradient Estimation (Geometry Unit):**
    *   A modified 2D Sobel filter estimates the X- and Y-components of the gradient vector in the re-sampled coordinate system [S2, unofficial].
    *   The gradient is 2D and view-dependent [S2, unofficial].

3.  **Data Transfer:**
    *   Sample density, X-gradient, and Y-gradient values are transmitted from the geometry unit to the ray-casting unit via the Geometry Bus [S1][S2, unofficial].
    *   Data is stored in FIFOs on the DSP boards to handle speed differences between the geometry unit and DSPs [S2, unofficial].

4.  **Ray Casting and Shading (Ray-Casting Unit):**
    *   The ray-casting unit consists of 16 programmable Digital Signal Processors (DSPs) [S1].
    *   DSPs cast rays through the resampled dataset according to user-programmable lighting and viewing parameters [S3].
    *   The Heidelberg illumination model is typically implemented [S1][S2, unofficial].
        *   The dataset is conceptually rotated so the viewer looks along a major axis [S2, unofficial].
        *   Two light sources are used: one along the viewer's direction, and one 45 degrees from the viewer's direction [S2, unofficial].
        *   Light intensity is calculated slice-by-slice [S2, unofficial].
        *   The final illumination value per sample is the summation of all light intensity emitted in the viewer's direction [S2, unofficial].
        *   This model accounts for reflection, absorption, emission, and shadows [S2, unofficial].
    *   A local master processor on the ray-casting board collects all scan lines from DSP memories and transfers the result to the host [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def virim_render(volume_data, volume_dims, view_params, classification_luts, lighting_params):
    """
    volume_data: 3D list/array of raw voxel values, max 256^3, 16-bit
    volume_dims: (D, H, W) dimensions of the volume
    view_params: orientation matrix for backward mapping
    classification_luts: list of 8 LUTs (one per memory bank)
    lighting_params: light source directions, shading model parameters
    """
    height, width = view_params.image_height, view_params.image_width
    final_image = [[0.0] * width for _ in range(height)]
    
    # 1. Geometry Unit: Resampling and Gradient Estimation
    # The geometry unit processes the projection plane (image grid)
    for y in range(height):
        for x in range(width):
            # Backward mapping to find source voxel coordinates in original volume
            src_x, src_y, src_z = backward_map(x, y, view_params.orientation)
            
            # Identify the 8 neighboring voxels in the 8-way interleaved memory
            # Clamp coordinates to volume boundaries to prevent IndexError [G]
            neighbors = []
            for dx in (0, 1):
                for dy in (0, 1):
                    for dz in (0, 1):
                        nx = min(max(int(src_x) + dx, 0), volume_dims[2] - 1)
                        ny = min(max(int(src_y) + dy, 0), volume_dims[1] - 1)
                        nz = min(max(int(src_z) + dz, 0), volume_dims[0] - 1)
                        neighbors.append((nx, ny, nz))
            
            # Retrieve raw voxel values
            raw_values = [volume_data[nz][ny][nx] for nx, ny, nz in neighbors]
            
            # Map to density values using Density LUTs
            density_values = [classification_luts[i].lookup(raw_values[i]) for i in range(8)]
            
            # Interpolate using weights from Interpolation Weight Memory
            # Weights can be arbitrary (e.g., Gaussian) or tri-linear
            weights = get_interpolation_weights(src_x, src_y, src_z)
            interpolated_density = sum(density_values[i] * weights[i] for i in range(8))
            
            # Estimate 2D gradient using modified Sobel filter in re-sampled coordinates
            # Gradient is view-dependent [S2, unofficial]
            gradient_x = sobel_x_filter(density_values, neighbors, view_params)
            gradient_y = sobel_y_filter(density_values, neighbors, view_params)
            
            # Transfer to Ray-Casting Unit via Geometry Bus
            # Data: density, gradient_x, gradient_y
            geometry_bus_queue.append((interpolated_density, gradient_x, gradient_y))
            
    # 2. Ray-Casting Unit: DSP Processing
    # 16 DSPs process the stream of samples
    for density, grad_x, grad_y in geometry_bus_queue:
        # Calculate illumination using Heidelberg model
        # Light 1: Viewer direction (0 degrees)
        # Light 2: 45 degrees from viewer direction
        light_dir_1 = view_params.view_direction
        light_dir_2 = rotate_vector(view_params.view_direction, math.radians(45))
        
        # Light intensity calculated slice-by-slice [S2, unofficial]
        # Summation of light intensity emitted in viewer's direction [S2, unofficial]
        # The Heidelberg model involves complex ray-tracing within the slice
        # accounting for reflection, absorption, emission, and shadows [S2, unofficial].
        # Sources do not specify the exact slice-by-slice ray-tracing formula [Comment].
        illumination = heidelberg_slice_integration(density, grad_x, grad_y, light_dir_1, light_dir_2, lighting_params)
        
        # Store in DSP local memory
        dsp_memory.write(illumination)
        
    # 3. Final Assembly
    # Local master processor collects scan lines from DSP memories
    final_image = master_processor.collect_scan_lines(dsp_memory)
    
    return final_image

def backward_map(screen_x, screen_y, orientation_matrix):
    """Projects screen coordinates back into volume space using the view orientation [S2, unofficial]."""
    # Standard inverse projection matrix multiplication [G]
    return orientation_matrix.inverse() * [screen_x, screen_y, 0.0, 1.0]

def get_interpolation_weights(src_x, src_y, src_z):
    """Returns 8 interpolation weights for the voxel neighborhood [S2, unofficial]."""
    # Default to tri-linear weights if not overridden by Interpolation Weight Memory [S2, unofficial]
    fx, fy, fz = src_x % 1, src_y % 1, src_z % 1
    weights = []
    for dx in (0, 1):
        for dy in (0, 1):
            for dz in (0, 1):
                w = (1 if dx == 0 else fx) * (1 if dy == 0 else fy) * (1 if dz == 0 else fz)
                weights.append(w)
    return weights

def sobel_x_filter(density_values, neighbors, view_params):
    """Computes X-component of the 2D gradient using a modified Sobel operator [S2, unofficial]."""
    # Standard Sobel kernel application on the 8-neighbor density values [G]
    # Adjusted for view-dependent coordinate system [S2, unofficial]
    return sum(density_values[i] * sobel_kernel_x[i] for i in range(8))

def sobel_y_filter(density_values, neighbors, view_params):
    """Computes Y-component of the 2D gradient using a modified Sobel operator [S2, unofficial]."""
    return sum(density_values[i] * sobel_kernel_y[i] for i in range(8))

def heidelberg_slice_integration(density, grad_x, grad_y, light_dir_1, light_dir_2, params):
    """
    Placeholder for the Heidelberg illumination model's slice-by-slice ray-tracing.
    Sources describe the model conceptually (two lights, summation, shadows) but 
    do not provide the exact mathematical formulation for the slice integration [Comment].
    """
    # Simplified contribution for implementation completeness [G]
    normal = normalize([grad_x, grad_y, 0.0])
    diff_1 = max(0.0, dot(normal, light_dir_1))
    diff_2 = max(0.0, dot(normal, light_dir_2))
    return density * (diff_1 + diff_2) * params.ambient + params.emission

def normalize(v):
    length = math.sqrt(sum(c*c for c in v))
    return [c/length for c in v]

def dot(v1, v2):
    return sum(a*b for a, b in zip(v1, v2))

def rotate_vector(v, angle):
    # Placeholder for 3D rotation logic [G]
    return v
```

**Parameters:**
*   **Dataset Size:** Maximum $256^3$ voxels (16-bit) [S1][S4, unofficial].
*   **Geometry Bus:** 48 bits, 40 MHz, 240 MB/s peak transfer rate [S1][S2, unofficial].
*   **Memory Read-out Rate:** Maximum 640 MB/s [S1].
*   **Sample Generation Rate:** Geometry unit generates max 36 million tri-linear samples per second [S1].
*   **Light Source Angle:** Second light source is 45 degrees from the viewer's direction [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Polygon Intermixing:** VIRIM does not allow arbitrary intermixing of volumes and polygons [S1][S4, unofficial].
*   **Parallel Scaling:** To circumvent data dependency problems in parallel configurations, the entire volume memory must be replicated in each parallel geometry unit [S4, unofficial][S5]. This contrasts with architectures like VOGUE that partition the dataset [S4, unofficial][S5].
*   **Gradient Limitation:** The gradient estimation is only 2D and view-dependent, which may limit shading accuracy compared to full 3D gradient methods [S2, unofficial].
*   **Boundary Handling:** Backward mapping can produce coordinates outside the volume bounds; hardware typically clamps or reflects coordinates to prevent out-of-bounds memory access [G].

## Operating characteristics
*   **Time Complexity:** Not explicitly defined in asymptotic terms, but performance is measured in frames per second (fps) for specific dataset sizes [S1][S2, unofficial].
*   **Memory Complexity:** Requires 8-way interleaved memory. In parallel setups, memory usage scales linearly with the number of modules due to full replication of the volume dataset [S4, unofficial][S5].
*   **Measured Performance:**
    *   Single module (4 boards): 10 Hz for $256^3$ datasets [S3].
    *   Four modules (16 boards): 10 Hz for $256^3$ datasets [S3].
    *   Eight modules (32 boards): 10 Hz for $256^3$ datasets [S3].
    *   Existing hardware implementation (16 processing boards): 10 fps for $256 \times 256 \times 128$ datasets [S2, S4, unofficial].
    *   General performance: Up to 2.5 fps for $256 \times 256 \times 128$ datasets without parallelization [S1].
*   **Strengths:**
    *   High flexibility due to programmable DSPs for shading and classification [S1][S4, unofficial].
    *   High-quality illumination models (Heidelberg algorithm) with shadows [S1][S2, unofficial].
    *   Dedicated hardware for resampling and gradient estimation ensures speed [S2, unofficial].
    *   Arbitrary interpolation weights supported [S2, unofficial].
*   **Limitations:**
    *   No support for arbitrary volume-polygon intermixing [S1][S4, unofficial].
    *   2D view-dependent gradient estimation limits shading fidelity [S2, unofficial].
    *   Parallel scaling requires full memory replication, increasing hardware cost [S4, unofficial][S5].
*   **When to use:** When high-quality, flexible volume rendering with custom illumination models is required, and polygon integration is not a primary need [S1][S4, unofficial].
*   **When not to use:** When arbitrary volume-polygon intermixing is required, or when memory efficiency in parallel scaling is critical [S1][S4, unofficial].

## Background
VIRIM (Virtual Reality in Medicine) was developed at the University of Mannheim and is described in papers from the mid-1990s [S1][S2, S4, unofficial]. It was commercially distributed by Volume Graphics GmbH, Heidelberg, Germany [S1]. The architecture was designed to provide a flexible and programmable solution for volume visualization, contrasting with more rigid, fixed-function accelerators of the time [S3][S4, unofficial]. It was one of the early examples of object-order ray casting implemented in dedicated hardware [S1][S2, unofficial].

## Sources
[S1] Architectures for real-time volume rendering, https://doi.org/10.1016/s0167-739x(98)00051-x, fetched 2026-11-01
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S3] Ray Casting Architectures for Volume Visualization, https://vcg.seas.harvard.edu/publications/19990101-ray-casting-architetures-for-volume-visualization/paper, fetched 2026-10-27
[S4] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S5] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
