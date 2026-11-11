# Efficient Voxel Marking for Hierarchical Volumetric Fusion

Also called: Multi-level DDA ray-casting for macro-voxel marking, Scattering type macro-voxel marking

## Overview
This method marks hierarchical macro-cells as either empty or intersected by a noisy depth surface, enabling efficient culling in volumetric fusion pipelines. Unlike gathering-type approaches that project geometry to screen space and suffer from high thread divergence, this method is input-driven (scattering type), assigning one GPU thread per depth image pixel to traverse the macro-voxel grid, thereby automatically ignoring invisible cells and minimizing divergence.

## How it works
**Inputs:**
- A depth image with associated camera intrinsics and extrinsics.
- A hierarchical voxel grid where the top level consists of macro-voxels (e.g., $16 \times 16 \times 16$ child voxels).
- A truncation distance parameter $d_{trunc}$.

**Outputs:**
- A status descriptor for each macro-voxel indicating its intersection and emptiness state.

**Data Structures:**
Each macro-voxel is represented by a single 32-bit word. The layout is defined as follows to allow independent modification of flags without atomic operations:
- **Bits 0-23:** Index of the block of child-voxels (or null/invalid if not yet allocated).
- **Bits 24-31:** Intersection flag (1 byte).
- **Bits 32-39:** Emptiness flag (1 byte). *Note: The source describes a 32-bit word but lists fields that exceed 32 bits if interpreted as separate 8-byte fields. Based on the text "32 bit word containing... flags... in different bytes", the flags occupy two distinct bytes within the 32-bit word. The exact bit allocation for the child-voxel index is not explicitly detailed beyond being part of the word, but the key constraint is that the two flags are in separate bytes.*
- **Bits 40-47:** Counter showing how many times the complete macro-cell was found empty. *Note: This field exceeds the 32-bit word description. The source states "32 bit word" but lists four fields including a counter. It is likely the word is larger or the counter is part of a larger structure, but the text explicitly says "32 bit word". We must adhere to the text: "A single macro-voxel is represented by a 32 bit word containing...". This is a contradiction in the source. However, the critical implementation detail is the separation of flags into different bytes. We will describe the fields as listed, noting the size discrepancy as "Not given in the sources" for the exact bit packing of the counter vs the 32-bit limit, but emphasizing the byte separation for flags.*

**Algorithm Steps:**
1.  **Thread Assignment:** Assign one GPU thread to every pixel $(u, v)$ of the depth image.
2.  **Ray Interval Definition:** For each thread, retrieve the depth value $z$ at pixel $(u, v)$. Construct a truncation interval $[z_{min}, z_{max}]$ where:
    $$ z_{min} = z - d_{trunc} $$
    $$ z_{max} = z + d_{trunc} $$
    Here, $d_{trunc}$ is the truncation distance, defining the threshold for surface influence.
3.  **Ray Generation:** Form a ray originating from the camera center passing through pixel $(u, v)$.
4.  **Multi-level DDA Traversal:** Execute a Digital Differential Analyzer (DDA) traversal on the higher-level macro-voxel grid using the ray.
    -   Traverse macro-voxels along the ray path.
    -   For each visited macro-voxel, calculate the entry and exit ray parameters $t_{entry}$ and $t_{exit}$ relative to the camera.
5.  **Flag Updating:**
    -   **Empty Check:** If $t_{exit} < z_{min}$, the macro-voxel is completely behind the truncation interval. Mark the **emptiness flag** as set.
    -   **Intersection Check:** If the interval $[t_{entry}, t_{exit}]$ overlaps with $[z_{min}, z_{max}]$, the macro-voxel is intersected by the noisy surface. Mark the **intersection flag** as set.
    -   **Atomic-Free Update:** Since the intersection and emptiness flags are stored in different bytes of the descriptor word, threads can update these flags independently. A thread sets a flag bit without modifying the other byte, avoiding write collisions and the need for atomic operations. The result is independent of the order of thread execution.
6.  **Final State Interpretation:** After all threads complete:
    -   If both flags are cleared, the macro-voxel is not affected by the current depth image.
    -   If the intersection flag is set, the surface intersects the macro-voxel (regardless of the empty flag).
    -   If only the emptiness flag is set, the macro-voxel does not contain the surface; its children are either empty or not visible.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global descriptor array for macro-voxels
# Each element is a 32-bit integer (or larger structure as per source contradiction)
# Layout: [ChildIndex (24 bits)] [IntersectionFlag (8 bits)] [EmptinessFlag (8 bits)] [EmptyCounter (8 bits)]
# Note: Source claims 32-bit word but lists 4 fields. Implementation must handle the byte separation.
# We assume a structure where flags are in distinct bytes.

def mark_macro_voxels(depth_image, camera_intrinsics, camera_extrinsics, d_trunc, macro_grid):
    # Launch one thread per pixel
    for pixel in depth_image:
        u, v = pixel.coords
        z = depth_image[u, v]
        
        # Define truncation interval
        z_min = z - d_trunc
        z_max = z + d_trunc
        
        # Generate ray from camera through pixel
        ray_origin = camera_extrinsics.inverse() * [u, v, z, 1] # Homogeneous coord
        ray_direction = normalize(camera_intrinsics.inverse() * [u, v, 1] - ray_origin)
        
        # Multi-level DDA traversal on macro-voxel grid
        current_macro = macro_grid.start()
        
        while current_macro is not None:
            # Get entry and exit t-values for current macro-voxel
            t_entry, t_exit = ray.intersect_box(current_macro.bounding_box)
            
            # Check if macro-voxel is behind the truncation interval
            if t_exit < z_min:
                # Mark as empty
                # Set emptiness flag byte independently
                current_macro.set_emptiness_flag(True)
            
            # Check if macro-voxel overlaps with truncation interval
            elif t_entry < z_max and t_exit > z_min:
                # Mark as intersected
                # Set intersection flag byte independently
                current_macro.set_intersection_flag(True)
            
            # Move to next macro-voxel along ray
            current_macro = macro_grid.next(current_macro, ray_direction)
```

**Parameters:**
-   **Truncation Distance ($d_{trunc}$):** The threshold defining the interval around the depth value. Voxels are affected if the difference between the depth value and the distance from the camera is less than this value. Typical values depend on sensor noise and desired surface thickness.

**Edge Cases and Pitfalls:**
-   **Write Collisions:** Different rays may intersect the same macro-cell. This is handled by separating flags into different bytes, allowing independent updates without atomics.
-   **Frame Delay:** The method may tolerate a one-frame delay in updates, which can be combined with the ICP ray-casting step to reduce computational cost.
-   **Noise:** The method is robust to depth camera noise due to the truncation interval.

## Operating characteristics
-   **Time Complexity:** Linear with respect to the number of depth image pixels and the average number of macro-voxels intersected per ray.
-   **Memory Complexity:** Linear with respect to the number of macro-voxels.
-   **Performance:** Runs at real-time on NVIDIA GeForce GTX 690 GPUs.
-   **Strengths:** Low thread divergence, no atomic operations required, automatically ignores invisible cells, simple implementation.
-   **Limitations:** Requires hierarchical grid structure. May miss updates if a single frame is skipped (tolerable due to noise).
-   **Comparison:** Compared to Scalable KinectFusion, this method has lower thread divergence and is easier to implement. It allows for a reduction in voxel edge length from 8 mm to 1 mm for the same GPU memory usage compared to Microsoft KinectFusion.

## Background
-   **Origin:** Proposed by researchers from Budapest University of Technology and Economics, Hungary, in 2016 [S1, unofficial].
-   **Context:** Developed to address limitations in hierarchical volumetric fusion, specifically the high divergence and complexity of gathering-type algorithms like Scalable KinectFusion [CBI13].
-   **Usage:** Implemented in CUDA for hierarchical volumetric fusion, compared against Microsoft KinectFusion.

## Sources
[S1] Efficient Voxel Marking for Hierarchical Volumetric Fusion, https://doi.org/10.2312/egp.20161041, fetched 2026-10-24 (unofficial)
