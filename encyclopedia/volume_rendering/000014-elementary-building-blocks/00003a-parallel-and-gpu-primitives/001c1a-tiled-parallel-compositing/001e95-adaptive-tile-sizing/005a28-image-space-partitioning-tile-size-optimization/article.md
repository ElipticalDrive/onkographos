# Image-Space Partitioning (Tile Size Optimization)

Also called: Image Tile Sizing and Work Assignment Strategy, Image Tile Tuning, Work Block Sizing

## Overview
Image-Space Partitioning divides the output image into rectangular tiles to distribute ray-casting work among parallel threads, balancing the trade-off between mutex contention (small tiles) and load imbalance (large tiles) [S1]. It differs from sibling methods by explicitly treating tile dimensions as tunable parameters subject to empirical sweep, rather than fixed constants, to identify performance "sweet spots" on specific hardware [S1].

## How it works
**Inputs:**
*   Image dimensions ($W \times H$).
*   Hardware constraints (e.g., max threads per block, register limits).
*   Memory layout (e.g., row-major).

**Outputs:**
*   Optimal tile width ($T_w$) and height ($T_h$).
*   Work assignment strategy (static or dynamic).

**Data Structures:**
*   **Work List:** A queue of tiles to be processed. For dynamic assignment, this requires synchronization primitives (e.g., mutex) [S1].
*   **Tile Grid:** The image is decomposed into a grid of tiles. If $W$ is not divisible by $T_w$ or $H$ by $T_h$, boundary tiles are handled by clamping coordinates or processing partial tiles [G].

**Steps:**
1.  **Define Search Space:** Select ranges for tile width and height. Typical sweeps use powers of two: $\{1, 2, 4, \dots, 256, 512\}$ [S1].
2.  **Filter Invalid Configurations:** Exclude combinations that result in fewer than 2 total work blocks. For a $512 \times 512$ image, configurations like $256 \times 512$, $512 \times 512$, and $512 \times 256$ are invalid as they produce only 1 or 2 blocks, preventing effective parallelization [S1].
3.  **Benchmark:** For each valid $(T_w, T_h)$ pair:
    a. Render multiple views (e.g., 30 views) using the chosen work assignment strategy [S1].
    b. Measure Frame Rendering Time (FRT).
4.  **Normalize Performance:** Calculate normalized FRT for each configuration:
    $$ \text{Normalized FRT} = \frac{\text{FRT}_{config}}{\min(\text{FRT}_{all})} $$
    Values range from $1.0$ (optimal) to $\infty$ [S1].
5.  **Identify Sweet Spot:** Select the configuration with the lowest normalized FRT. In the reference study, $32 \times 32$ was optimal for CPU pthreads, and $4 \times 2$ (thread block dimensions) was optimal for CUDA FX5800 [S1].
6.  **Select Work Assignment:**
    *   For small block sizes, static assignment may be slightly faster [S1].
    *   For medium and large block sizes (e.g., $32 \times 32$), dynamic assignment is preferred due to better load balance [S1].

**Parameters:**
*   **Optimal Tile Size (CPU):** $32 \times 32$ pixels [S1].
*   **Optimal Thread Block Size (GPU, FX5800):** $4 \times 2$ threads [S1].
*   **GPU Constraints:** Max 512 threads/block, 16KB register memory per block [S1].

**Pitfalls:**
*   **Small Tiles:** Cause serialized access to the work list via mutex, degrading performance [S1].
*   **Large Tiles:** Result in insufficient tiles for good load balance [S1].
*   **Cache Misses:** Block widths of 1, 2, and 4 exhibit poor performance due to high L1, L2, and TLB cache miss rates caused by degraded spatial/temporal locality in row-major memory layouts [S1].
*   **Invalid Decompositions:** Configurations yielding 1 or 2 total blocks are invalid for parallelization [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_optimal_tile_size(image_width, image_height, hardware_constraints):
    """
    Finds optimal tile dimensions by sweeping a parameter space.
    Returns (optimal_w, optimal_h, strategy).
    """
    # Define search space: powers of 2 up to image dimensions
    widths = [1, 2, 4, 8, 16, 32, 64, 128, 256, 512]
    heights = [1, 2, 4, 8, 16, 32, 64, 128, 256, 512]
    
    # Filter widths/heights to not exceed image dimensions
    widths = [w for w in widths if w <= image_width]
    heights = [h for h in heights if h <= image_height]
    
    min_frt = float('inf')
    best_config = None
    
    for w in widths:
        for h in heights:
            # Calculate total number of tiles
            num_tiles_w = (image_width + w - 1) // w
            num_tiles_h = (image_height + h - 1) // h
            total_tiles = num_tiles_w * num_tiles_h
            
            # Pitfall: Invalid block sizes (1 or 2 total blocks)
            if total_tiles < 2:
                continue
                
            # Pitfall: GPU hardware constraints (example for FX5800)
            # If using CUDA, ensure thread block size fits register/memory limits
            # e.g., max 512 threads, 16KB registers
            if is_gpu_hardware(hardware_constraints):
                if w * h > 512: # Max threads per block
                    continue
                # Additional register/memory checks would go here [S1]
            
            # Benchmark this configuration
            frt = measure_frame_rendering_time(w, h, image_width, image_height)
            
            if frt < min_frt:
                min_frt = frt
                best_config = (w, h)
                
    # Determine work assignment strategy
    # Dynamic is preferred for medium/large tiles for better load balance [S1]
    strategy = "dynamic" if best_config[0] * best_config[1] >= 32*32 else "static"
    
    return best_config, strategy

def measure_frame_rendering_time(tile_w, tile_h, img_w, img_h):
    """
    Renders multiple views and returns the minimum Frame Rendering Time (FRT).
    """
    frts = []
    for view in range(30): # Sweep over 30 views [S1]
        # Initialize work list with tiles
        work_list = create_tile_grid(img_w, img_h, tile_w, tile_h)
        
        # Render using parallel threads
        # If dynamic: threads pop from work_list (mutex protected) [S1]
        # If static: threads are assigned fixed ranges of work_list [S1]
        render_time = parallel_raycast(work_list)
        frts.append(render_time)
        
    return min(frts) # Minimum times represent optimal, contention-free conditions [S1]
```

## Operating characteristics
*   **Performance Variation:** The worst-performing tile size can be up to 3.0 times slower than the best configuration (normalized FRT max ~3.0) [S1].
*   **Complexity:** The optimization phase is $O(N \cdot V \cdot T)$, where $N$ is the number of tile configurations, $V$ is the number of views, and $T$ is the render time per view. This is a one-time or infrequent cost.
*   **Strengths:** Identifies hardware-specific optimal parameters; dynamic assignment handles load imbalance well for medium/large tiles [S1].
*   **Limitations:** Requires empirical tuning; small tiles suffer from mutex overhead; large tiles suffer from load imbalance; row-major memory layout exacerbates cache misses for certain tile shapes (e.g., width 1, 2, 4) [S1].
*   **When to Use:** When deploying to specific hardware architectures where tile size significantly impacts cache behavior and synchronization overhead [S1].

## Background
The method originates from studies on hybrid parallelism for volume rendering on multi-core systems, specifically analyzing shared-memory raycasting on AMD Opteron CPUs and CUDA GPUs (FX5800) [S1]. It addresses the need to balance parallel efficiency with memory access patterns in image-space partitioning [S1].

## Sources
[S1] Hybrid Parallelism for Volume Rendering on Large, Multi- and Many-core Systems, https://digital.library.unt.edu/ark:/67531/metadc840908/?q=out-of-core volume rendering, fetched 2026-10-28
