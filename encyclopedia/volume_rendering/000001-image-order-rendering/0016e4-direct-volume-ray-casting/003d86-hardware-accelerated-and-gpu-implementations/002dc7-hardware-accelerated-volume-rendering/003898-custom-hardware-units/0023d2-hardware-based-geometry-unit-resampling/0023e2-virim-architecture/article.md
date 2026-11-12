# VIRIM Architecture

Also called: VIRIM, Virtual Reality in Medicine

## Overview
VIRIM (Virtual Reality in Medicine) is an object-order volume rendering engine developed at the University of Mannheim and commercially distributed by Volume Graphics GmbH [S1, S2]. It solves the problem of high-performance, flexible volume visualization by separating the pipeline into a dedicated geometry unit for resampling and a programmable ray-casting unit for image generation [S1][S3, unofficial]. Unlike image-order architectures that process pixels directly, VIRIM uses projection plane raycasting, where rays belonging to a scanline reside on the same projection plane, allowing for efficient hardware-based resampling and view-dependent gradient estimation [S1, S2]. It is distinguished by its use of programmable DSPs for the shading stage, enabling high-quality illumination models like the Heidelberg algorithm, at the cost of requiring replicated volume memory in parallel configurations [S4].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume dataset (up to $256^3$ voxels, 16-bit), viewing parameters (orientation, projection type), and user-defined classification/lighting parameters [S1, S2, S4].
*   **Output:** Final 2D image frame [S1].

**Data Structures:**
1.  **Volume Memory:** 8-way interleaved memory system [S1, S2].
2.  **Density Lookup Tables (LUTs):** 8 LUTs (one per memory bank) mapping raw voxel values to classified density values [S2].
3.  **Interpolation Weight Memory:** Stores arbitrary interpolation weights (e.g., Gaussian) for the 8-voxel neighborhood [S2].
4.  **Geometry Bus:** Transmits sample data from geometry unit to ray-casting unit. Width: 48 bits; Clock: 40 MHz; Peak Rate: 240 MB/s [S1, S2].
5.  **FIFOs:** First-In-First-Out memories on DSP boards to decouple the high-speed geometry unit from the slower DSPs [S2].
6.  **DSP Memory:** Local memory on DSP boards storing scan lines of the final image [S1].

**Algorithm Steps:**

1.  **Rotation and Resampling (Geometry Unit):**
    The Rotator Board aligns the volume dataset with the viewing position using backward mapping [S2].
    *   The geometry processor generates addresses for the 8 memory banks using a rotation matrix [S2].
    *   Voxel values are retrieved from the 8-way interleaved memory [S1].
    *   Values are mapped to density values using the Density LUTs [S1, S2].
    *   Interpolation is performed on the classified density values using a weighted interpolation mask over the 8-voxel neighborhood [S2]. The weights are stored in the Interpolation Weight Memory [S2].
    *   *Note:* While tri-linear interpolation is the default, arbitrary weights (e.g., Gaussian) can be used [S2].

2.  **Gradient Estimation (Geometry Unit):**
    *   A modified 2D Sobel filter estimates the X- and Y-components of the gradient vector in the re-sampled coordinate system [S2].
    *   The gradient is 2D and view-dependent [S2].

3.  **Data Transfer:**
    *   Sample density, X-gradient, and Y-gradient values are transmitted from the geometry unit to the ray-casting unit via the Geometry Bus [S1, S2].
    *   Data is stored in FIFOs on the DSP boards to handle speed differences between the geometry unit and DSPs [S2].

4.  **Ray Casting and Shading (Ray-Casting Unit):**
    *   The ray-casting unit consists of 16 programmable Digital Signal Processors (DSPs) [S1].
    *   DSPs cast rays through the resampled dataset according to user-programmable lighting and viewing parameters [S3, unofficial].
    *   The Heidelberg illumination model is typically implemented [S1, S2].
        *   The dataset is conceptually rotated so the viewer looks along a major axis [S2].
        *   Two light sources are used: one along the viewer's direction, and one 45 degrees from the viewer's direction [S2].
        *   Light intensity is calculated slice-by-slice [S2].
        *   The final illumination value per sample is the summation of all light intensity emitted in the viewer's direction [S2].
        *   This model accounts for reflection, absorption, emission, and shadows [S2].
    *   A local master processor on the ray-casting board collects all scan lines from DSP memories and transfers the result to the host [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# VIRIM Architecture Implementation Logic

def virim_render(volume_data, view_params, classification_params, lighting_params):
    """
    volume_data: 3D array, max 256^3, 16-bit voxels
    view_params: orientation matrix, projection type
    classification_params: density LUTs (8 LUTs)
    lighting_params: light source directions, shading model
    """
    
    # 1. Geometry Unit: Resampling and Gradient Estimation
    # The geometry unit processes scanlines/projection planes
    
    # Initialize 8-way interleaved memory access
    memory_banks = [0] * 8
    
    # For each sample position in the resampled coordinate system:
    for sample_pos in resampled_grid:
        
        # Backward mapping to find source voxel coordinates in original volume
        source_coords = backward_map(sample_pos, view_params.orientation)
        
        # Identify the 8 neighboring voxels in the 8-way interleaved memory
        neighbor_indices = get_8_neighbor_indices(source_coords)
        
        # Retrieve raw voxel values
        raw_values = [volume_data[idx] for idx in neighbor_indices]
        
        # Map to density values using Density LUTs
        # Each bank has its own LUT
        density_values = [classification_params.luts[i].lookup(raw_values[i]) 
                          for i in range(8)]
        
        # Interpolate using weights from Interpolation Weight Memory
        # Weights can be arbitrary (e.g., Gaussian) or tri-linear
        weights = get_interpolation_weights(sample_pos, source_coords)
        interpolated_density = sum(density_values[i] * weights[i] for i in range(8))
        
        # Estimate 2D gradient using modified Sobel filter in re-sampled coordinates
        # Gradient is view-dependent
        gradient_x = sobel_x_filter(density_values, neighbor_indices, view_params)
        gradient_y = sobel_y_filter(density_values, neighbor_indices, view_params)
        
        # Transfer to Ray-Casting Unit via Geometry Bus
        # Data: density, gradient_x, gradient_y
        geometry_bus.send(interpolated_density, gradient_x, gradient_y)
        
    # 2. Ray-Casting Unit: DSP Processing
    # 16 DSPs process the stream of samples
    
    # Initialize Heidelberg Illumination Model
    # Light 1: Viewer direction
    # Light 2: 45 degrees from viewer direction
    
    for sample in geometry_bus.receive_stream():
        density = sample.density
        grad_x = sample.gradient_x
        grad_y = sample.gradient_y
        
        # Calculate illumination using Heidelberg model
        # Light intensity calculated slice-by-slice
        # Summation of light intensity emitted in viewer's direction
        
        # Example simplified Heidelberg calculation
        light_intensity_1 = calculate_light_contribution(density, grad_x, grad_y, 
                                                         light_direction_1=viewer_direction)
        light_intensity_2 = calculate_light_contribution(density, grad_x, grad_y, 
                                                         light_direction_2=45_deg_from_viewer)
        
        # Account for absorption/emission/shadowing as per Heidelberg model
        final_color = apply_heidelberg_shading(density, light_intensity_1, light_intensity_2, 
                                               lighting_params)
        
        # Store in DSP local memory
        dsp_memory.write(final_color)
        
    # 3. Final Assembly
    # Local master processor collects scan lines from DSP memories
    final_image = master_processor.collect_scan_lines(dsp_memory)
    
    return final_image

def calculate_light_contribution(density, grad_x, grad_y, light_direction):
    """
    Simplified placeholder for Heidelberg slice-by-slice light calculation.
    In actual hardware, this is done by DSPs.
    """
    # The Heidelberg model involves complex ray-tracing within the slice
    # accounting for reflection, absorption, emission, and shadows.
    # Here we assume a pre-computed or simplified contribution based on 
    # the provided notes that it sums intensity emitted in viewer's direction.
    return density * dot_product(normalize(grad_x, grad_y), light_direction)
```

**Parameters:**
*   **Dataset Size:** Maximum $256^3$ voxels (16-bit) [S1, S4].
*   **Geometry Bus:** 48 bits, 40 MHz, 240 MB/s peak transfer rate [S1, S2].
*   **Memory Read-out Rate:** Maximum 640 MB/s [S1].
*   **Sample Generation Rate:** Geometry unit generates max 36 million tri-linear samples per second [S1].
*   **Light Source Angle:** Second light source is 45 degrees from the viewer's direction [S2].

**Edge Cases and Pitfalls:**
*   **Polygon Intermixing:** VIRIM does not allow arbitrary intermixing of volumes and polygons [S1, S4].
*   **Parallel Scaling:** To circumvent data dependency problems in parallel configurations, the entire volume memory must be replicated in each parallel geometry unit [S4]. This contrasts with architectures like VOGUE that partition the dataset [S4].
*   **Gradient Limitation:** The gradient estimation is only 2D and view-dependent, which may limit shading accuracy compared to full 3D gradient methods [S2].

## Operating characteristics
*   **Time Complexity:** Not explicitly defined in asymptotic terms, but performance is measured in frames per second (fps) for specific dataset sizes [S1, S2].
*   **Memory Complexity:** Requires 8-way interleaved memory. In parallel setups, memory usage scales linearly with the number of modules due to full replication of the volume dataset [S4].
*   **Measured Performance:**
    *   Single module (4 boards): 10 Hz for $256^3$ datasets [S2].
    *   Four modules (16 boards): 10 Hz for $256^3$ datasets [S2].
    *   Eight modules (32 boards): 10 Hz for $256^3$ datasets [S2].
    *   Existing hardware implementation (16 processing boards): 10 fps for $256 \times 256 \times 128$ datasets [S3, unofficial].
    *   General performance: Up to 2.5 fps for $256 \times 256 \times 128$ datasets without parallelization [S1].
*   **Strengths:**
    *   High flexibility due to programmable DSPs for shading and classification [S1, S4].
    *   High-quality illumination models (Heidelberg algorithm) with shadows [S1, S2].
    *   Dedicated hardware for resampling and gradient estimation ensures speed [S2].
    *   Arbitrary interpolation weights supported [S2].
*   **Limitations:**
    *   No support for arbitrary volume-polygon intermixing [S1, S4].
    *   2D view-dependent gradient estimation limits shading fidelity [S2].
    *   Parallel scaling requires full memory replication, increasing hardware cost [S4].
*   **When to use:** When high-quality, flexible volume rendering with custom illumination models is required, and polygon integration is not a primary need [S1, S4].
*   **When not to use:** When arbitrary volume-polygon intermixing is required, or when memory efficiency in parallel scaling is critical [S1, S4].

## Background
VIRIM (Virtual Reality in Medicine) was developed at the University of Mannheim and is described in papers from the mid-1990s [S1, S2, S4]. It was commercially distributed by Volume Graphics GmbH, Heidelberg, Germany [S1]. The architecture was designed to provide a flexible and programmable solution for volume visualization, contrasting with more rigid, fixed-function accelerators of the time [S3, unofficial][S4]. It was one of the early examples of object-order ray casting implemented in dedicated hardware [S1, S2].

## Sources
[S1] Architectures for real-time volume rendering, https://doi.org/10.1016/s0167-739x(98)00051-x, fetched 2026-11-01
[S2] Ray Casting Architectures for Volume Visualization, https://vcg.seas.harvard.edu/publications/19990101-ray-casting-architetures-for-volume-visualization/paper, fetched 2026-10-27
[S3] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S4] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
