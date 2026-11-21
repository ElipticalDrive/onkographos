# Tiled Parallel Compositing


## Overview
Tiled parallel compositing partitions the output framebuffer into disjoint spatial tiles, enabling independent, parallel processing of each tile with zero synchronization, no atomics, and no locks [S1, unofficial]. It solves concurrency conflicts in parallel volume rendering or splatting by ensuring each thread writes exclusively to its assigned pixel region [S1, unofficial]. It sets itself apart from [Atomic Fragment List Insertion](2465-atomic-fragment-list-insertion.md) by avoiding atomic contention, and from [Parallel Direct Compositing](3201-parallel-direct-compositing.md) by relying on spatial partitioning rather than data partitioning for parallelism [G].

## How it works
**Inputs:**
- `w_i`, `h_i`: Framebuffer width and height in pixels [S1, unofficial].
- `TILE_W`, `TILE_H`: Tile dimensions in pixels [S1, unofficial].
- `bins.offsets`: An array of size `num_tiles + 1` defining the start and end indices of splats assigned to each tile [S1, unofficial].
- `projected`: An array of projected splat data (position, covariance, color) indexed by splat ID [S1, unofficial].
- `fb`: A framebuffer buffer storing color and alpha values (e.g., `Vec3` color and `f32` alpha) [S1, unofficial].

**Data Structures:**
- `ScratchBuffers`: Reusable memory for intermediate calculations.
  - `sort_aux: Vec<Projected>`: Auxiliary storage for sorting or temporary projection data [S1, unofficial].
  - `tiles: TileBins`: Structure containing the `offsets` array mapping tiles to splat ranges [S1, unofficial].
- `FbPtr`: A `Send + Sync` wrapper around `*mut (Vec3, f32)` that enables safe concurrent mutable access to the framebuffer across threads, relying on the geometric guarantee that tile $(t_x, t_y)$ only writes to pixels $[t_x \cdot TILE\_W .. (t_x+1) \cdot TILE\_W, t_y \cdot TILE\_H .. (t_y+1) \cdot TILE\_H]$ [S1, unofficial].

**Parameters:**
- `TILE_W`, `TILE_H`: Tile grid dimensions in pixels [S1, unofficial].
- `AlphaThreshold`: Minimum alpha value required to contribute to the framebuffer, used to skip transparent pixels during compositing [G].

**Steps:**
1. **Parallel Iteration:** Iterate over all tile indices $t$ in parallel using a thread pool or parallel iterator [S1, unofficial].
2. **Tile Bounds Calculation:** For each tile index $t$, determine grid coordinates $(t_x, t_y)$ and pixel bounds:
   $$
   \begin{aligned}
   px0 &= t_x \cdot TILE\_W \\
   py0 &= t_y \cdot TILE\_H \\
   px1 &= \min(px0 + TILE\_W - 1, w_i - 1) \\
   py1 &= \min(py0 + TILE\_H - 1, h_i - 1)
   \end{aligned}
   $$
   [S1, unofficial].
3. **Validity Check:** If $px0 > px1$ or $py0 > py1$, the tile is invalid or outside bounds; return immediately [S1, unofficial].
4. **Splat Range Retrieval:** Retrieve the splat index range for this tile:
   $$
   \begin{aligned}
   start &= bins.offsets[t] \\
   end &= bins.offsets[t + 1]
   \end{aligned}
   $$
   [S1, unofficial].
5. **Empty Check:** If $start == end$, the tile has no splats; return immediately [S1, unofficial].
6. **Splat Processing:** For each splat ID $sid$ in the range $[start, end)$:
   a. Retrieve the projected splat $p$ using `projected.get_unchecked(sid)` [S1, unofficial].
   b. Calculate the intersection of the splat's bounding box $[bx0, by0, bx1, by1]$ with the tile's pixel bounds $[px0, py0, px1, py1]$:
      $$
      \begin{aligned}
      x0 &= \max(bx0, px0) \\
      y0 &= \max(by0, py0) \\
      x1 &= \min(bx1, px1) \\
      y1 &= \min(by1, py1)
      \end{aligned}
      $$
      [S1, unofficial].
   c. If $x0 > x1$ or $y0 > y1$, the splat does not intersect the tile; continue to the next splat [S1, unofficial].
   d. Call `composite_splat_region(p, fb, w_i, x0, y0, x1, y1)` to render the splat into the intersecting pixels [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tiled_parallel_compositing(w_i, h_i, TILE_W, TILE_H, bins, projected, fb):
    num_tiles_x = (w_i + TILE_W - 1) // TILE_W
    num_tiles_y = (h_i + TILE_H - 1) // TILE_H
    num_tiles = num_tiles_x * num_tiles_y
    
    # Parallel loop over tile indices
    for t in range(num_tiles):
        # 1. Calculate tile coordinates and pixel bounds
        t_x = t % num_tiles_x
        t_y = t // num_tiles_x
        
        px0 = t_x * TILE_W
        py0 = t_y * TILE_H
        px1 = min(px0 + TILE_W - 1, w_i - 1)
        py1 = min(py0 + TILE_H - 1, h_i - 1)
        
        # 2. Check validity
        if px0 > px1 or py0 > py1:
            continue
            
        # 3. Get splat range
        start = bins.offsets[t]
        end = bins.offsets[t + 1]
        
        # 4. Check if empty
        if start == end:
            continue
            
        # 5. Process splats in this tile
        for sid in range(start, end):
            # Get projected splat data
            p = projected[sid]
            
            # Calculate intersection with tile bounds
            x0 = max(p.bx0, px0)
            y0 = max(p.by0, py0)
            x1 = min(p.bx1, px1)
            y1 = min(p.by1, py1)
            
            # Check intersection validity
            if x0 > x1 or y0 > y1:
                continue
                
            # Composite splat into framebuffer
            composite_splat_region(p, fb, w_i, x0, y0, x1, y1)

def composite_splat_region(p, fb, width, x0, y0, x1, y1):
    # Standard front-to-back alpha compositing for the intersecting region [G]
    for y in range(y0, y1 + 1):
        for x in range(x0, x1 + 1):
            idx = y * width + x
            # Blend splat color/alpha into framebuffer at (x, y)
            fb[idx].color = blend(fb[idx].color, p.color, p.alpha)
            fb[idx].alpha = blend_alpha(fb[idx].alpha, p.alpha)
```

**Edge Cases and Pitfalls:**
- **Borrow Checker/Concurrency:** In languages like Rust, multiple threads cannot hold `&mut` references to the same buffer. This is handled by wrapping the raw framebuffer pointer in a `Send + Sync` newtype (`FbPtr`) and using unsafe blocks, relying on the disjoint tile geometry to guarantee no data races [S1, unofficial].
- **Tile Bounds:** Tiles at the image boundary must clamp pixel bounds to prevent out-of-bounds memory access [S1, unofficial].
- **Empty Tiles:** Tiles with no overlapping geometry must be skipped early to avoid unnecessary overhead [S1, unofficial].

## Subtopics

### [Tile-Based Alpha Compositing](1215-tile-based-alpha-compositing.md)
Tile-Based Alpha Compositing blends multiple semi-transparent primitives into a final image by processing them in depth order from nearest to farthest within disjoint tiles. It is distinguished from back-to-front compositing by its tile-based parallelization, which allows independent processing of each tile without synchronization [G]. Choose this when rendering overlapping splats or Gaussians that require strict depth ordering per tile.

### [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md)
Dynamic Load Balancing via Tile Queue splits screen space into tiles and assigns them to processors via a shared queue to ensure high processor utilization over a wide range of views. It addresses the unpredictability of isosurface extraction complexity per pixel, which static partitioning cannot handle efficiently. It sets itself apart from static partitioning by dynamically adjusting task granularity to balance load and reduce contention.

### [Adaptive Tile Sizing](1805-adaptive-tile-sizing.md)
A strategy to select between two tile dimensions (16x16 and 16x32) based on the number of pixels to render to optimize performance [G]. It sets itself apart from fixed-size tiling by adapting to resolution, reducing duplicated Gaussians at high resolutions while avoiding excessive sequential load per tile at low resolutions [G]. Choose this when rendering at variable resolutions.

### [Tile Boundary Computation Setup](2678-tile-boundary-computation-setup.md)
This method initializes the data structures required for tiled parallel compositing by computing the spatial extent (boundaries) of each tile within the framebuffer [G]. It solves the problem of determining which input primitives or data elements contribute to which output tiles, enabling subsequent parallel processing stages to operate on disjoint subsets of the image without synchronization [G]. It differs from sibling methods like [Coarse Raster](2807-coarse-raster.md) by focusing on the setup of the boundary metadata itself rather than the rasterization of geometry, and from [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md) by being a static setup phase rather than a runtime scheduling mechanism [G].

### [Parallel Image Assembly](3271-parallel-image-assembly.md)
Parallel Image Assembly is the final step in spatial decomposition volume rendering where sub-images rendered by independent threads or processes are combined into a single final image. It solves the problem of merging disjoint spatial data portions into a coherent visual output by placing sub-images at their correct spatial coordinates and blending them along the z-axis using front-to-back alpha compositing. Unlike temporal sequencing, which has minimal overhead, spatial compositing incurs significant overhead due to the data collection and blending requirements.

### [Multi-Threaded Tile-Based Rendering](3784-multi-threaded-tile-based-rendering.md)
Multi-Threaded Tile-Based Rendering is a parallelization strategy that divides the screen height into tiles and processes each tile in a separate thread to parallelize the rasterization workload. It differs from [Tile-Based Alpha Compositing](1215-tile-based-alpha-compositing.md) by focusing on the rasterization phase itself rather than just compositing pre-rasterized fragments, and from [Screen Parallel Rendering](5725-screen-parallel-rendering.md) by dividing the screen into tiles processed by threads on a single machine rather than subscreens distributed across multiple processors [G].

### [Tile-Based Wave Synchronization](5069-tile-based-wave-synchronization.md)
Manages tile assignment and synchronization across worker threads using wave indices and tokens [G]. It sets itself apart from lock-free tiling by providing explicit synchronization points for tasks requiring global consistency or progress tracking [G]. Choose this when implementing a path tracer or ray caster that requires wave-based synchronization [G].

### [Parallel Binary-Swap Compositing](10844-parallel-binary-swap-compositing.md)
Parallel Binary-Swap Compositing is a method for combining independently rendered image tiles from multiple processors in a massively parallel environment. It solves the synchronization and communication bottlenecks of naive parallel compositing by reducing the data volume at each step, ensuring all processors remain busy with useful work. Unlike approaches where communication dominates or processors idle, this method maintains full processor utilization by pairing processors in a binary-swap fashion.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of tiles and available threads, provided the workload is balanced [G].
- **Strengths:** Zero synchronization overhead, no atomics, no locks, effective on CPU [S1, unofficial]. Simple to implement with standard parallel loops [S1, unofficial].
- **Limitations:** Load imbalance occurs if geometry clusters in specific tiles [G].
- **When to use:** When rendering large numbers of independent primitives (e.g., Gaussians, splats) with minimal inter-tile dependencies [G].
- **When not to use:** When global synchronization is required between tiles [G].

## Background
Tiled parallel compositing is a standard spatial decomposition technique for parallel rendering [G]. It leverages spatial decomposition to enable lock-free parallelism [S1, unofficial]. The approach has been adapted for various hardware architectures, including CPU multi-threading (e.g., using `rayon` in Rust) and GPU compute shaders [S1, S2, unofficial].

## Sources
[S1] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
