# VOXLAP RLE Voxel Storage

Also called: VxlCompiledRLE, VxlMap

## Overview
VOXLAP RLE Voxel Storage is an in-memory compressed voxel data structure used by the VOXLAP engine for CPU-based volume rendering. It solves the problem of efficiently storing and editing sparse voxel grids using a Run-Length Encoding (RLE) scheme organized as a 2D array of columns. It sets itself apart from generic RLE methods by its specific column-oriented arena allocation and batched edit support, tailored for the constraints of the original VOXLAP engine [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method manages a voxel grid defined by dimensions $X, Y, Z$. The primary input is a set of rectangular edit operations (slabs) defined by $(x_{start}, x_{end}, y_{start}, y_{end}, z_{start}, z_{end})$ and a state value. The output is the updated internal state of the voxel storage, allowing for subsequent rendering or querying.

**Data Structures:**

1.  **VxlColor**: A packed 4-byte structure representing color and brightness.
    *   Layout: `b` (u8), `g` (u8), `r` (u8), `shade` (u8).
    *   The `shade` field acts as a brightness multiplier: $0x00 \rightarrow 0.0$, $0x80 \rightarrow 1.0$, $0xFF \rightarrow 2.0$ [S1, unofficial].

2.  **VxlState**: An enumeration of voxel states.
    *   Variants: `Air`, `Unexposed`, `Color(VxlColor)` [S1, unofficial].

3.  **VxlSlab**: Represents a contiguous vertical range of voxels.
    *   Fields: `start` (u8), `end` (u8).
    *   Semantics: Represents an inclusive z-range $[start, end]$. Internally, it is stored as a half-open interval $[start, end + 1)$ to facilitate arithmetic [S1, unofficial].

4.  **VxlArena**: A fixed-size memory pool for storing RLE spans.
    *   Total Capacity: $VXL\_ARENA\_WORDS = 4,194,304$ 32-bit words.
    *   Per-Column Allocation: $VXL\_ARENA\_WORDS\_PER\_COLUMN = 16$ words.
    *   Maximum Dimensions: $VSID = 512$ (max x and y), $MAXZDIM = 64$ (max z).
    *   Derived Constant: $MAX\_VERTICAL\_STACK = 32$ ($MAXZDIM / 2$) [S1, unofficial].
    *   The arena uses a bitarray (`VxlArenaBits`) to track allocated words and a data array (`VxlData`) to store the RLE values.

**Algorithm Steps:**

The storage supports batched edits via a context object (`ctx`). The following steps describe the `set_rect` operation, which applies a rectangular volume change:

1.  **Initialize Context**: Create a `VxlSlab` with `start = z_start` and `end = z_end + 1`. This defines the vertical extent of the edit [S1, unofficial].
2.  **Iterate Y**: Loop $y$ from $y_{start}$ to $y_{end}$ (inclusive) [S1, unofficial].
3.  **Iterate X**: Loop $x$ from $x_{start}$ to $y_{end}$ (inclusive). *Note: The source code iterates to $y_{end}$ instead of $x_{end}$, which is a documented implementation detail or potential bug in the reference implementation* [S1, unofficial].
4.  **Get Column**: Call `ctx.edit_column(x)` to retrieve a mutable reference to the column data structure for the current x-coordinate [S1, unofficial].
5.  **Apply State**: Determine the target state (solid or empty).
    *   If the target state is solid (non-air), call `column.insert(slab)` to add the RLE span to the column [S1, unofficial].
    *   If the target state is empty (air), call `column.remove(slab)` to remove the RLE span [S1, unofficial].
6.  **Flush**: After the inner X loop completes for a given y, call `ctx.advance_and_flush()` to commit changes for that row [S1, unofficial].
7.  **Finalize**: After all loops complete, call `ctx.finish()` to finalize the batched edits [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def set_rect(ctx, x_start, x_end, y_start, y_end, z_start, z_end, is_solid):
    """
    Applies a rectangular edit to the voxel storage.
    Note: The reference implementation iterates x to y_end.
    """
    # Step 1: Initialize slab context
    slab = VxlSlab(start=z_start, end=z_end + 1)
    
    # Step 2: Outer loop Y
    for y in range(y_start, y_end + 1):
        # Step 3: Inner loop X (Reference uses y_end as limit)
        for x in range(x_start, y_end + 1):
            # Step 4: Get column reference
            column = ctx.edit_column(x)
            
            # Step 5: Apply state
            if is_solid:
                column.insert(slab)
            else:
                column.remove(slab)
        
        # Step 6: Flush after each Y iteration
        ctx.advance_and_flush()
    
    # Step 7: Finish batch
    ctx.finish()
```

**Parameters and Constants:**
*   $VSID = 512$: Maximum dimension for X and Y axes [S1, unofficial].
*   $MAXZDIM = 64$: Maximum dimension for Z axis [S1, unofficial].
*   $VXL\_ARENA\_WORDS\_PER\_COLUMN = 16$: Words allocated per column [S1, unofficial].
*   $VXL\_ARENA\_WORDS = 4,194,304$: Total arena size in 32-bit words [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Iteration Bug**: The reference implementation uses $y_{end}$ as the upper bound for the X loop, which may cause incorrect behavior if $x_{end} \neq y_{end}$ [S1, unofficial].
*   **Arena Saturation**: The arena has a fixed size. If the number of RLE spans exceeds the allocated words, allocation may fail or overwrite data. The source notes that saturation is unlikely in normal use [S1, unofficial].
*   **Precision**: The Z coordinates are stored as `u8`, limiting the maximum Z dimension to 256, though the engine restricts it to 64 [S1, unofficial].

## Operating characteristics
*   **Memory**: The storage uses a fixed-size arena of 16 MB ($4,194,304 \times 4$ bytes). Memory usage is static and does not grow with the number of voxels, but is limited by the $VSID \times VSID \times 16$ capacity [S1, unofficial].
*   **Time Complexity**: Edit operations are $O((x_{end}-x_{start}) \times (y_{end}-y_{start}))$ for the loop overhead, plus the cost of RLE insertion/removal within columns. The source does not provide specific complexity for the RLE operations themselves [S1, unofficial].
*   **Performance**: Optimized for CPU rendering in the VOXLAP engine. The column-oriented structure allows for efficient batched updates [S1, unofficial].
*   **Limitations**: The fixed arena size limits the maximum number of distinct RLE spans. The Z dimension is limited to 64. The iteration bug in the reference implementation is a significant pitfall for direct porting [S1, unofficial].

## Background
VOXLAP is a voxel engine created by Ken Silverman. The specific RLE storage format described here is a reimplementation of the storage used in VOXLAP, written in Rust by MuffinTastic. The project was created for educational purposes to learn about VOXLAP and Rust, and is not intended for production use in modern GPU-centric pipelines [S1, unofficial].

## Sources
[S1] MuffinTastic/rust-voxlap-vxl, https://github.com/MuffinTastic/rust-voxlap-vxl, fetched 2026-10-28 (unofficial)
