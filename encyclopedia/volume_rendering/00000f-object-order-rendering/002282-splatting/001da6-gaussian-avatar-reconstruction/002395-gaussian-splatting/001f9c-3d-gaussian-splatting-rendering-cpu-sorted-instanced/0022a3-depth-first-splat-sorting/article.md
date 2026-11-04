# Depth-First Splat Sorting

Also called: Pre-fragment depth sort, Sorting visible splats by depth

## Overview
Depth-First Splat Sorting is a preprocessing step for 3D Gaussian Splatting rendering that sorts visible Gaussian splats by camera-space depth before they are expanded into per-tile fragments. It solves the problem of high sorting overhead inherent in the original 3DGS pipeline, where fragments (one per tile a splat touches) are sorted using a combined 48-bit key. By sorting the smaller set of visible splats first, this method reduces the number of items to sort by a factor of 1.5 to 4, significantly lowering computational cost while maintaining correct depth ordering for subsequent rasterization.

## How it works
The method operates on a list of "staged splats" (visible 3D Gaussians projected into screen space). The goal is to produce a list of tile fragments that preserves the depth order of the original splats, allowing for efficient grouping by tile ID later.

**Inputs:**
- `splat_list`: An array of $N$ visible splats. Each splat $i$ contains:
  - `depth`: Camera-space depth value ($z_i$).
  - `tile_ids`: A list of tile IDs touched by the splat.
  - `tile_counts`: The number of tiles touched ($k_i = \text{len}(tile\_ids)$).
  - `index`: The original index in the unsorted list (used as the sort value).

**Outputs:**
- `fragment_list`: An array of tile fragments. The total size is $\sum_{i=0}^{N-1} k_i$. Fragments are stored such that fragments from splat $i$ occupy a contiguous block in the array, and these blocks are ordered by the depth of splat $i$.

**Data Structures:**
- **Splat Item**: Contains `depth` (float), `tile_ids` (array of integers), `tile_count` (integer).
- **Fragment Item**: Contains `tile_id` (integer), `splat_index` (integer, referencing the original splat), and potentially other attributes copied from the splat (position, covariance, color). The exact layout depends on the downstream rasterizer but must include `tile_id` for grouping.

**Steps:**

1. **Sort Splats by Depth**:
   Perform a stable radix sort on the `splat_list` using `depth` as the key and `index` as the value.
   - Key: 32-bit depth value.
   - Value: 32-bit splat index.
   - This produces a permutation of splats ordered from nearest to farthest.

2. **Compute Exclusive Prefix Sum**:
   Calculate the exclusive prefix sum of the `tile_counts` from the sorted splats to determine storage offsets for fragments.
   Let $k_i$ be the number of tiles touched by the $i$-th splat in the sorted list.
   Let $O_i$ be the offset for the $i$-th splat's fragments.
   $$ O_0 = 0 $$
   $$ O_i = \sum_{j=0}^{i-1} k_j \quad \text{for } i > 0 $$
   This can be computed in a single pass:
   ```python
   offsets = []
   current_offset = 0
   for splat in sorted_splat_list:
       offsets.append(current_offset)
       current_offset += splat.tile_count
   ```

3. **Create Fragments**:
   Iterate through the sorted splats and their corresponding offsets. For each splat $i$:
   - Let $O_i$ be the computed offset.
   - For each tile ID $t$ in `splat.tile_ids`:
     - Create a fragment object.
     - Set `fragment.tile_id = t`.
     - Set `fragment.splat_index = splat.index`.
     - Copy necessary attributes from the splat to the fragment.
     - Store the fragment at `fragment_list[O_i + j]`, where $j$ is the index of the tile ID in `splat.tile_ids`.

**Edge Cases and Pitfalls:**
- **Precision**: Depth values must be compared consistently. Using 32-bit floats is standard, but ensure the radix sort handles the bit representation correctly for depth comparison.
- **Empty Splats**: If a splat touches no tiles (e.g., outside the screen), it should be filtered out before sorting or handled with a tile count of 0.
- **Stability**: The sort must be stable to preserve the relative order of splats with identical depths, although this is rare in practice. The prefix sum approach relies on the sorted order being preserved during fragment creation.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_first_splat_sort(splat_list):
    """
    Sorts splats by depth and creates depth-ordered tile fragments.
    
    Args:
        splat_list: List of dicts with keys 'depth', 'tile_ids', 'index'.
        
    Returns:
        fragment_list: List of fragment dicts.
    """
    # Step 1: Sort splats by depth
    # Using a stable sort to preserve original order for equal depths
    sorted_splats = sorted(splat_list, key=lambda s: s['depth'])
    
    # Step 2: Compute exclusive prefix sum of tile counts
    offsets = []
    current_offset = 0
    for splat in sorted_splats:
        offsets.append(current_offset)
        current_offset += len(splat['tile_ids'])
        
    total_fragments = current_offset
    fragment_list = [None] * total_fragments
    
    # Step 3: Create fragments
    for i, splat in enumerate(sorted_splats):
        offset = offsets[i]
        tile_ids = splat['tile_ids']
        splat_index = splat['index']
        
        for j, tile_id in enumerate(tile_ids):
            frag_index = offset + j
            fragment_list[frag_index] = {
                'tile_id': tile_id,
                'splat_index': splat_index,
                # Copy other necessary attributes from splat here
                'depth': splat['depth']
            }
            
    return fragment_list
```

## Subtopics

### Exclusive Prefix Sum Fragment Creation
This variant emphasizes the use of an exclusive prefix sum to determine fragment storage offsets. It ensures that fragments from each splat are stored contiguously in the output array, inheriting the depth-sorted order of the splats. This allows for a subsequent stable sort by tile ID to group fragments per tile without breaking the depth order within each tile. The key advantage is that the initial depth sort is performed on $N$ splats rather than $M$ fragments ($M > N$), reducing sorting complexity.

## Operating characteristics
- **Time Complexity**: Sorting $N$ splats takes $O(N \log N)$ or $O(N)$ with radix sort. Prefix sum and fragment creation take $O(M)$, where $M$ is the total number of fragments. Since $M \approx 1.5-4N$, the dominant cost is the initial sort on $N$ items, which is significantly cheaper than sorting $M$ fragments.
- **Memory Complexity**: Requires $O(N)$ for the sorted splats and $O(M)$ for the fragment list. The prefix sum array requires $O(N)$.
- **Performance**: Reduces sorting overhead by a factor of 1.5 to 4 compared to the original 3DGS approach. This is particularly beneficial for scenes with high overlap where splats touch many tiles.
- **Strengths**: Lower sorting cost, simpler key generation (32-bit vs 48-bit), easier to implement on CPU.
- **Limitations**: Requires two passes (sort splats, then create fragments) compared to the original single sort of fragments. The fragment list is not yet grouped by tile, requiring a subsequent stable sort by tile ID.

## Background
This method was introduced in the Splatshop paper [S1] as an optimization for editing large Gaussian Splat models. It addresses the inefficiency of the original 3DGS rendering pipeline, which sorts fragments after splitting splats into tiles. By sorting splats first, it reduces the number of items to sort, leveraging the fact that the number of fragments is typically 1.5 to 4 times the number of visible splats.

## Sources
[S1] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
