# Tiled Parallel Compositing

Also called: Tile-based compositing, Disjoint tile compositing

## Overview
Tiled parallel compositing is a rendering strategy that divides the framebuffer into disjoint spatial tiles to enable lock-free, zero-synchronization parallel processing of semi-transparent primitives (such as splats or Gaussians) [S1, unofficial]. It solves the concurrency conflict inherent in alpha compositing by ensuring that each thread writes exclusively to its assigned tile's pixels, eliminating the need for atomics or locks [S1, unofficial]. Unlike [Parallel Direct Compositing](3201-parallel-direct-compositing.md), which may require global synchronization or atomic operations for overlapping writes, tiled compositing relies on spatial partitioning to guarantee data consistency [S1, unofficial].

## How it works
**Inputs:**
- Framebuffer dimensions: width $W$, height $H$.
- Tile dimensions: width $T_W$, height $T_H$ (typically 16x16 or 16x32) [S1, unofficial][S2].
- Pre-sorted splat data: A list of projected splats `projected` and a binning structure `bins` containing offset arrays that map tile indices to ranges of splat IDs [S1, unofficial].
- Camera/Projection parameters: Used to project splats to screen space (assumed pre-computed for the compositing step) [S1, unofficial].

**Outputs:**
- Updated framebuffer: An array of RGBA values (or Vec3 + alpha) where each tile has been composited independently [S1, unofficial].

**Data Structures:**
- `ScratchBuffers`: A structure for reusable memory, containing:
  - `sort_aux`: A vector of `Projected` splats for temporary sorting or storage [S1, unofficial].
  - `tiles`: A `TileBins` structure containing offset arrays (`offsets`) that define the start and end indices of splats belonging to each tile [S1, unofficial].
- `FbPtr`: A wrapper around a raw mutable pointer to the framebuffer (`*mut (Vec3, f32)`), marked as `Send + Sync` to allow safe sharing across threads despite the borrow checker's restrictions [S1, unofficial].

**Steps:**
1. **Tile Iteration:** Iterate over all tile indices $t$ in parallel. The total number of tiles is $N_T = \lceil W/T_W \rceil \times \lceil H/T_H \rceil$ [S1, unofficial].
2. **Tile Bounds Calculation:** For each tile index $t$, determine its grid coordinates $(tx, ty)$ and pixel bounds:
   $$
   \begin{aligned}
   px0 &= tx \times T_W \\
   py0 &= ty \times T_H \\
   px1 &= \min(px0 + T_W - 1, W - 1) \\
   py1 &= \min(py0 + T_H - 1, H - 1)
   \end{aligned}
   $$
   If $px0 > px1$ or $py0 > py1$, the tile is invalid; return immediately [S1, unofficial].
3. **Splat Range Retrieval:** Retrieve the range of splat indices for this tile from the binning offsets:
   $$
   \begin{aligned}
   start &= bins.offsets[t] \\
   end &= bins.offsets[t + 1]
   \end{aligned}
   $$
   If $start == end$, the tile has no splats; return immediately [S1, unofficial].
4. **Splat Processing:** For each splat ID $sid$ in the range $[start, end)$:
   a. Retrieve the projected splat $p$ from the `projected` array [S1, unofficial].
   b. Calculate the intersection of the splat's axis-aligned bounding box $[bx0, by0, bx1, by1]$ with the tile's pixel bounds $[px0, py0, px1, py1]$:
      $$
      \begin{aligned}
      x0 &= \max(bx0, px0) \\
      y0 &= \max(by0, py0) \\
      x1 &= \min(bx1, px1) \\
      y1 &= \min(by1, py1)
      \end{aligned}
      $$
   c. If $x0 > x1$ or $y0 > y1$, the splat does not intersect the tile; continue to the next splat [S1, unofficial].
   d. Call `composite_splat_region` with the splat $p$, framebuffer pointer, width $W$, and intersection bounds $[x0, y0, x1, y1]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tiled_parallel_compositing(framebuffer, width, height, tile_width, tile_height, bins, projected):
    # Calculate tile grid dimensions
    num_tiles_x = (width + tile_width - 1) // tile_width
    num_tiles_y = (height + tile_height - 1) // tile_height
    total_tiles = num_tiles_x * num_tiles_y

    # Parallel iteration over tiles (conceptual, e.g., using rayon or thread pool)
    for tile_idx in range(total_tiles):
        # 1. Calculate tile coordinates
        tx = tile_idx // num_tiles_y
        ty = tile_idx % num_tiles_y

        # 2. Calculate pixel bounds
        px0 = tx * tile_width
        py0 = ty * tile_height
        px1 = min(px0 + tile_width - 1, width - 1)
        py1 = min(py0 + tile_height - 1, height - 1)

        # 3. Check validity
        if px0 > px1 or py0 > py1:
            continue

        # 4. Retrieve splat range
        start = bins.offsets[tile_idx]
        end = bins.offsets[tile_idx + 1]

        # 5. Check if empty
        if start == end:
            continue

        # 6. Process splats in range
        for sid in range(start, end):
            p = projected[sid]
            
            # 7. Calculate intersection with tile bounds
            x0 = max(p.bbox.x0, px0)
            y0 = max(p.bbox.y0, py0)
            x1 = min(p.bbox.x1, px1)
            y1 = min(p.bbox.y1, py1)

            # 8. Check intersection validity
            if x0 > x1 or y0 > y1:
                continue

            # 9. Composite
            composite_splat_region(p, framebuffer, width, x0, y0, x1, y1)

def composite_splat_region(splat, framebuffer, width, x0, y0, x1, y1):
    # Iterate over pixels in the intersection rectangle
    for y in range(y0, y1 + 1):
        for x in range(x0, x1 + 1):
            # Calculate pixel index
            idx = y * width + x
            
            # Retrieve current pixel color and alpha
            current_color = framebuffer[idx].color
            current_alpha = framebuffer[idx].alpha
            
            # Evaluate splat contribution at (x, y)
            # This depends on the splat type (e.g., Gaussian, billboard)
            # Assuming a function `splat_alpha` and `splat_color` exists
            s_alpha = splat_alpha(splat, x, y)
            s_color = splat_color(splat, x, y)
            
            # Alpha compositing: C_out = C_s + C_c * (1 - alpha_s)
            # Note: Order matters. For back-to-front, splat is "new" (closer) or "old" (farther)?
            # In standard forward compositing (front-to-back), the splat is the new layer.
            # However, splats are often sorted back-to-front.
            # If sorted back-to-front, we composite: C_new = C_splat + C_old * (1 - alpha_splat)
            
            new_alpha = s_alpha + current_alpha * (1.0 - s_alpha)
            if new_alpha > 0.0:
                new_color = (s_color * s_alpha) + (current_color * current_alpha * (1.0 - s_alpha))
            else:
                new_color = current_color
                new_alpha = current_alpha
            
            framebuffer[idx].color = new_color
            framebuffer[idx].alpha = new_alpha
```

**Parameters:**
- Tile Size ($T_W, T_H$): Commonly 16x16 or 16x32 pixels [S1, unofficial][S2].
- Alpha Threshold: A threshold below which transmittance is considered fully opaque, allowing early termination in some variants [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
- **Borrow Checker/Concurrency:** In languages like Rust, multiple threads cannot hold mutable references to the same buffer. This is handled by using a raw pointer wrapped in a `Send + Sync` type, relying on the geometric guarantee that tiles are disjoint [S1, unofficial].
- **Invalid Bounds:** Tiles at the image boundaries may have smaller dimensions; bounds must be clamped to image dimensions [S1, unofficial].
- **Empty Tiles:** Tiles with no overlapping splats must be skipped to avoid overhead [S1, unofficial].

## Subtopics

### Adaptive Tile Sizing
A strategy to select between two tile dimensions (16x16 and 16x32) based on the number of pixels to render to optimize performance [S2]. It is distinguished from fixed-size tiling by its dynamic adjustment based on resolution [S2]. Choose this when rendering at varying resolutions where performance characteristics change significantly with pixel count [S2]. Link: [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md).
- **Threshold:** 1.5 million pixels. If pixel count < 1.5M, use 16x32 tiles; otherwise use 16x16 tiles [S2].
- **Pitfall:** Tile orientation performance depends on memory indexing. With column-major indexing, 16x32 outperforms 32x16. With row-major indexing, the reverse is true [S2].
- **Performance:** On the low-resolution Bicycle scene, using 16x32 tiles increased the average render time to 9.77 ms, demonstrating that larger tiles are not always faster [S2].
- **Comparison:** Increasing tile size reduces duplicated Gaussians but increases sequential processing load per tile and complicates thread-block size optimization [S2].

### Tile-Based Wave Synchronization
Manages tile assignment and synchronization across worker threads using wave indices and tokens [S3, unofficial]. It is distinguished from lock-free tiling by introducing wave-based synchronization for dynamic load balancing or job cancellation [S3, unofficial]. Choose this when dynamic load balancing or early termination across waves is required [S3, unofficial]. Link: [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md).
- **Step:** A `token` holds a tile index, wave index, and job ID. It is created by `TileProvider::next()` [S3, unofficial].
- **Step:** When a token is destroyed, it updates the provider's tile wave map and notifies all waiting threads: `m_tp.m_tile_wave[m_idx] = m_wave_idx; m_tp.m_tile_wave[m_idx].notify_all();` [S3, unofficial].
- **Step:** Workers can call `stop_at_next_wave()` to finish their current wave before stopping [S3, unofficial].
- **Formula:** ETA calculation: `avg_progress_rate = progress / elapsed_time`, `eta = (1 - progress) / avg_progress_rate` [S3, unofficial].

### [Tile Boundary Computation Setup](2678-tile-boundary-computation-setup.md)
This method initializes the data structures required for tiled parallel compositing by computing which geometry overlaps each tile in the output framebuffer. It differs from [Tile-Based Alpha Compositing](1215-tile-based-alpha-compositing.md) and [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md) by relying on a global radix-sort to organize tile relationships rather than per-tile queues or expensive coverage tests during setup. Link: [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md).

### [Parallel Image Assembly](3271-parallel-image-assembly.md)
Parallel Image Assembly is the final stage of tiled parallel volume rendering where per-process or per-rank RGBA sub-images are collected and blended into a single final image. It solves the problem of reconstructing a coherent global view from disjoint spatial tiles processed independently. Unlike its siblings which handle tile distribution or boundary computation, this method focuses on the global accumulation of color and opacity, typically using front-to-back alpha compositing. Link: [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md).

### [Multi-Threaded Tile-Based Rendering](3784-multi-threaded-tile-based-rendering.md)
Multi-Threaded Tile-Based Rendering is a CPU-based parallelization strategy that divides the screen height into disjoint horizontal tiles, assigning each tile to a separate thread to parallelize the rasterization workload. It builds on [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) by providing the thread management and tile decomposition logic, allowing for lock-free, zero-synchronization processing of screen regions. Unlike [Screen Parallel Rendering](771-parallel-and-gpu-primitives.md#screen-parallel-rendering), which requires every processor to hold the entire volume in memory, this method allows for more flexible memory management as tiles are processed independently. Link: [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md).

## Operating characteristics
- **Time Complexity:** $O(N_T \times N_S)$, where $N_T$ is the number of tiles and $N_S$ is the average number of splats per tile. The parallel execution reduces wall-clock time by a factor proportional to the number of threads, assuming balanced load [S1, unofficial].
- **Memory Complexity:** $O(W \times H)$ for the framebuffer, plus $O(N_S)$ for the splat data and binning structures [S1, unofficial]. Scratch buffers add minimal overhead [S1, unofficial].
- **Measured Performance:** On the Bicycle scene, adaptive tile sizing showed that 16x32 tiles increased render time to 9.77 ms at low resolution, while 16x16 tiles were faster [S2]. At higher resolutions, 16x32 tiles were more efficient [S2].
- **Scaling:** Scales well with the number of threads as long as the load is balanced across tiles [S1, unofficial].
- **Strengths:** Zero synchronization, no atomics, no locks, making it highly effective on CPU architectures [S1, unofficial]. Simple to implement and debug [S1, unofficial].
- **Limitations:** Load imbalance can occur if splats are unevenly distributed across tiles [S1, unofficial]. Adaptive sizing adds complexity [S2].
- **When to use:** CPU-based rendering of 2D Gaussian splats or similar primitives where lock-free parallelism is desired [S1, unofficial].
- **When not to use:** When dynamic load balancing is critical and splat distribution is highly irregular, consider [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md) [S1, unofficial].

## Background
Tiled parallel compositing originated as a strategy to parallelize alpha compositing without synchronization overhead [S1, unofficial]. It has been widely adopted in CPU-based Gaussian Splatting renderers, such as `tsplat`, to leverage multi-core CPUs efficiently [S1, S1, unofficial]. The technique is also discussed in the context of 3D Gaussian Splatting optimization, where tile size adaptation is used to improve performance at different resolutions [S2].

## Sources
[S1] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S2] Fast 3D Gaussian Splatting Rendering via Easily Integrable Improvements, https://doi.org/10.1109/lsp.2024.3521379, fetched 2026-10-26
[S3] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
