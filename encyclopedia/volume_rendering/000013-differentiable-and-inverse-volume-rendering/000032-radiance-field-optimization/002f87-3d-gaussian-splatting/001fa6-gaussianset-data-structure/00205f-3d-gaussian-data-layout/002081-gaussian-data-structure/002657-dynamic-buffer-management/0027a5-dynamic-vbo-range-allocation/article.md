# Dynamic VBO Range Allocation


## Overview
Dynamic VBO Range Allocation manages the lifecycle of vertex buffer object (VBO) memory ranges for 3D Gaussian splats by dynamically resizing the allocated capacity based on the current number of active splats. It addresses the problem of frequent reallocation and data copying during buffer merging or expansion by using a fixed over-allocation factor. This method differs from static allocation by adapting to scene complexity and from CUDA Virtual Memory management by relying on standard OpenGL buffer reallocation rather than virtual memory mapping [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `quadCount` (current number of active splats/quad primitives).
- Output: A VBO range with sufficient capacity to hold `quadCount` elements.

**Data Structures:**
- The VBO range is a contiguous memory block allocated via OpenGL buffer objects.
- The layout of the data within the VBO is determined by the parent Gaussian Data Structure, typically storing position, covariance, opacity, and spherical harmonics coefficients per Gaussian. The specific byte offset and stride are not detailed in the source [S1, unofficial].

**Parameters:**
- `OVER_CAPACITY_INCREASE_FACTOR`: A constant factor of 1.25 used to over-allocate VBO capacity to reduce reallocation frequency [S1, unofficial].

**Algorithm:**
1. Check if a VBO range currently exists for the splat buffer.
2. If a VBO range exists:
   a. Determine if the current capacity is sufficient for `quadCount`.
   b. If the current capacity is insufficient or if the implementation strategy dictates a full refresh upon change:
      i. Delete the existing VBO range.
      ii. Calculate the new capacity: `newCapacity = quadCount * OVER_CAPACITY_INCREASE_FACTOR`.
      iii. Allocate a new VBO range with `newCapacity`.
3. If no VBO range exists:
   a. Calculate the initial capacity: `initialCapacity = quadCount * OVER_CAPACITY_INCREASE_FACTOR`.
   b. Allocate a new VBO range with `initialCapacity`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
OVER_CAPACITY_INCREASE_FACTOR = 1.25

# State variables
vbo_range_exists = False
vbo_capacity = 0

def update_vbo_range(quad_count):
    """
    Ensures the VBO range has sufficient capacity for the given number of quads.
    """
    global vbo_range_exists, vbo_capacity
    
    # Calculate required capacity with over-allocation
    required_capacity = quad_count * OVER_CAPACITY_INCREASE_FACTOR
    
    # Check if reallocation is needed
    # Note: The source implies deletion and reallocation if the range exists,
    # potentially ignoring incremental growth if the factor is applied to the total count.
    if vbo_range_exists:
        # If current capacity is less than required, reallocate
        if vbo_capacity < required_capacity:
            delete_vbo_range()
            vbo_range_exists = False
            
    if not vbo_range_exists:
        # Allocate new range
        new_capacity = required_capacity
        allocate_vbo_range(new_capacity)
        vbo_capacity = new_capacity
        vbo_range_exists = True

def delete_vbo_range():
    """
    Deletes the existing VBO range.
    """
    # Implementation detail: Call OpenGL glDeleteBuffers or equivalent
    pass

def allocate_vbo_range(capacity):
    """
    Allocates a new VBO range with the specified capacity.
    """
    # Implementation detail: Call OpenGL glGenBuffers and glBufferData
    pass
```

**Edge Cases and Pitfalls:**
- **Precision:** The multiplication `quadCount * 1.25` should be handled as floating-point or integer ceiling to ensure sufficient capacity.
- **Boundaries:** If `quadCount` is zero, the capacity becomes zero, which may require special handling to avoid invalid buffer allocations.
- **Degenerate Input:** Negative `quadCount` values should be guarded against [G].

## Subtopics

### [Dynamic Buffer Management via CUDA Virtual Memory](4119-dynamic-buffer-management-via-cuda-virtual-memory.md)
This method uses CUDA’s virtual memory management (VMM) to manage dynamically sized splat buffers, allowing physical memory to grow or shrink within a single consecutive virtual address range without data copying. It specifically addresses the memory overhead of merging buffers by enabling the expansion of an existing target buffer rather than allocating a new combined buffer.

## Operating characteristics
- **Time Complexity:** Reallocation is $O(N)$ where $N$ is the number of elements, due to the deletion and reallocation of the buffer. However, the over-allocation factor reduces the frequency of reallocations.
- **Memory Complexity:** The memory usage is proportional to `quadCount * 1.25`, resulting in a 25% overhead compared to the exact required size.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance degrades if `quadCount` fluctuates frequently, as each fluctuation may trigger a full reallocation.
- **Strengths:** Simple to implement using standard OpenGL APIs; reduces reallocation frequency via over-allocation.
- **Limitations:** Does not support in-place expansion; requires buffer deletion and recreation, which may cause GPU stalls.
- **When to use:** Suitable for scenes with relatively stable Gaussian counts or when CPU-side buffer management is preferred over GPU virtual memory management.
- **When not:** Not suitable for highly dynamic scenes with frequent large changes in Gaussian count, or when zero-copy expansion is required.

## Background
This method is derived from the implementation details of the "Cube" project by gabkhanfig, which applies 3D Gaussian Splatting techniques. The specific use of a 1.25 over-allocation factor is a design choice to balance memory efficiency and reallocation frequency [S1, unofficial].

## Sources
[S1] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
