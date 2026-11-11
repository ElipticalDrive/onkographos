# Fast Splat Sorting

Also called: Depth and TileID Sorting, Split Radix Sort for Splatting, Separating Depth and Tile Sorting, Splat Sorting Dispatch

## Overview
Fast Splat Sorting is an optimization for volume rendering pipelines that replaces the single 48-bit combined tileID+depth radix sort with two sequential, smaller-key sorts: a 32-bit depth sort of visible splats followed by a 16-bit tileID sort of generated fragments [S2]. It sets itself apart from its parent [Radix Sort for Splat Depth](1207-radix-sort-for-splat-depth.md) by decoupling depth ordering from tile assignment, and from [Gaussian Sorting](2845-gaussian-splat-sorting.md) by relying on explicit fragment duplication rather than implicit spatial partitioning [S2]. This separation reduces computational complexity by leveraging the fact that the number of visible splats ($n$) is typically smaller than the number of fragments ($m$) [S2].

## How it works
The method replaces the standard 48-bit radix sort (used in 3DGS) which sorts fragments by a combined key of tileID and depth. Instead, it performs two distinct sorting passes.

**Inputs and Outputs:**
*   **Input:** A set of $n$ visible splats, each with a 32-bit depth value, plus camera parameters and sorting flags [S1, unofficial][S2].
*   **Intermediate:** A set of $m$ fragments, where each fragment corresponds to a splat intersecting a specific tile. Each fragment carries a 16-bit tileID [S2].
*   **Output:** An `ordering` array of indices, sorted such that splats are ordered by depth within each tile, and tiles are processed in raster order [S1, unofficial][S2].

**Data Structures:**
1.  **Splat Data:** Each visible splat $i$ has a depth value $d_i$ (stored as `f32`) [S2].
2.  **Fragment Data:** Each fragment $j$ has a tileID $t_j$ (stored as `u16`) [S2].
3.  **Ordering Array:** `Uint32Array` of size $m$, storing indices to the fragment list [S1, unofficial].

**Steps:**
1.  **Depth Sort (Visible Splats):**
    *   Collect all $n$ visible splats.
    *   Perform a radix sort on the 32-bit depth values [S2].
    *   This step produces a list of splats sorted by depth [S2].
2.  **Fragment Generation:**
    *   For each splat in the depth-sorted list, determine which tiles it intersects [S2].
    *   Generate a fragment for each intersected tile. Each fragment inherits the splat's depth and is assigned the tile's 16-bit ID [S2].
    *   *Note:* The sources mention using tighter AABBs and approximate tile-ellipse intersection tests to reduce false-positive fragments, but the sorting logic itself assumes the fragments are already generated [S2].
3.  **TileID Sort (Fragments):**
    *   Perform a radix sort on the $m$ fragments using their 16-bit tileID as the key [S2].
    *   Because the splats were already sorted by depth in Step 1, and radix sort is stable, the resulting list is sorted primarily by tileID (raster order) and secondarily by depth (front-to-back) [S2].

**Formulas:**
The total complexity is split as follows:
$$ T_{total} = T_{32\text{-bit}}(n) + T_{16\text{-bit}}(m) $$
Where $T_{48\text{-bit}}(m) \approx 3 \times T_{16\text{-bit}}(m)$ assuming linear scaling of radix sort runtime with the number of bits [S2].

**Parameters:**
*   `num_splats` ($u32$): Total number of splats to process [S1, unofficial].
*   `camera_x, camera_y, camera_z` ($f64$): Camera position coordinates [S1, unofficial].
*   `direction_x, direction_y, direction_z` ($f32$): Camera viewing direction vector [S1, unofficial].
*   `radial` ($bool$): Enables radial sorting mode [S1, unofficial].
*   `fast_sort` ($bool$): Enables fast sorting mode [S1, unofficial].
*   `front_sort` ($bool$): Selects between 16-bit front sort and 32-bit sort [S1, unofficial].

**Pseudocode:**
The following pseudocode implements the dispatch logic described in the sources [S1, unofficial][S2]. It includes validation checks and inline stable radix sort implementations to avoid external dependencies [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_splat_sort(buffers, max_splats, num_splats, camera, direction, 
                    radial, fast_sort, front_sort):
    # Validate input array lengths and bounds [S1, unofficial]
    if not (len(buffers.range_mesh_ids) == len(buffers.range_bases) == len(buffers.range_counts)):
        raise ValueError("range_mesh_ids, range_bases, and range_counts must have equal lengths")
    if not all(0 <= idx < buffers.range_counts[i] for i, idx in enumerate(buffers.center_update_range_indices)):
        raise ValueError("center_update_range_indices out of bounds")
    if sum(buffers.range_counts) * 3 != len(buffers.update_centers):
        raise ValueError("update_centers length mismatch")
    if not all(0 <= idx < buffers.range_counts[i] for i, idx in enumerate(buffers.matrix_update_range_indices)):
        raise ValueError("matrix_update_range_indices out of bounds")
    if len(buffers.update_matrices) != len(buffers.matrix_update_range_indices) * 16:
        raise ValueError("update_matrices length mismatch")

    active_splats = 0
    if front_sort:
        active_splats = sort16_centers_front_internal(buffers, max_splats, num_splats, camera, direction)
    else:
        active_splats = sort32_centers_internal(buffers, max_splats, num_splats, camera, direction, radial, fast_sort)

    if active_splats > 0:
        buffers.ordering[0:active_splats] = buffers.ordering[:active_splats]

    return active_splats

def sort16_centers_front_internal(buffers, max_splats, num_splats, camera, direction):
    # Performs 16-bit radix sort on fragment tileIDs [S2]
    return stable_radix_sort_16(buffers.fragment_tile_ids, buffers.fragment_indices, num_splats)

def sort32_centers_internal(buffers, max_splats, num_splats, camera, direction, radial, fast_sort):
    # Performs 32-bit radix sort on visible splat depths [S2]
    return stable_radix_sort_32(buffers.splat_depths, buffers.splat_indices, num_splats)

def stable_radix_sort_16(keys, indices, n):
    # Standard 16-bit stable radix sort (counting sort per 8-bit digit) [G]
    for digit in [0, 1]:
        bucket = [[] for _ in range(256)]
        for i in range(n):
            bucket[(keys[i] >> (digit * 8)) & 0xFF].append(i)
        idx = 0
        for b in bucket:
            for i in b:
                indices[idx] = i
                idx += 1
    return n

def stable_radix_sort_32(keys, indices, n):
    # Standard 32-bit stable radix sort (counting sort per 8-bit digit) [G]
    for digit in [0, 1, 2, 3]:
        bucket = [[] for _ in range(256)]
        for i in range(n):
            bucket[(keys[i] >> (digit * 8)) & 0xFF].append(i)
        idx = 0
        for b in bucket:
            for i in b:
                indices[idx] = i
                idx += 1
    return n
```

**Edge Cases and Pitfalls:**
*   **Fragment Count:** The number of fragments $m$ can be significantly larger than $n$. The 16-bit sort must handle this larger dataset efficiently [S2].
*   **False Positives:** Without tight bounding boxes, the fragment generation step may create unnecessary fragments, increasing $m$ and slowing down the 16-bit sort. The sources recommend using tight AABBs and approximate tile-ellipse intersection tests to mitigate this [S2].
*   **Validation:** The implementation must validate that range arrays and update arrays have consistent lengths and bounds to prevent out-of-bounds memory access [S1, unofficial].
*   **Sort Stability:** The algorithm relies on stable radix sort to preserve the depth ordering established in Step 1 when sorting by tileID in Step 2 [S2].

## Subtopics

### [Two-Stage Sorting](9581-two-stage-sorting.md)
A variant that explicitly splits the generation of depth-sorted per-tile splat lists into two separate radix sort routines. It sets itself apart by decoupling depth ordering from tile assignment, allowing the use of smaller key sizes (32-bit and 16-bit) instead of a single large key.

## Operating characteristics
*   **Time Complexity:** The total sorting time is the sum of a 32-bit sort on $n$ splats and a 16-bit sort on $m$ fragments. Assuming linear scaling with bit-width, the 48-bit sort is approximately three times as expensive as the 16-bit sort [S2].
*   **Measured Performance:**
    *   **Berlin Scene:** 0.24 ms total (0.11 ms depth + 0.13 ms tileID) vs. 0.39 ms for 48-bit sort [S2].
    *   **Campus Scene:** 1.00 ms total (0.38 ms depth + 0.62 ms tileID) vs. 1.86 ms for 48-bit sort [S2].
*   **Scaling:** Performance improves significantly when the majority of splats contribute to the rendered image across multiple tiles, as the depth sort operates on a smaller set of visible splats [S2].
*   **Strengths:** Reduced computational complexity and better GPU utilization by avoiding large 48-bit sorts [S2].
*   **Limitations:** Requires an additional fragment generation step. Performance depends on the ratio of fragments to splats ($m/n$). If $m$ is very large, the 16-bit sort may become a bottleneck [S2].
*   **When to Use:** Use when rendering large Gaussian Splat models where the 48-bit sort is a bottleneck and the number of visible splats is significantly smaller than the number of fragments [S2].
*   **When Not to Use:** Avoid if the fragment generation step is already expensive or if the scene has a very high ratio of fragments to splats [S2].

## Background
Fast Splat Sorting was introduced in the "Splatshop" paper (2025) as part of an efficient CUDA-based splat editing system [S2]. It was developed to address the performance limitations of the original 3DGS rendering pipeline, which used a combined 48-bit tileID+depth radix sort [S2]. The method leverages the observation that sorting visible splats by depth first, and then sorting the resulting fragments by tileID, is more efficient than sorting all fragments by a combined key [S2].

## Sources
[S1] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
