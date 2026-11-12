# Dynamic VBO Range Allocation
## Overview
Dynamic VBO Range Allocation manages vertex buffer object (VBO) memory ranges for 3D Gaussian splats by dynamically resizing the allocated capacity based on the current number of active splats (`quadCount`) [S1, unofficial]. It addresses frequent reallocation during buffer updates by applying a fixed over-allocation factor [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: `quadCount` (current number of active splats or quad primitives) [S1, unofficial].
- Output: A VBO range with capacity equal to `quadCount * OVER_CAPACITY_INCREASE_FACTOR` [S1, unofficial].

**Data Structures:**
- The VBO range is a contiguous memory block allocated via OpenGL buffer objects [S1, unofficial].
- The internal data layout is determined by the parent Gaussian Data Structure; specific byte offsets and strides are not detailed in the sources [S1, unofficial].

**Parameters:**
- `OVER_CAPACITY_INCREASE_FACTOR`: A constant factor of 1.25 used to over-allocate VBO capacity to reduce reallocation frequency [S1, unofficial].

**Algorithm:**
1. If a VBO range currently exists, delete it [S1, unofficial].
2. Calculate the new capacity as `quadCount * OVER_CAPACITY_INCREASE_FACTOR` [S1, unofficial].
3. Allocate a new VBO range with the calculated capacity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
OVER_CAPACITY_INCREASE_FACTOR = 1.25
vbo_handle = None
vbo_capacity = 0

def update_vbo_range(quad_count):
    global vbo_handle, vbo_capacity
    if vbo_handle is not None:
        delete_buffer(vbo_handle)
        vbo_handle = None
    new_capacity = quad_count * OVER_CAPACITY_INCREASE_FACTOR
    vbo_handle = create_buffer(new_capacity)
    vbo_capacity = new_capacity
```

**Edge Cases and Pitfalls:**
- Precision, boundaries, and degenerate input handling are not detailed in the sources [S1, unofficial].

## Subtopics

### [Dynamic Buffer Management via CUDA Virtual Memory](4119-dynamic-buffer-management-via-cuda-virtual-memory.md)
This method uses CUDA’s virtual memory management (VMM) to manage dynamically sized splat buffers, allowing physical memory to grow or shrink within a single consecutive virtual address range without data copying. It specifically addresses the memory overhead of merging buffers by enabling the expansion of an existing target buffer rather than allocating a new combined buffer.

## Operating characteristics
- **Time Complexity:** Reallocation is $O(N)$ where $N$ is the number of elements, due to buffer deletion and recreation [G]. The over-allocation factor reduces reallocation frequency [S1, unofficial].
- **Memory Complexity:** Proportional to `quadCount * 1.25`, resulting in a 25% overhead compared to the exact required size [S1, unofficial].
- **Strengths:** Simple implementation using standard OpenGL APIs; reduces reallocation frequency via over-allocation [S1, unofficial].
- **Limitations:** Does not support in-place expansion; requires buffer deletion and recreation [S1, unofficial].

## Background
This method originates from the implementation details of the "Cube" project by gabkhanfig, which applies 3D Gaussian Splatting techniques [S1, unofficial]. The constant factor of 1.25 is used to over-allocate VBO capacity to reduce reallocation frequency [S1, unofficial].

## Sources
[S1] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
