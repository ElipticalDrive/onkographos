# Parallel Shear-Warp on Shared Memory

Also called: Parallel Rendering (Shared Memory), Real-time volume rendering on shared memory multiprocessors using the shear-warp factorization

## Overview
This method is a parallel volume rendering algorithm designed for shared-memory multiprocessors that minimizes computation, synchronization, and data communication time [S2, unofficial]. It builds on [Shear-warp](24-shear-warp.md) to leverage hardware support for low-latency fine-grain communication and hardware caching [S2, unofficial]. It is distinguished from distributed-memory approaches by using image decomposition where inter-node communication is performed via actual memory transfers, which are faster than network transfers [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume dataset (typically $256^3$ or $128^3$ voxels), viewing parameters (view direction, projection type), and a target image resolution.
- Output: A 2D rendered image.

**Data Structures:**
- Volume Data: Stored in physically contiguous memory accessible by all processors [S1, unofficial].
- Image Buffer: Partitioned across processors (image decomposition) [S1, unofficial].
- Task Queue: Used for dynamic load balancing to minimize synchronization events [S2, unofficial].

**Steps:**
1. **Decomposition:** The image space is partitioned among the available processors using image decomposition [S1, unofficial]. This allows each processor to work on a subset of the final pixels.
2. **Shear-Warp Execution:** Each processor executes the shear-warp algorithm [S2, unofficial]. This involves:
    - Permuting the volume data so the principal viewing axis aligns with the Z-axis [S2, unofficial].
    - Shearing the volume slices to align viewing rays with the Z-axis, allowing fixed bilinear interpolation weights [S2, unofficial].
    - Resampling sheared slices into an intermediate 2D buffer using bilinear interpolation [S2, unofficial].
    - Compositing slices in front-to-back order using the Porter-Duff "over" operator [S2, unofficial].
    - Applying the 2D warp matrix to transform the intermediate image to the final image space [S2, unofficial].
3. **Parallel Execution:** Processors independently render their assigned image partitions. Since the memory is shared, data redistribution and communication costs are low because they rely on actual memory transfers rather than network communication [S1, unofficial].
4. **Synchronization:** Dynamic load balancing is used to assign tasks to processors, minimizing the number of synchronization events required [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Shared memory environment
# volume_data: 3D array of voxel values
# image_partition: 2D array of pixels assigned to this processor
# view_params: viewing direction, projection matrix

def parallel_shear_warp_shared_memory(volume_data, image_partition, view_params):
    # 1. Image Decomposition is handled by the runtime/hardware assigning
    # specific pixel regions to this processor.
    
    # 2. Shear-Warp Factorization (per processor)
    # Determine principal axis based on view direction
    principal_axis = get_principal_axis(view_params.direction)
    
    # Permute volume data conceptually
    permuted_volume = permute_volume(volume_data, principal_axis)
    
    # Compute shear parameters
    sx, sy = compute_shear_params(view_params.direction, principal_axis)
    
    # Initialize intermediate buffer for this processor's output region
    intermediate_buffer = zeros(image_partition.height, image_partition.width)
    
    # Process each slice of the permuted volume
    for slice_idx in range(volume_depth):
        slice_data = permuted_volume[slice_idx]
        
        # Shear the slice
        sheared_slice = shear_slice(slice_data, sx, sy)
        
        # Resample and Compositing
        # For each pixel in this processor's partition
        for y, x in image_partition.pixels:
            # Map pixel (x,y) to sheared slice coordinates
            src_u, src_v = warp_to_slice_coords(x, y, view_params, slice_idx)
            
            # Bilinear interpolation
            alpha_slice, color_slice = bilinear_interpolate(sheared_slice, src_u, src_v)
            
            # Front-to-back compositing
            accum_color, accum_alpha = get_accumulated_color_alpha(intermediate_buffer, x, y)
            
            # Porter-Duff "over" operator
            new_alpha = alpha_slice + accum_alpha * (1 - alpha_slice)
            new_color = color_slice * alpha_slice + accum_color * (1 - alpha_slice)
            
            # Early ray termination
            if new_alpha > 0.95:
                break
            
            set_accumulated_color_alpha(intermediate_buffer, x, y, new_color, new_alpha)
    
    # 3. Warp to final image space
    final_image = warp_2d(intermediate_buffer, view_params.warp_matrix)
    
    # 4. Write to shared image buffer
    write_to_shared_image(final_image, image_partition)
```

**Parameters:**
- Threshold for early ray termination: 0.95 [S2, unofficial].
- Typical dataset sizes: $256^3$ voxels, $128^3$ voxels [S2, unofficial].

**Edge Cases and Pitfalls:**
- Cache locality requirements impose a limit on parallelism; shared-memory machines with hundreds of processors are only useful for rendering very large data sets [S2, unofficial].
- Scalability is limited because building systems with larger amounts of shared memory becomes prohibitively expensive [S1, unofficial].

## Subtopics

### [Perspective Shear-Warp](6047-perspective-shear-warp.md)
Extends the shear-warp factorization to perspective projection. Choose when perspective distortion correction is required.

### [Shear-Warp Algorithm](4304-shear-warp-algorithm.md)
The serial baseline algorithm. Choose when parallelism is not needed or for understanding the core factorization.

### [VolumePro Ray-Casting with Slice-by-Slice Processing](1865-volumepro-ray-casting-with-slice-by-slice-processing.md)
A hardware-accelerated approach. Choose when dedicated hardware is available.

## Operating characteristics
- **Time Complexity:** The algorithm achieves over 10 Hz for $256^3$ voxel data sets and over 30 Hz for $128^3$ voxel data sets on a 16-processor Silicon Graphics Challenge [S2, unofficial].
- **Memory Complexity:** Requires physically contiguous shared memory for the volume dataset [S1, unofficial].
- **Scaling:** Limited by cache locality requirements. Shared-memory machines with hundreds of processors are only useful for very large data sets [S2, unofficial].
- **Strengths:** Minimizes computation, synchronization, and data communication time. Leverages hardware caching to hide latency [S2, unofficial].
- **Limitations:** Prohibitively expensive to build systems with larger amounts of shared memory; cannot easily be extended [S1, unofficial].
- **When to use:** Real-time volume rendering on shared-memory multiprocessors where low-latency communication is critical.
- **When not to use:** When scaling to hundreds of processors with moderate-sized data sets, or when distributed memory systems are preferred for scalability.

## Background
This method was presented by Philippe Lacroute in January 1995 [S2, unofficial]. It was implemented on a 16-processor Silicon Graphics Challenge machine [S2, unofficial]. The work concluded that on shared-memory architectures, data redistribution and communication costs do not dominate rendering time, but cache locality imposes limits on parallelism [S2, unofficial].

## Sources
[S1] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S2] The Perspective Shear-Warp Algorithm in a Virtual Environment., https://www.researchgate.net/publication/220943822_The_Perspective_Shear-Warp_Algorithm_in_a_Virtual_Environment, fetched 2026-10-29 (unofficial)
