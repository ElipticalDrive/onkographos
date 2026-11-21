# Parallel Shear-Warp on Shared Memory

Also called: Parallel Rendering (Shared Memory), Real-time volume rendering on shared memory multiprocessors using the shear-warp factorization

## Overview
This method is a parallel volume rendering algorithm designed for shared-memory multiprocessors that minimizes computation, synchronization, and data communication time [S2, unofficial]. It builds on [Shear-warp](24-shear-warp.md) to leverage hardware support for low-latency fine-grain communication and hardware caching [S2, unofficial]. It is distinguished from distributed-memory approaches by using image decomposition where inter-node communication is performed via actual memory transfers, which are faster than network transfers [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume dataset (typically $256^3$ or $128^3$ voxels), viewing parameters (view direction, projection type), and a target image resolution [S2, unofficial].
- Output: A 2D rendered image.

**Data Structures:**
- Volume Data: Stored in physically contiguous memory accessible by all processors [S1, unofficial].
- Image Buffer: Partitioned across processors using image decomposition [S1, unofficial].
- Task Queue: Used for dynamic load balancing to minimize synchronization events [S2, unofficial].

**Steps:**
1. **Decomposition:** The image space is partitioned among the available processors using image decomposition [S1, unofficial].
2. **Shear-Warp Execution:** Each processor executes the shear-warp algorithm [S2, unofficial]. This involves permuting the volume data so the principal viewing axis aligns with the Z-axis, shearing the volume slices to align viewing rays with the Z-axis, resampling sheared slices into an intermediate 2D buffer using bilinear interpolation, compositing slices in front-to-back order using the Porter-Duff "over" operator, and applying the 2D warp matrix to transform the intermediate image to the final image space [S2, unofficial].
3. **Parallel Execution:** Processors independently render their assigned image partitions. Since the memory is shared, data redistribution and communication costs are low because they rely on actual memory transfers rather than network communication [S1, unofficial].
4. **Synchronization:** Dynamic load balancing is used to assign tasks to processors, minimizing the number of synchronization events required [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_shear_warp_shared_memory(volume_data, image_partition, view_direction, img_w, img_h):
    # 1. Image Decomposition: image_partition specifies the pixel subgrid [y0:y1, x0:x1] assigned to this processor.
    # 2. Shear-Warp Factorization
    abs_dir = [abs(view_direction[0]), abs(view_direction[1]), abs(view_direction[2])]
    principal_axis = abs_dir.index(max(abs_dir))
    volume_depth = len(volume_data)

    vx, vy, vz = view_direction
    if principal_axis == 2:
        sx, sy = -vx / vz, -vy / vz
    elif principal_axis == 1:
        sx, sy = -vx / vy, -vz / vy
    else:
        sx, sy = -vy / vx, -vz / vx

    h, w = image_partition.height, image_partition.width
    intermediate_buffer = [[0.0] * w for _ in range(h)]
    accum_alpha_buffer = [[0.0] * w for _ in range(h)]

    for z in range(volume_depth):
        slice_data = volume_data[z]
        for dy in range(h):
            for dx in range(w):
                src_u = dx + sx * z
                src_v = dy + sy * z
                alpha_slice, color_slice = bilinear_interpolate(slice_data, src_u, src_v)
                accum_color = intermediate_buffer[dy][dx]
                accum_alpha = accum_alpha_buffer[dy][dx]
                new_alpha = alpha_slice + accum_alpha * (1.0 - alpha_slice)
                new_color = color_slice * alpha_slice + accum_color * (1.0 - alpha_slice)
                if new_alpha > 0.95:
                    break
                intermediate_buffer[dy][dx] = new_color
                accum_alpha_buffer[dy][dx] = new_alpha

    final_image = apply_2d_warp(intermediate_buffer, view_direction)
    write_to_shared_memory(image_partition, final_image)

def bilinear_interpolate(data, u, v):
    # Standard bilinear interpolation on 2D slice data. Not given in the sources. [G]
    u0, v0 = int(u), int(v)
    du, dv = u - u0, v - v0
    c00 = data[v0][u0]
    c10 = data[v0][u0+1]
    c01 = data[v0+1][u0]
    c11 = data[v0+1][u0+1]
    color = (1-du)*(1-dv)*c00 + du*(1-dv)*c10 + (1-du)*dv*c01 + du*dv*c11
    return 1.0, color

def apply_2d_warp(buffer, view_direction):
    # Applies 2D affine warp matrix M_warp2D to transform intermediate image to final image space. Not given in the sources. [G]
    return buffer

def write_to_shared_memory(partition, image):
    # Writes rendered pixels to the physically contiguous shared memory buffer. Not given in the sources. [G]
    pass
```

**Parameters:**
- Threshold for early ray termination: Not given in the sources.
- Typical dataset sizes: $256^3$ voxels, $128^3$ voxels [S2, unofficial].

**Edge Cases and Pitfalls:**
- Cache locality requirements impose a limit on parallelism; shared-memory machines with hundreds of processors are only useful for rendering very large data sets [S2, unofficial].
- Scalability is limited because building systems with larger amounts of shared memory becomes prohibitively expensive and they cannot easily be extended [S1, unofficial].

## Subtopics

### [Perspective Shear-Warp](6047-perspective-shear-warp.md)
Extends the shear-warp factorization to perspective projection. Choose when perspective distortion correction is required.

### [Shear-Warp Algorithm](4304-shear-warp-algorithm.md)
The serial baseline algorithm. Choose when parallelism is not needed or for understanding the core factorization.

### [VolumePro Ray-Casting with Slice-by-Slice Processing](1865-volumepro-ray-casting-with-slice-by-slice-processing.md)
A hardware-accelerated approach. Choose when dedicated hardware is available.

### Parallel Shear-Warp (Distributed Memory)
A parallel shear-warp implementation on distributed memory architectures that partitions in sheared volume space and uses adaptive load balancing [S3, unofficial]. It divides the volume data after the shear operation into subvolumes parallel to an intermediate image plane [S3, unofficial].

## Operating characteristics
- **Time Complexity:** The algorithm achieves over 10 Hz for $256^3$ voxel data sets and over 30 Hz for $128^3$ voxel data sets on a 16-processor Silicon Graphics Challenge [S2, unofficial].
- **Memory Complexity:** Requires physically contiguous shared memory for the volume dataset [S1, unofficial].
- **Scaling:** Limited by cache locality requirements. Shared-memory machines with hundreds of processors are only useful for very large data sets [S2, unofficial].
- **Strengths:** Minimizes computation, synchronization, and data communication time. Leverages hardware caching to hide latency [S2, unofficial].
- **Limitations:** Prohibitively expensive to build systems with larger amounts of shared memory; cannot easily be extended [S1, unofficial].
- **When to use:** Real-time volume rendering on shared-memory multiprocessors where low-latency communication is critical [S2, unofficial].
- **When not to use:** When scaling to hundreds of processors with moderate-sized data sets, or when distributed memory systems are preferred for scalability [S1, unofficial][S2, unofficial].

## Background
This method was presented by Philippe Lacroute in January 1995 [S2, unofficial]. It was implemented on a 16-processor Silicon Graphics Challenge machine [S2, unofficial]. The work concluded that on shared-memory architectures, data redistribution and communication costs do not dominate rendering time, but cache locality imposes limits on parallelism [S2, unofficial].

## Sources
[S1] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S2] The Perspective Shear-Warp Algorithm in a Virtual Environment., https://www.researchgate.net/publication/220943822_The_Perspective_Shear-Warp_Algorithm_in_a_Virtual_Environment, fetched 2026-10-29 (unofficial)
[S3] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
