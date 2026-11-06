# Per-Pixel Linked List Voxelization

Also called: A-Buffer Ray Intersection List, ABuffer Voxel Output Representation & Traversal, Atomic Counter Voxelization, Linked List Voxel Output, Shader Storage Buffer Object Linked List Voxelization

## Overview
Per-Pixel Linked List Voxelization is a GPU-based technique that constructs a depth-sorted linked list of fragment depths per screen pixel using atomic counters and shader storage buffers, allowing for subsequent sorting and processing of depth-sorted voxels [S1, unofficial]. It solves the problem of accumulating multiple surface intersections (depths) per pixel into a coherent, sorted structure suitable for volume rendering or ray marching, distinguishing itself from rasterization-based methods by explicitly handling multiple hits per pixel via dynamic linked lists rather than fixed-depth buffers or simple occupancy masks [S1, unofficial].

## How it works
The method operates in two primary shader stages: list construction (`MakeFragList`) and list sorting (`SortFragList`). It relies on Shader Storage Buffer Objects (SSBOs) to manage the linked list nodes and head pointers.

### Inputs and Outputs
*   **Inputs**:
    *   Mesh geometry (vertices and indices).
    *   Model transformation matrix.
    *   Framebuffer width (uniform).
    *   Initial head pointers (initialized to -1).
    *   Atomic counter (initialized to 0).
*   **Outputs**:
    *   A sorted linked list of fragment depths per pixel, stored in `NodeData`.
    *   The linked list structure is defined by `HeadPointers` (start of list per pixel) and `NodePointers` (next node index).

### Data Structures
The implementation uses three SSBOs bound to specific bindings:

1.  **HeadPointers** (Binding 1, `std430`):
    *   Array of `int`.
    *   Size: `width * height`.
    *   Content: Index of the first node in the linked list for each pixel. Initialized to -1.
2.  **NodePointers** (Binding 2, `std430`):
    *   Array of `int`.
    *   Size: Maximum number of fragments (or a sufficiently large upper bound).
    *   Content: Index of the next node in the linked list.
3.  **NodeData** (Binding 3, `std430`):
    *   Array of `int`.
    *   Size: Maximum number of fragments.
    *   Content: Depth value, scaled by `framebufferWidth` (i.e., `int(depth * framebufferWidth)`).

The output structure returned to the CPU is:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Voxelizer_ABufferLinkedList {
    int width;
    int height;
    int depth;
    int fragmentCount;
    int* headPointers;
    int* nodePointers;
    int* nodeData;
};
```

### Numbered Steps

#### Phase 1: List Construction (`MakeFragList`)
Executed during rasterization of the mesh.

1.  **Increment Counter**: Atomically increment a global atomic counter to obtain a unique index `count` for the current fragment.
    $$ count = \text{atomicCounterIncrement}(counter) $$
2.  **Calculate Head Index**: Compute the index into the `HeadPointers` array based on fragment screen coordinates.
    $$ headPointerIndex = \lfloor gl\_FragCoord.x \rfloor + \lfloor gl\_FragCoord.y \rfloor \times framebufferWidth $$
3.  **Atomic Link**: Atomically exchange the current head pointer value with the new fragment index. The previous head pointer value (`prevPtr`) becomes the next pointer for the new node.
    $$ prevPtr = \text{atomicExchange}(HeadPointers[headPointerIndex], count) $$
4.  **Set Next Pointer**: Store `prevPtr` in the `NodePointers` array at index `count`.
    $$ NodePointers[count] = prevPtr $$
5.  **Store Depth**: Store the fragment depth, scaled by the framebuffer width, in the `NodeData` array at index `count`.
    $$ NodeData[count] = \lfloor gl\_FragCoord.z \times framebufferWidth \rfloor $$

#### Phase 2: List Sorting (`SortFragList`)
Executed as a full-screen pass (e.g., drawing a screen quad) to sort the lists. This implementation uses a selection sort variant.

1.  **Iterate Pixels**: Loop over all screen pixels $(i, j)$ where $0 \le i < width$ and $0 \le j < height$.
2.  **Get Head Pointer**: Calculate `headPointerIndex` as in Phase 1 and retrieve `headPtr = HeadPointers[headPointerIndex]`.
3.  **Check Empty**: If `headPtr == -1`, discard the pixel (no fragments).
4.  **Initialize Outer Loop**: Set `outerPtr = headPtr`.
5.  **Outer Loop**: Iterate while `outerPtr != -1` and a safety counter (e.g., < 1024) is not exceeded.
    a.  **Find Minimum**:
        i.   Set `smallestPtr = outerPtr` and `smallestData = NodeData[smallestPtr]`.
        ii.  Set `innerPtr = NodePointers[outerPtr]`.
        iii. **Inner Loop**: Iterate while `innerPtr != -1` and a safety counter (e.g., < 1024) is not exceeded.
            *   If `NodeData[innerPtr] < smallestData`:
                *   `smallestPtr = innerPtr`
                *   `smallestData = NodeData[smallestPtr]`
            *   `innerPtr = NodePointers[innerPtr]`
    b.  **Swap Data**: If `smallestPtr != outerPtr`:
        *   `temp = NodeData[outerPtr]`
        *   `NodeData[outerPtr] = NodeData[smallestPtr]`
        *   `NodeData[smallestPtr] = temp`
    c.  **Advance Outer**: `outerPtr = NodePointers[outerPtr]`

#### Phase 3: Traversal
After sorting, the lists are depth-sorted (ascending). To extract voxels for a ray at pixel $(i, j)$:

1.  **Get Start**: `curPtr = HeadPointers[i + j * width]`.
2.  **Iterate**: While `curPtr != -1`:
    a.  The interval `[startVoxel, endVoxel)` represents contiguous filled voxels along the ray.
    b.  Process the voxel defined by `NodeData[curPtr]`.
    c.  Advance: `curPtr = NodePointers[curPtr]`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Phase 1: MakeFragList (Fragment Shader)
# Inputs: gl_FragCoord, framebufferWidth, counter (atomic), HeadPointers, NodePointers, NodeData
def make_frag_list():
    # 1. Get unique index
    count = atomic_counter_increment(counter)
    
    # 2. Calculate head pointer index
    head_pointer_index = int(gl_FragCoord.x) + int(gl_FragCoord.y) * framebufferWidth
    
    # 3. Atomic exchange to link
    prev_ptr = atomic_exchange(HeadPointers[head_pointer_index], count)
    
    # 4. Set next pointer
    NodePointers[count] = prev_ptr
    
    # 5. Store scaled depth
    NodeData[count] = int(gl_FragCoord.z * framebufferWidth)

# Phase 2: SortFragList (Fragment Shader, Full Screen Pass)
# Inputs: gl_FragCoord, framebufferWidth, HeadPointers, NodePointers, NodeData
def sort_frag_list():
    head_pointer_index = int(gl_FragCoord.x) + int(gl_FragCoord.y) * framebufferWidth
    head_ptr = HeadPointers[head_pointer_index]
    
    # Handle empty lists
    if head_ptr == -1:
        discard
    
    outer_ptr = head_ptr
    outer_loop_count = 0
    
    # Outer loop for selection sort
    while outer_ptr != -1 and outer_loop_count < 1024:
        smallest_ptr = outer_ptr
        smallest_data = NodeData[smallest_ptr]
        
        inner_ptr = NodePointers[outer_ptr]
        inner_loop_count = 0
        
        # Inner loop to find minimum
        while inner_ptr != -1 and inner_loop_count < 1024:
            if NodeData[inner_ptr] < smallest_data:
                smallest_ptr = inner_ptr
                smallest_data = NodeData[smallest_ptr]
            inner_ptr = NodePointers[inner_ptr]
            inner_loop_count += 1
            
        # Swap data if needed
        if smallest_ptr != outer_ptr:
            temp = NodeData[outer_ptr]
            NodeData[outer_ptr] = NodeData[smallest_ptr]
            NodeData[smallest_ptr] = temp
            
        outer_ptr = NodePointers[outer_ptr]
        outer_loop_count += 1

# Phase 3: Traversal (CPU or Ray Marching Shader)
def traverse_pixel(pixel_x, pixel_y, width):
    head_ptr = HeadPointers[pixel_x + pixel_y * width]
    cur_ptr = head_ptr
    
    while cur_ptr != -1:
        depth = NodeData[cur_ptr] / framebuffer_width
        # Process voxel at this depth
        process_voxel(depth)
        cur_ptr = NodePointers[cur_ptr]
```

### Parameters
*   `framebufferWidth`: Integer uniform representing the width of the framebuffer. Used to calculate pixel indices and scale depth values.
*   `counter`: Atomic unsigned integer used to generate unique indices for each fragment.
*   `HeadPointers` initialization: All elements must be initialized to -1 before rasterization.
*   `NodePointers`/`NodeData` size: Must be large enough to hold the maximum possible number of fragments. Not given in the sources.

### Edge Cases and Pitfalls
*   **Infinite Loops**: Linked list traversal can enter infinite loops if the list is corrupted. The implementation prevents this by limiting the outer and inner loops to a maximum of 1024 iterations [S1, unofficial].
*   **Empty Lists**: Pixels with no fragments have a head pointer of -1. The sort shader discards these pixels to avoid processing errors [S1, unofficial].
*   **Depth Precision**: Depth is stored as an integer scaled by `framebufferWidth`. This may lose precision for very large framebuffers or very small depth differences. Not given in the sources.
*   **Memory Exhaustion**: If the number of fragments exceeds the allocated size of `NodePointers` and `NodeData`, memory corruption or crashes may occur. Not given in the sources.

## Operating characteristics
*   **Time Complexity**:
    *   Construction: $O(N)$ where $N$ is the number of fragments, due to atomic operations.
    *   Sorting: $O(K^2)$ per pixel where $K$ is the number of fragments per pixel, due to the selection sort implementation. This is inefficient for pixels with many fragments.
*   **Memory Complexity**: $O(W \times H + N)$ where $W, H$ are framebuffer dimensions and $N$ is the maximum number of fragments.
*   **Strengths**:
    *   Handles multiple hits per pixel naturally.
    *   Depth-sorted output is ready for ray marching or volume rendering.
    *   GPU-accelerated.
*   **Limitations**:
    *   Sorting is $O(K^2)$, which is slow for high fragment counts per pixel.
    *   Requires large SSBOs for `NodePointers` and `NodeData`.
    *   Atomic operations can cause contention.
*   **When to Use**: Suitable for scenes with moderate complexity where multiple hits per pixel are expected and a depth-sorted list is required for subsequent rendering steps.
*   **When Not to Use**: Not suitable for very high fragment counts per pixel due to sorting inefficiency, or when memory bandwidth is a critical bottleneck.

## Background
The method is implemented in the `Voxelizer` project by Kelsey Kurzeja (2014) [S1, unofficial]. It is based on the A-Buffer concept, adapted for voxelization using modern GPU features like atomic counters and Shader Storage Buffer Objects (SSBOs). The technique allows for the accumulation of depth information from rasterized geometry into a structure that can be used for volume rendering or further processing.

## Sources
[S1] khkurzeja/Voxelizer, https://github.com/khkurzeja/Voxelizer, fetched 2026-10-26 (unofficial)
