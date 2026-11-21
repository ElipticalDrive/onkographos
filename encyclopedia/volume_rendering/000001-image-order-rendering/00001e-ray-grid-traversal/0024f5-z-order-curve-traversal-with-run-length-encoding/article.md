# Z-order curve traversal with run-length encoding

Also called: Morton order traversal, Z-curve RLE stream

## Overview
This method traverses a uniform voxel grid in Z-order (Morton) sequence while compressing contiguous runs of identical voxel values into run-length encoded (RLE) records, thereby skipping redundant sampling steps [S1, unofficial]. It uses a flat, stack-based iteration over a padded power-of-two grid structure rather than ray-based advancement or hierarchical tree navigation [S1, unofficial].

## How it works
**Inputs:** A uniform voxel grid of dimensions $W \times H \times D$ [S1, unofficial].
**Outputs:** A stream of `IndexedLeaf` records, each containing a voxel value and the starting linear index of the run [S1, unofficial].

**Data Structures:**
The traversal state is maintained in a `ZStream` object and a stack of `ZFrame` objects [S1, unofficial].
1.  **ZFrame struct:**
    *   `size` (unsigned int): The side length of the current cube in the Z-order hierarchy [S1, unofficial].
    *   `offset` (3-component integer vector): Stores the x, y, z coordinates of the cube's origin [S1, unofficial].
    *   `i` (uint8_t): An iterator counter (0–7) tracking which of the 8 children of the current node has been visited [S1, unofficial].
2.  **ZStream class fields:**
    *   `model` (pointer to voxel data): Reference to the voxel data source [S1, unofficial].
    *   `index` (unsigned integer): The current linear index in the Z-order stream [S1, unofficial].
    *   `_open` (bool): Flag indicating if the stream has more data [S1, unofficial].
    *   `stream_leaf` (voxel value): The value of the most recently fetched voxel [S1, unofficial].
    *   `stack` (stack of ZFrame): The traversal stack [S1, unofficial].
    *   `power` (unsigned int): The exponent such that $2^{power}$ is the padded grid dimension [S1, unofficial].
    *   `stream_size` (unsigned integer): The total number of voxels in the padded grid ($2^{3 \cdot power}$) [S1, unofficial].

**Algorithm Steps:**
1.  Compute `power` as $\lceil \log_2(\max(W, H, D)) \rceil$ and `stream_size` as $2^{3 \cdot power}$ [S1, unofficial].
2.  Initialize the stack with a root `ZFrame`: `size` = $2^{power}$, `offset` = $(0,0,0)$, `i` = 0 [S1, unofficial].
3.  Initialize `index` = 0, `_open` = true, and fetch the first voxel using `next_coords()` to populate `stream_leaf` [S1, unofficial].
4.  Loop while `_open` is true: call `next_coords()` to get the next voxel position and fetch its value from `model` [S1, unofficial].
5.  If the fetched value equals `stream_leaf`, increment `index` and repeat the fetch until a different value is found or the stream ends [S1, unofficial].
6.  If the value differs from `stream_leaf` or the stream ends, return an `IndexedLeaf` containing the previous `stream_leaf` value and the `index` where that run started, then update `stream_leaf` to the new value and increment `index` [S1, unofficial].
7.  **Coordinate Generation (`next_coords`):** If `frame.size` == 1, return `frame.offset`, pop the frame, backtrack by popping frames where `top.i` >= 8, set `_open` = false if the stack becomes empty, and increment `index` [S1, unofficial]. Otherwise, compute `child_size` = `frame.size` / 2, determine the child offset by adding `child_size` to the x, y, or z axis if bits 4, 2, or 1 of `frame.i` are set, increment `frame.i`, push a new `ZFrame` with `size` = `child_size`, `offset` = `child_offset`, `i` = 0, and recurse [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def next_coords(stack, index_ref):
    """Generates the next coordinate in Z-order traversal using a stack.
    Returns the coordinate tuple (x, y, z). Modifies stack and index_ref."""
    while True:
        frame = stack[-1]
        
        # Base case: leaf node
        if frame['size'] == 1:
            coord = frame['offset']
            stack.pop()
            
            # Backtrack if all 8 children of the parent have been visited
            while len(stack) > 0 and stack[-1]['i'] >= 8:
                stack.pop()
            
            index_ref[0] += 1
            return coord
        
        # Recursive step: determine child
        child_size = frame['size'] // 2
        child_idx = frame['i']
        
        # Calculate child offset based on bits 4, 2, 1 of frame.i
        offset = list(frame['offset'])  # [x, y, z]
        if child_idx & 4:
            offset[2] += child_size  # z-axis
        if child_idx & 2:
            offset[1] += child_size  # y-axis
        if child_idx & 1:
            offset[0] += child_size  # x-axis
            
        frame['i'] += 1
        
        # Push new frame for the child
        new_frame = {
            'size': child_size,
            'offset': tuple(offset),
            'i': 0
        }
        stack.append(new_frame)
        
        # Continue to the next level (simulating recursion)
        # Loop back to process the new top of the stack

def traverse_rle(model, width, height, depth):
    """Main traversal function. Returns a list of (value, start_index) tuples."""
    power = math.ceil(math.log2(max(width, height, depth)))
    stream_size = 2 ** (3 * power)
    
    stack = [{'size': 2**power, 'offset': (0, 0, 0), 'i': 0}]
    index_ref = [0]  # Mutable container for index
    _open = True
    
    # Fetch first voxel
    first_coord = next_coords(stack, index_ref)
    prev_leaf = model.at(first_coord)
    
    results = []
    
    while _open:
        curr_coord = next_coords(stack, index_ref)
        curr_leaf = model.at(curr_coord)
        
        if curr_leaf != prev_leaf:
            results.append((prev_leaf, index_ref[0]))
            prev_leaf = curr_leaf
            
        # If stack is empty, _open becomes false inside next_coords
        if not stack:
            _open = False
            
    # Append the final run
    results.append((prev_leaf, index_ref[0]))
    
    return results
```

**Edge Cases and Pitfalls:**
*   **Non-Power-of-Two Dimensions:** The grid is padded to the next power of two with implicit empty voxels [S1, unofficial].
*   **Stack Depth:** The traversal uses a stack-based iterative approach to simulate recursion, and the stack depth is bounded by the 8-children-per-node limit [S1, unofficial].
*   **Run Compression:** If all voxels are identical, the RLE stream contains a single entry; if no runs exist, performance degrades to standard Z-order traversal overhead [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N = 2^{3 \cdot power}$ is the number of voxels in the padded grid [S1, unofficial].
*   **Memory Complexity:** $O(\log N)$ for the traversal stack [S1, unofficial].
*   **Performance:** Highly efficient for volumes with large homogeneous regions; performance degrades if the volume is noisy or lacks contiguous runs [S1, unofficial].
*   **Strengths:** Simple stack-based implementation, predictable cache-friendly traversal pattern, and effective compression for coherent data [S1, unofficial].
*   **Limitations:** Requires power-of-two padding which wastes memory for non-aligned dimensions, and is unsuitable for highly sparse data without significant overhead [S1, unofficial].

## Background
The method is derived from the Zazen project by Gabriel Müller, which implements a Z-order curve traversal with run-length encoding for volume rendering [S1, unofficial]. It leverages the spatial locality of Morton order to provide a predictable traversal pattern while compressing redundant data.

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
