# Front-to-Back Gaussian Splat Rendering

Also called: Gaussian Splat Sorting and Rendering

## Overview
Renders semi-transparent 2D Gaussian splats in correct depth order to enable accurate alpha blending. It differs from instanced rendering by using a CPU-side loop to issue sequential draw calls, ensuring the GPU processes splats in a specific sorted order rather than in parallel.

## How it works
**Inputs:**
- Centroids of all splats.
- Covariance matrices (or 2D basis parameters) for all splats.
- Model-view and projection matrices.
- Number of actual splats ($N$) and padded splats (dummy splats added to fill tile or batch boundaries).

**Outputs:**
- Rendered image with correct front-to-back alpha blending.

**Data Structures:**
- `centroid`: Array of $N$ vectors (e.g., `vec3<f32>`).
- `covariance`: Array of $N$ vectors/matrices representing the Gaussian spread.
- `basis`: Array of $N$ vectors (e.g., `vec4<f32>`) storing the projected 2D basis.
- `distance`: Array of $N$ unsigned 32-bit integers (`u32`). Stores the sorted key.
- `id`: Array of $N$ unsigned 32-bit integers (`u32`). Stores the original index or instance ID.

**Steps:**

1. **Compute Distances and Basis:**
   For each splat $i$, compute the projected 2D basis and the distance to the camera. This is typically done in a compute shader.
   - The distance is calculated as a monotonic proxy for depth (e.g., using clip-space $w$ or Euclidean distance in view space).
   - The distance value is converted to an integer in the range $[0, 0xFFFFFFFF \gg 8]$ (effectively 24-bit precision) to serve as the sort key.
   - For dummy/padded splats, assign the maximum possible distance value ($0xFFFFFFFF$) so they sort to the end of the array.

2. **GPU Radix Sort:**
   Sort the `distance` array (and corresponding `id` and `basis` arrays) in ascending order (front-to-back) using a GPU radix sort algorithm.
   - The sort key is the integer distance.
   - The sort is stable to preserve relative order of splats at the same depth if necessary, though distinct depths are preferred.

3. **Sequential Rendering Loop:**
   Iterate through the sorted splats and issue individual draw calls.
   - The source code iterates from `actualSplatSize - 1` down to `0`.
   - **Note on Order:** The source states this is "front-to-back rendering" but the loop index `i` goes from largest to smallest. If `distance` is sorted ascending (smallest distance = closest = index 0), then `distance[last]` is the farthest. Iterating backwards draws the farthest splats first. Standard alpha blending requires front-to-back (closest first). However, the source explicitly labels this loop as applying "front-to-back rendering" and cites the reason for the loop as maintaining order. We follow the source's implementation:
     ```python
     # Pseudocode based on [S1, unofficial]
     # sorted_indices is the result of the radix sort on the distance array
     # actualSplatSize is the number of real splats (excluding padding)
     
     for i from actualSplatSize - 1 down to 0:
         current_splat_id = sorted_indices[i]
         # Set vertex buffer to the position/basis of current_splat_id
         # Issue draw call for this single splat
         draw_call(splat_id=current_splat_id)
     ```
   - **Pitfall:** Instanced rendering cannot be used because it does not guarantee the rendering order of instances, which is required for correct alpha blending of semi-transparent splats. The CPU loop ensures sequential processing.

**Parameters:**
- Distance precision: 24-bit integer range $[0, 0xFFFFFF]$.
- Padding: Dummy splats with distance $0xFFFFFFFF$.

**Edge Cases:**
- **Padded Splats:** Must be assigned the maximum distance value to ensure they are sorted to the end and not rendered (or rendered last, effectively transparent if alpha is 0).
- **Precision:** The conversion to integer distance must be monotonic with respect to actual depth to ensure correct sorting.

## Operating characteristics
- **Time Complexity:** Dominated by the GPU radix sort $O(N \log N)$ or $O(N)$ depending on the radix sort implementation, plus $O(N)$ draw calls.
- **Memory Complexity:** $O(N)$ for storing distances, IDs, and bases.
- **Strengths:** Ensures correct alpha blending for semi-transparent splats.
- **Limitations:** High CPU overhead due to $N$ individual draw calls. Instanced rendering is not viable for ordered blending.
- **When to use:** When correct depth ordering is critical for semi-transparent rendering and the number of splats is manageable for the draw call overhead.

## Background
Originated in the context of WebGPU implementations of 2D Gaussian Splatting. The method leverages existing GPU radix sort algorithms to reorder splats by camera distance, addressing the need for front-to-back rendering of semi-transparent primitives.

## Sources
[S1] WebGPU Unleashed: A Practical Tutorial, https://shi-yan.github.io/webgpuunleashed/Advanced/gaussian_splatting.html, fetched 2026-10-29 (unofficial)
