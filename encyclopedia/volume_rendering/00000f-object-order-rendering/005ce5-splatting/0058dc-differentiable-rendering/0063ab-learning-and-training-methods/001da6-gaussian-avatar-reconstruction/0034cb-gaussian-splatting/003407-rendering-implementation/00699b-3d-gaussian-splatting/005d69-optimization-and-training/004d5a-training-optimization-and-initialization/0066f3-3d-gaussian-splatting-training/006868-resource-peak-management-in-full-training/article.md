# Resource Peak Management in Full Training

Also called: BoundedGS Image Supply, BoundedGS Structure Updates

## Overview
This method manages memory peaks during the structure updates (density control) and image supply phases of 3D Gaussian Splatting training. It solves the problem of high GPU memory usage caused by concurrent old and new state layouts during densification and high-resolution image decoding. It sets itself apart from standard implementations by deferring layout rebuilding until after view caches are released and using bounded prefetch queues for image supply, thereby decoupling memory peaks from the Gaussian count and image resolution [S1, unofficial].

## How it works
The method consists of two primary subsystems: Structure Update Management and Image Supply Management.

### Inputs and Outputs
- **Inputs**: List of Gaussian parameters (position, scale, rotation, SH coefficients, opacity), optimizer states, statistics, and a set of training images (compressed).
- **Outputs**: Updated Gaussian parameters, optimizer states, and statistics ready for the next training step, with bounded peak memory usage.

### Data Structures
1.  **Gaussian State**: Contains parameters, optimizer state, and statistics. Layout is rebuilt only once per density control cycle.
2.  **Prefetch Queue**: A fixed-capacity queue holding decoded images. Capacity is bounded to limit residency.
3.  **Pinned Memory Buffer**: Used for transferring leaf regions.

### Steps

#### 1. Structure Update (Density Control)
Standard methods rebuild the Gaussian layout after every clone, split, parent removal, or pruning action, creating intermediate layouts with concurrent old and new state. BoundedGS defers this:

1.  **Candidate Grouping**: Group candidates and accumulate repeated IDs before a single update [S1, unofficial].
2.  **Selection**: Density control selects retained and new Gaussians based on thresholds [S1, unofficial].
3.  **Cache Release**: Ensure view caches are released before rebuilding [S1, unofficial].
4.  **Final Mapping**: Perform one final mapping to construct the new parameters, optimizer state, and statistics [S1, unofficial].
5.  **CPU-Assisted Upload**:
    -   Hold the target layout on the host (CPU) memory [S1, unofficial].
    -   Release the old device (GPU) layout [S1, unofficial].
    -   Upload the target layout to the GPU [S1, unofficial].
    -   *Note*: This trades host memory and transfer time for reduced GPU overlap peaks [S1, unofficial].

#### 2. Image Supply
To manage the memory cost of high-resolution images:

1.  **Host Storage**: Keep compressed images (e.g., JPEG) on the host CPU [S1, unofficial].
2.  **Prefetch Queue**: Decode images through a fixed-capacity prefetch queue [S1, unofficial].
3.  **Overlap**: Overlap CPU decoding work with GPU training [S1, unofficial].
4.  **Bounded Residency**: Bound the number of decoded images in memory [S1, unofficial].
5.  **Leaf Transfer**:
    -   Pack each leaf’s region in pinned memory [S1, unofficial].
    -   Transfer as `uint8` on a separate CUDA stream [S1, unofficial].
    -   Convert to floating-point supervision when used [S1, unofficial].
    -   Release the memory after loss backward [S1, unofficial].
    -   Prefetch holds at most the current and next leaf [S1, unofficial].

### Formulas
-   **Decoded Image Memory Cost**: If retaining every decoded `uint8` image, the cost is:
    $$ M_{images} \approx 3 \sum_{v} P_v \text{ bytes} $$
    Where $v$ is the number of RGB images and $P_v$ is the number of pixels per image [S1, unofficial].

### Parameters
-   **Prefetch Queue Capacity**: Bounded to hold at most the current and next leaf [S1, unofficial].
-   **Image Format**: Compressed (JPEG) on host, `uint8` during transfer, float for supervision [S1, unofficial].

### Edge Cases and Pitfalls
-   **Host Memory Pressure**: The CPU-assisted path increases host RSS. In experiments, RSS rose from 6.30 GiB to 13.37 GiB [S1, unofficial].
-   **Transfer Overhead**: Uploading the target layout from host to device adds training time (11.7% increase in one benchmark) [S1, unofficial].
-   **Precision**: Images are transferred as `uint8` and converted to float later. Ensure conversion happens before loss computation [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def boundedgs_training_step(gaussians, optimizer, images_compressed, config):
    """
    One training step with resource peak management.
    """
    # 1. Structure Update (Density Control)
    # Step 1: Group candidates and accumulate repeated IDs
    candidates = group_candidates(gaussians)
    
    # Step 2: Density control selects retained and new Gaussians
    retained_ids, new_gaussians = density_control(gaussians, candidates)
    
    # Step 3: Release view caches to free GPU memory
    release_view_caches()
    
    # Step 4: Final mapping on Host (CPU)
    # Hold target layout on host
    target_layout_host = construct_layout_host(retained_ids, new_gaussians, optimizer, gaussians.stats)
    
    # Release old device layout
    release_old_device_layout(gaussians)
    
    # Upload target to GPU
    gaussians = upload_to_device(target_layout_host)
    
    # 2. Image Supply
    # Keep compressed images on host
    # Decode through fixed-capacity prefetch queue
    current_leaf = prefetch_queue.get_current_leaf()
    next_leaf = prefetch_queue.peek_next_leaf()
    
    # Transfer leaf region
    leaf_region_uint8 = pack_leaf_region(current_leaf, pinned_memory)
    leaf_region_float = transfer_and_convert(leaf_region_uint8, stream)
    
    # Compute loss and backward
    loss = compute_loss(gaussians, leaf_region_float)
    loss.backward()
    
    # Release leaf memory after backward
    release_leaf_memory(leaf_region_float)
    
    # Update optimizer
    optimizer.step()
    optimizer.zero_grad()
    
    return gaussians
```

## Subtopics

### Host-resident Predecoded Images
Stores predecoded images on the host CPU to reduce GPU structure-update peaks during training. It uses CPU assistance for structure updates instead of repeated GPU rebuilding. This lowers the structure-update peak from 12.83 to 8.68 GiB (32.4% reduction), but raises RSS from 6.30 to 13.37 GiB and adds 11.7% training time on five 5K scenes. It exchanges transfer or decoding overhead for lower resource peaks compared to repeated GPU rebuilding [S1, unofficial].

## Operating characteristics
-   **Memory Peaks**:
    -   Structure-update peak: Reduced from 12.83 GiB to 8.68 GiB (32.4% reduction) compared to repeated GPU rebuilding [S1, unofficial].
    -   Host RSS: Increased from 6.30 GiB to 13.37 GiB [S1, unofficial].
    -   Rendering working set: Held near 1.76 GiB [S1, unofficial].
-   **Performance**:
    -   Training time: Increased by 11.7% compared to repeated GPU rebuilding (33.95 min vs 30.39 min) [S1, unofficial].
    -   Compared to host-resident predecoded images, bounded JPEG supply lowers RSS from 17.60 to 13.37 GiB (24.0% reduction) at a 5.5% training-time cost [S1, unofficial].
-   **Scaling**:
    -   Larger Gaussian counts (e.g., 12M SH3 budget) increase the expected benefit of avoiding overlapping layouts [S1, unofficial].
    -   Higher resolution images (8K or 16K) increase the expected host-memory benefit of bounded JPEG supply [S1, unofficial].
-   **Strengths**:
    -   Controls high-resolution working sets effectively.
    -   Reduces GPU memory peaks during structure updates.
-   **Limitations**:
    -   Increases host memory usage.
    -   Adds training time due to CPU-GPU transfers and decoding overhead.
-   **When to Use**:
    -   When GPU memory is the bottleneck during structure updates.
    -   When training on high-resolution images (5K and above).
-   **When Not to Use**:
    -   When host memory is limited.
    -   When training time is critical and GPU memory is sufficient.

## Background
Originated in the BoundedGS paper (2026) by the authors of "BoundedGS: Memory-Efficient High-Resolution Training for 3D Gaussian Splatting" [S1, unofficial]. The method addresses the memory peaks observed in full training of 3D Gaussian Splatting, particularly during density control and image supply phases. It was tested on Mip-NeRF 360 and Tanks and Temples datasets [S1, unofficial].

## Sources
[S1] BoundedGS: Memory-Efficient High-Resolution Training for 3D Gaussian Splatting, https://arxiv.org/abs/2610.34233v1, fetched 2026-10-26 (unofficial)
