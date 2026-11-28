# Tile-Based Rasterization

Also called: CUDA tile rasterization

## Overview
Tile-Based Rasterization is a GPU rendering primitive that assigns one CUDA thread block per image tile to process fragments locally [S1]. It parallelizes the rasterization workload by having threads within a block share splat data in fast shared memory and blend them in front-to-back order [S1]. Unlike Screen Parallel Rendering, which assigns threads directly to pixels without tile-level coordination, this method groups pixels into fixed-size tiles to optimize shared memory reuse [S1, 5725-screen-parallel-rendering.md].

## How it works
**Inputs:**
- A list of splats (e.g., 3D Gaussians) projected to screen space [G].
- A tile grid covering the resolution (16×16 pixels) [S1].
- For each tile, a sorted list of splat indices assigned to that tile [S1].

**Outputs:**
- Final color and opacity values for each pixel in the tile [G].

**Data Structures:**
- **Shared Memory Buffer:** A 1D array storing up to 128 splats per iteration [S1].
- **Thread Block:** A CUDA thread block of 256 threads, mapped one-to-one to the 16×16 pixel tile [S1].

**Parameters:**
- Tile size: 16×16 pixels [S1].
- Threads per block: 256 [S1].
- Load batch size: 128 splats [S1].

**Steps:**
1. Assign one CUDA thread block per tile, with 256 threads mapping to the 16×16 pixel grid [S1].
2. Initialize pixel buffers for color and accumulated opacity [G].
3. Ping-pong blending loop: Phase 1 uses 128 threads to load the next batch of up to 128 splats from global memory into shared memory [S1]. Phase 2 uses all 256 threads to iterate through the loaded splats and blend them in front-to-back order [S1]. Repeat until all assigned splats are processed [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_rasterize(tile_splats, thread_id, num_pixels=256, load_batch=128):
    # Map thread index to pixel coordinates in the 16x16 tile
    pixel_coords = (thread_id % 16, thread_id // 16)
    pixel_color = [0.0, 0.0, 0.0]
    pixel_alpha = 0.0

    splat_idx = 0
    num_splats = len(tile_splats)
    shared_splats = [None] * load_batch

    while splat_idx < num_splats:
        # Phase 1: Load
        load_count = 0
        if thread_id < load_batch:
            end_load = min(splat_idx + load_batch, num_splats)
            for i in range(splat_idx, end_load):
                shared_splats[load_count] = tile_splats[i]
                load_count += 1
            splat_idx = end_load

        # Synchronize all 256 threads
        sync_threads()

        # Determine how many splats were actually loaded
        num_loaded = load_count if thread_id < load_batch else 0
        # Broadcast num_loaded from thread 0 to all threads
        if thread_id == 0:
            broadcast_num_loaded = num_loaded
        sync_threads()
        num_loaded = broadcast_num_loaded

        # Phase 2: Blend
        for s in range(num_loaded):
            splat = shared_splats[s]
            if pixel_in_splat_footprint(splat, pixel_coords):
                contribution = compute_splat_contribution(splat, pixel_coords)
                pixel_color, pixel_alpha = front_to_back_blend(
                    pixel_color, pixel_alpha,
                    contribution['color'], contribution['alpha']
                )

        sync_threads()

# Helper definitions
def sync_threads():
    # Simulates CUDA __syncthreads() barrier
    pass

def pixel_in_splat_footprint(splat, coords):
    # Checks if coords are within the splat's 2D bounding box
    return True

def compute_splat_contribution(splat, coords):
    # Computes color and alpha at coords
    return {'color': [0.0, 0.0, 0.0], 'alpha': 0.0}

def front_to_back_blend(c1, a1, c2, a2):
    # Standard alpha compositing
    out_a = a1 + a2 * (1.0 - a1)
    out_c = [c1[i] + c2[i] * a2 * (1.0 - a1) for i in range(3)]
    return out_c, out_a
```

**Edge Cases:**
- **Fewer than 128 splats:** The load phase handles partial batches. The blend phase must correctly determine the number of valid splats in shared memory to avoid reading garbage data [G].
- **Empty Tile:** If no splats are assigned to a tile, the loop does not execute, and the pixel retains its background value [G].

## Subtopics

### [Binned Raster Traversal](10295-binned-raster-traversal.md)
Binned Raster Traversal is a GPU-accelerated rasterization method for Gaussian Splatting that replaces the baseline's single globally depth-sorted scan with a linear scan over a concatenation of $K$ coarse, tile-local depth bins. It solves the performance bottleneck of global sorting by deferring depth ordering to a tile-local binning phase, allowing the rasterizer to process a contiguous memory interval without per-bin branching or complex control flow. Unlike the baseline traversal, which requires a single sorted range, this method scans the concatenated bins in front-to-back order, relying on pre-computed offsets to define the scan interval.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of splats assigned to the tile and the number of pixels in the tile [G].
- **Memory Complexity:** $O(N_{splat\_per\_tile})$ for shared memory, where $N_{splat\_per\_tile}$ is capped at 128 [S1].
- **Strengths:** Reduces global memory bandwidth pressure by reusing splat data in shared memory [S1]. Scales well with tile size and splat density [G].
- **Limitations:** Requires efficient binning of splats to tiles beforehand [G]. Performance may degrade if splat distribution is highly uneven across tiles [G].

## Background
Tile-Based Rasterization for Gaussian Splatting adapts the standard 3DGS rasterization approach to a tile-based parallel model on CUDA, leveraging the hardware's thread block architecture to optimize the blending process [S1].

## Sources
[S1] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
