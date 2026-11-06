# Z-order curve traversal with run-length encoding

Also called: Morton order traversal, Z-curve RLE stream

## Overview
This method traverses a uniform voxel grid in Z-order (Morton) sequence while compressing contiguous runs of identical voxel values into a single run-length encoded (RLE) record, thereby skipping redundant sampling steps [S1, unofficial]. It differs from standard DDA [3061-digital-differential-analyzer-dda-traversal.md] by following a space-filling curve rather than a ray, and from hierarchical methods [851-hierarchical-trilinear-ray-casting.md] by using a flat, stack-based iteration over a padded power-of-two grid structure. It is chosen when the volume data exhibits high spatial coherence (long runs of identical values) and the grid dimensions are suitable for padding to a power of two.

## How it works
**Inputs:** A uniform voxel grid of dimensions $W \times H \times D$.
**Outputs:** A stream of `IndexedLeaf` records, each containing a voxel value and the starting linear index of the run.

**Data Structures:**
The traversal state is maintained in a `ZStream` object and a stack of `ZFrame` objects [S1, unofficial].

1.  **ZFrame struct:**
    *   `size` (unsigned int): The side length of the current cube in the Z-order hierarchy.
    *   `offset` (int3): A 3-component vector storing the x, y, z coordinates of the cube's origin. Each component is an unsigned integer packed into 3 bits (implied by "int3: x,y,z unsigned int" and the context of Morton ordering, though standard implementation uses full integers; we treat them as integer coordinates) [S1, unofficial].
    *   `i` (uint8_t): An iterator counter (0–7) tracking which of the 8 children of the current node has been visited.

2.  **ZStream class fields:**
    *   `model` (const Model*): Pointer to the voxel data source.
    *   `index` (unsigned long long int): The current linear index in the Z-order stream.
    *   `_open` (bool): Flag indicating if the stream has more data.
    *   `stream_leaf` (Leaf): The value of the most recently fetched voxel.
    *   `stack` (std::stack<ZFrame>): The traversal stack.
    *   `power` (unsigned int): The exponent such that $2^{power}$ is the padded grid dimension.
    *   `stream_size` (unsigned long long int): The total number of voxels in the padded grid ($2^{3 \cdot power}$).

**Algorithm Steps:**

1.  **Initialization:**
    *   Compute `power` as $\lceil \log_2(\max(W, H, D)) \rceil$ [S1, unofficial].
    *   Compute `stream_size` as $2^{3 \cdot power}$ [S1, unofficial].
    *   Initialize the stack with a root `ZFrame`: `size` = $2^{power}$, `offset` = $(0,0,0)$, `i` = 0 [S1, unofficial].
    *   Initialize `index` = 0, `_open` = true.
    *   Fetch the first voxel using `next_coords()` and store in `stream_leaf` [S1, unofficial].

2.  **Traversal Loop (Fetching Next Value):**
    While `_open` is true:
    1.  Call `next_coords()` to get the next voxel position in Z-order.
    2.  Fetch the voxel value `v` from `model` at `next_coords()` [S1, unofficial].
    3.  If `v` equals `stream_leaf` (the previous value):
        *   Increment `index`.
        *   Repeat step 2 (skip to next coordinate) until a different value is found or the stream ends [S1, unofficial].
    4.  If `v` differs from `stream_leaf` or the stream ends:
        *   Return an `IndexedLeaf` containing the previous `stream_leaf` value and the `index` at which that run started.
        *   Update `stream_leaf` = `v`.
        *   Increment `index`.

3.  **Coordinate Generation (`next_coords`):**
    This function simulates a recursive Z-order traversal iteratively using the stack [S1, unofficial].
    1.  Let `frame` be `stack.top()`.
    2.  If `frame.size` == 1:
        *   Return `frame.offset` [S1, unofficial].
        *   Pop `frame` from the stack.
        *   While the stack is not empty and `stack.top().i` >= 8:
            *   Pop `stack.top()` (backtrack to parent) [S1, unofficial].
        *   If the stack is empty, set `_open` = false.
    3.  If `frame.size` > 1:
        *   Compute `child_size` = `frame.size` / 2.
        *   Determine the child index `c` based on `frame.i` (0–7).
        *   Compute `child_offset` by adding `child_size` to the respective axis of `frame.offset` if the corresponding bit of `c` is set (bit 0 for x, bit 1 for y, bit 2 for z, corresponding to bits 1, 2, 4 of the Morton code) [S1, unofficial].
        *   Increment `frame.i`.
        *   Push a new `ZFrame` onto the stack: `size` = `child_size`, `offset` = `child_offset`, `i` = 0.
        *   Recursively call `next_coords()` (or loop back to step 1 with the new top) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def next_coords(stack, power):
    """
    Generates the next coordinate in Z-order traversal using a stack.
    Returns the coordinate tuple (x, y, z).
    Modifies the stack and the global _open flag.
    """
    while True:
        frame = stack[-1]
        
        # Base case: leaf node
        if frame['size'] == 1:
            coord = frame['offset']
            stack.pop()
            
            # Backtrack if all 8 children of the parent have been visited
            while len(stack) > 0 and stack[-1]['i'] >= 8:
                stack.pop()
            
            if len(stack) == 0:
                global _open
                _open = False
            
            return coord
        
        # Recursive step: determine child
        child_size = frame['size'] // 2
        child_idx = frame['i']
        
        # Calculate child offset based on Morton order bits
        # Bit 0 (1) -> x, Bit 1 (2) -> y, Bit 2 (4) -> z
        offset = list(frame['offset']) # [x, y, z]
        if child_idx & 1:
            offset[0] += child_size
        if child_idx & 2:
            offset[1] += child_size
        if child_idx & 4:
            offset[2] += child_size
            
        frame['i'] += 1
        
        # Push new frame for the child
        new_frame = {
            'size': child_size,
            'offset': tuple(offset),
            'i': 0
        }
        stack.append(new_frame)
        
        # Continue to the next level (simulating recursion)
        # In a real implementation, this would be a recursive call or loop
        # Here we loop back to process the new top
```

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_rle(model, width, height, depth):
    """
    Main traversal function.
    Returns a list of (value, start_index) tuples.
    """
    power = math.ceil(math.log2(max(width, height, depth)))
    stream_size = 2 ** (3 * power)
    
    stack = [{'size': 2**power, 'offset': (0,0,0), 'i': 0}]
    index = 0
    _open = True
    
    # Fetch first voxel
    first_coord = next_coords(stack, power)
    prev_leaf = model.at(first_coord)
    
    results = []
    
    while _open:
        curr_coord = next_coords(stack, power)
        curr_leaf = model.at(curr_coord)
        
        if curr_leaf != prev_leaf:
            results.append((prev_leaf, index))
            prev_leaf = curr_leaf
        
        index += 1
        
    # Append the last run
    results.append((prev_leaf, index))
    
    return results
```

**Edge Cases and Pitfalls:**
*   **Non-Power-of-Two Dimensions:** The grid is padded to the next power of two. Implicit empty voxels are added. The traversal must ensure these padded voxels are handled correctly (typically treated as empty or skipped) [S1, unofficial].
*   **Stack Depth:** The maximum stack depth is $3 \cdot power$. For large grids, this can be significant, but it is bounded [S1, unofficial].
*   **Run Compression:** If all voxels are identical, the RLE stream will contain a single entry. If no runs exist (all voxels differ), performance degrades to standard Z-order traversal overhead [S1, unofficial].

## Subtopics
*   **Hierarchical Trilinear Ray Casting [851-hierarchical-trilinear-ray-casting.md]:** Uses a hierarchical grid structure for adaptive sampling. Choose when data has varying density. Z-order RLE is simpler but requires uniform padding.
*   **Hierarchical Digital Differential Analyzer (HDDA) [1122-hierarchical-digital-differential-analyzer-hdda.md]:** Combines DDA with hierarchical acceleration. Choose when ray coherence is high and grid is sparse. Z-order RLE is ray-independent.
*   **Sparse Voxel Octree Ray Traversal [2451-sparse-voxel-octree-ray-traversal.md]:** Uses an octree for sparse data. Choose when data is highly sparse. Z-order RLE requires dense padding.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N = 2^{3 \cdot power}$ is the number of voxels in the padded grid. The RLE compression reduces the number of output records but not the traversal cost.
*   **Memory Complexity:** $O(\log N)$ for the stack. The RLE stream size is at most $N$.
*   **Performance:** Highly efficient for volumes with large homogeneous regions. Performance degrades if the volume is noisy or has no runs.
*   **Strengths:** Simple implementation, no ray-specific logic, effective compression for coherent data.
*   **Limitations:** Requires power-of-two padding, which can waste memory for non-aligned dimensions. Not suitable for sparse data without significant overhead.

## Background
The method is derived from the Zazen project by Gabriel Müller, which implements a Z-order curve traversal with run-length encoding for volume rendering [S1, unofficial]. It leverages the properties of Morton order to provide a predictable, cache-friendly traversal pattern while compressing redundant data.

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
