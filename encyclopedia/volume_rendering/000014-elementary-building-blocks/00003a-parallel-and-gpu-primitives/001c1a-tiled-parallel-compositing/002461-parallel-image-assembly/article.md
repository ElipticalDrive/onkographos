# Parallel Image Assembly

Also called: Final Image Assembly & Alpha Blending, Image Stitching, Z-Order Compositing

## Overview
Parallel Image Assembly assembles per-process RGBA sub-images into a final image by placing them in their spatial coordinates and blending along the z-axis [S1, unofficial]. It solves the problem of reconstructing a coherent global view from disjoint spatial tiles processed independently [S1, S2, unofficial]. Unlike its siblings which handle tile distribution or boundary computation, this method focuses on the global accumulation of color and opacity [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Per-rank composited sub-images (color and opacity arrays), rank coordinates $(px, py, pz)$, global image dimensions $(X, Y)$, and the number of depth slices $PZ$ [S1, S2, unofficial].
- **Outputs:** A final RGBA image array and optionally a total opacity map [S1, S2, unofficial].

**Data Structures:**
- `final_image`: A numpy array of shape $(X, Y, 4)$ with dtype `float32` storing the final RGBA image [S1, unofficial], or $(X, Y, 3)$ for RGB with a separate opacity array [S2, unofficial].
- `total_opacity`: An array of shape $(X, Y)$ storing accumulated opacity [S2, unofficial].
- `z_split_images`: Intermediate storage for sub-images placed at spatial coordinates [S1, unofficial].

**Parameters:**
- `LOCATION_OF_Z`: Sorted unique z-start coordinates for each process chunk [S1, unofficial].
- `PZ`: Reference value for depth slice indexing, used to identify the middle depth slice via $pz == PZ // 2$ [S2, unofficial].

**Steps:**
1. **Gather:** Rank 0 gathers composited subdomain images and opacities from all other ranks [S1, S2, unofficial].
2. **Initialize:** On Rank 0, initialize `final_image` to zeros with shape $(PX \times sub\_height, PY \times sub\_width, 4)$ or $(PX \times sub\_height, PY \times sub\_width, 3)$ depending on implementation [S1, S2, unofficial].
3. **Place:** Iterate through all gathered ranks. For each rank, determine its $(px, py, pz)$ coordinates [S2, unofficial].
4. **Blend:** For each z-slice (or specifically the middle depth slice $pz == PZ // 2$ as per some implementations), blend the sub-image into `final_image` using front-to-back alpha compositing [S1, S2, unofficial].
   - Formula:
     $$ C_{final} = C_{prev} + C_{z} \cdot (1 - \alpha_{prev}) $$
     $$ \alpha_{final} = \alpha_{prev} + \alpha_{z} \cdot (1 - \alpha_{prev}) $$
   - Where $C$ is color (RGB) and $\alpha$ is opacity [S1, unofficial].
5. **Convert:** Convert to `uint8` RGBA and save as PNG [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def parallel_image_assembly(gathered_colors, gathered_opacities, rank_coords, px, py, pz, sub_width, sub_height, x_dim, y_dim):
    """
    Assembles per-rank sub-images into a final image.
    
    Args:
        gathered_colors: List of color arrays from all ranks.
        gathered_opacities: List of opacity arrays from all ranks.
        rank_coords: List of (px, py, pz) tuples for each rank.
        px, py, pz: Dimensions of the process grid.
        sub_width, sub_height: Width and height of each sub-image.
        x_dim, y_dim: Total image dimensions.
        
    Returns:
        final_image: numpy array of shape (x_dim, y_dim, 4) dtype uint8.
    """
    final_image = np.zeros((x_dim, y_dim, 4), dtype=np.float32)
    total_opacity = np.zeros((x_dim, y_dim), dtype=np.float32)
    
    for rank_idx, (r_px, r_py, r_pz) in enumerate(rank_coords):
        color = gathered_colors[rank_idx]
        opacity = gathered_opacities[rank_idx]
        
        x_start = r_px * sub_width
        y_start = r_py * sub_height
        x_end = x_start + sub_width
        y_end = y_start + sub_height
        
        current_alpha = final_image[y_start:y_end, x_start:x_end, 3]
        current_alpha_expanded = current_alpha[:, :, np.newaxis]
        
        if color.shape[2] == 3:
            final_image[y_start:y_end, x_start:x_end, :3] += color * (1.0 - current_alpha_expanded)
        else:
            z_alpha = color[:, :, 3]
            z_alpha_expanded = z_alpha[:, :, np.newaxis]
            final_image[y_start:y_end, x_start:x_end, :3] += color[:, :, :3] * (1.0 - current_alpha_expanded)
            current_alpha = current_alpha + z_alpha * (1.0 - current_alpha)
            
        final_image[y_start:y_end, x_start:x_end, 3] = current_alpha + opacity * (1.0 - current_alpha)
        total_opacity[y_start:y_end, x_start:x_end] += opacity * (1.0 - total_opacity[y_start:y_end, x_start:x_end])
        
    final_image_uint8 = np.clip(final_image, 0, 1).astype(np.uint8)
    return final_image_uint8
```

**Edge Cases and Pitfalls:**
- **Precision:** Use `float32` for accumulation to avoid overflow and maintain dynamic range [S1, unofficial].
- **Boundaries:** Ensure sub-images are placed correctly without overlap or gaps [S2, unofficial].
- **Degenerate Input:** Handle cases where sub-images might be empty or have zero opacity [G].

## Subtopics

### [Tile-Based Alpha Compositing](1215-tile-based-alpha-compositing.md)
Handles compositing within a tile. Choose when memory is limited per tile.

### [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md)
Distributes tiles dynamically. Choose when workload is uneven.

### [Tile Boundary Computation Setup](2678-tile-boundary-computation-setup.md)
Computes tile boundaries. Choose when tile sizes vary.

### [Multi-Threaded Tile-Based Rendering](3784-multi-threaded-tile-based-rendering.md)
Uses threads for tile processing. Choose for CPU parallelism.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of pixels, dominated by the compositing step [S1, S2, unofficial].
- **Memory Complexity:** $O(X \cdot Y)$ for the final image [S1, unofficial].
- **Performance:** Depends on network bandwidth for gathering and CPU speed for compositing [S1, S2, unofficial].
- **Strengths:** Simple, deterministic, no synchronization needed during compositing [S1, S2, unofficial].
- **Limitations:** Requires gathering all data to one rank, which can be a bottleneck for large images [S1, S2, unofficial].
- **When to Use:** When a single final image is required [S1, unofficial].
- **When Not to Use:** When the image is too large to fit in memory or when distributed final image generation is needed [G].

## Background
Parallel Image Assembly is used in parallel volume rendering, often implemented using MPI for communication [S1, S2, unofficial]. It builds on the principles of tiled parallel compositing [Parent].

## Sources
[S1] Parallel-3D-Volume-Rendering-using-MPI, https://github.com/vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, fetched 2026-10-27 (unofficial)
[S2] Parallel-3D-Volume-Rendering-with-MPI-CS677, https://github.com/jhadivyanshu24/Parallel-3D-Volume-Rendering-with-MPI-CS677, fetched 2026-10-27 (unofficial)
