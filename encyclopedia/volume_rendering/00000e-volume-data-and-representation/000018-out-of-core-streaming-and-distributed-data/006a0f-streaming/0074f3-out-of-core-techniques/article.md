# Out-Of-Core Techniques

Also called: Out-of-Core Processing, Out-of-Core Strategies

## Overview
Out-of-core techniques partition volumetric data and/or computation to process subsets independently, thereby lowering in-core memory requirements when datasets exceed available main memory [S1]. Unlike in-core methods, these strategies solve the I/O bottleneck inherent in large-scale volume rendering by ensuring only necessary data blocks are loaded, often designed for batch processing rather than interactive frame rates [S1].

## How it works
**Inputs and Outputs:**
- Input: Large volumetric dataset (exceeding RAM capacity), visualization parameters (camera, transfer functions).
- Output: Rendered images or processed volume data (e.g., segmented, multi-resolution).

**Data Structures:**
- **Bricked Representation:** The volume is subdivided into spatial blocks (bricks). Each brick is a manageable chunk of the original grid, stored contiguously on disk [S1].
- **Multi-resolution Hierarchy:** A tree of data blocks at varying resolutions, allowing coarse-to-fine access [S1].

**Steps:**
1. **Partitioning:** Divide the volumetric dataset into spatial blocks (bricks) or logical chunks that fit within available memory limits [S1].
2. **Access and Prefetch:** Request data blocks from storage (disk or network) based on visibility or processing needs. Implement prefetching mechanisms to overlap I/O latency with computation [S1, PF02, CKS03].
3. **Ghost Data Generation (for Bricked Representations):** When processing bricks that share boundaries, create "ghost" or halo data on-the-fly. These are copies of neighboring brick edge values required for gradient calculations or interpolation across brick boundaries [S1, ILC10].
4. **Computation:** Perform the specific task (e.g., ray casting, segmentation, PDE solving) on the loaded brick [S1].
5. **Multi-resolution Computation (Optional):** Compute or traverse multi-resolution hierarchies to accelerate rendering or analysis by skipping empty or low-detail regions [S1, HBJP12].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def out_of_core_rendering(volume_dataset, camera, transfer_function):
    # 1. Partition data into bricks
    bricks = partition_into_bricks(volume_dataset)
    
    # 2. Determine visible bricks based on camera frustum
    visible_bricks = get_visible_bricks(bricks, camera)
    
    # 3. Initialize output image
    output_image = create_empty_image()
    
    for brick in visible_bricks:
        # 4. Access and prefetch brick data
        brick_data = load_brick_from_disk(brick.id)
        
        # 5. Create on-the-fly ghost data for boundary continuity
        ghost_data = generate_ghost_data(brick, bricks)
        
        # 6. Compute rendering for this brick
        brick_image = render_brick(
            brick_data, 
            ghost_data, 
            camera, 
            transfer_function
        )
        
        # 7. Composite brick image into output
        output_image = composite(output_image, brick_image, brick.position)
        
    return output_image
```

**Parameters:**
- **Brick Size:** Not given in the sources. Typically chosen to fit within L2/L3 cache or available RAM.
- **Prefetch Depth:** Not given in the sources. Number of bricks to preload ahead of current processing.

**Edge Cases and Pitfalls:**
- **Boundary Artifacts:** Failure to generate ghost data correctly leads to visible seams between bricks [S1].
- **I/O Bottleneck:** Insufficient prefetching or small brick sizes can cause CPU/GPU idle time waiting for data [S1].
- **Memory Pressure:** Loading too many bricks simultaneously exceeds RAM, causing thrashing [S1].

## Subtopics

### [Out-of-Core Rendering](23536-out-of-core-rendering.md)
Out-of-core rendering is a strategy for visualizing volumetric datasets that exceed the capacity of main memory by subdividing the data in space and time into manageable blocks.

## Operating characteristics
- **Time Complexity:** Depends on the number of visible bricks and I/O latency. Prefetching aims to hide I/O latency, making computation the dominant factor [S1].
- **Memory Complexity:** Proportional to the size of a single brick (or a few bricks) plus ghost data, independent of total dataset size [S1].
- **Strengths:** Enables visualization of datasets far larger than RAM; supports batch processing of large sequences [S1].
- **Limitations:** Generally not interactive due to I/O overhead; requires careful tuning of brick size and prefetching [S1].
- **When to Use:** Large-scale volume visualization where data exceeds main memory capacity [S1].
- **When Not to Use:** Interactive applications requiring real-time frame rates with small datasets [S1].

## Background
Out-of-core techniques originated from the need to handle datasets that could not fit into main memory. Silva et al. [SCC* 02] provided a comprehensive overview of out-of-core methods for visualization and graphics. Subsequent research focused on specific aspects such as data access [PF02], prefetching [CKS03], ghost data generation for bricked representations [ILC10], and multi-resolution hierarchy computation [HBJP12]. These methods are applicable at all stages of the visualization pipeline [S1].

## Sources
[S1] State-of-the-Art in GPU-Based Large-Scale Volume Visualization, https://vcg.seas.harvard.edu/publications/20150513-stateof-the-art-in-gpu-based-large-scale-volume-visualization/paper, fetched 2026-10-24
