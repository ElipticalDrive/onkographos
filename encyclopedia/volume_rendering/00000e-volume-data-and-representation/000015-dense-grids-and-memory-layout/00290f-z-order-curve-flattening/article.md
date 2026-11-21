# Z-order Curve Flattening

Also called: Morton-order curve, Z-curve, Z-order memory layout

## Overview
Z-order curve flattening is a memory layout technique that maps 3D voxel coordinates to a 1D linear index using a space-filling curve, placing voxels that are spatially adjacent in memory adjacent to each other [S1, S2, unofficial]. It solves the index-space locality problem inherent in standard row-major (a-order) layouts, where coordinates that are neighbors in higher dimensions (e.g., $y$ or $z$) are separated by large memory strides [S2, unofficial]. By improving spatial locality, it increases GPU cache hit rates during ray traversal and reduces memory bandwidth pressure when nearby rays access similar data [S1, unofficial]. It is chosen over other space-filling curves like the Hilbert curve primarily because Z-order indices are computationally inexpensive to calculate [S2, unofficial].

## How it works
**Inputs:**
*   Grid dimensions $(M_x, M_y, M_z)$, ideally even powers of two [S2, unofficial].
*   Input data array $A$ in a-order (row-major) layout [S2, unofficial].
*   Element size in bytes (e.g., `sizeof(float)`) [S2, unofficial].

**Outputs:**
*   A new data array $Z$ containing the same values as $A$ but rearranged into Z-order layout [S2, unofficial].
*   A mapping structure (e.g., `ZOrderStruct`) containing precomputed tables for fast index calculation [S2, unofficial].

**Data Structures:**
*   `ZOrderStruct`: An opaque structure returned by initialization. It contains precomputed tables used to accelerate the computation of both a-order and z-order indices [S2, unofficial].
*   Linear Index $I$: A 64-bit integer offset (e.g., `off_t`) representing the position in the 1D array [S2, unofficial].

**Algorithm:**
The method involves three phases: Initialization, Conversion, and Access.

1.  **Initialization:**
    Call `zoInitZOrderIndexing` with dimensions $(i, j, k)$ and a layout enumerator (`ZORDER_LAYOUT` or `AORDER_LAYOUT`). This returns a `ZOrderStruct` pointer. The routine precomputes small tables to accelerate index computation [S2, unofficial].

2.  **Data Conversion:**
    Call `zoConvertArrayOrderToZOrder` with the `ZOrderStruct`, the input array pointer $A$ (cast to `void*`), and the element size. This allocates a new array $Z$ and copies/rearranges data from $A$ into Z-order layout. This step is not zero-copy; it always allocates new memory [S2, unofficial].

3.  **Index Calculation and Access:**
    To access a voxel at $(i, j, k)$:
    *   Compute the linear index $I$ using `zoGetIndex`, `zoGetIndexZOrder`, or `zoGetIndexArrayOrder` depending on the layout specified during initialization [S2, unofficial].
    *   Access the value: $V = Z[I]$ [S2, unofficial].

**Pseudocode:**
The following Python-like pseudocode illustrates the logical flow of the Z-order library API described in [S2, unofficial]. It assumes the existence of the underlying C library functions.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import ctypes

# 1. Initialization
# Define dimensions (ideally powers of 2)
dims = (1024, 1024, 1024)
layout_type = ZORDER_LAYOUT  # Enum: ZORDER_LAYOUT or AORDER_LAYOUT

# Call C library function to initialize structure
# zoInitZOrderIndexing returns a pointer to ZOrderStruct
zo_struct_ptr = zoInitZOrderIndexing(dims[0], dims[1], dims[2], layout_type)

# 2. Data Conversion
# Assume 'a_order_data' is a numpy array or buffer in row-major order
# zoConvertArrayOrderToZOrder takes: ZOrderStruct, input_ptr (void*), element_size
# It returns a pointer to a new array 'z_order_data'
element_size = ctypes.sizeof(ctypes.c_double) # Example for doubles
z_order_data_ptr = zoConvertArrayOrderToZOrder(zo_struct_ptr, a_order_data_ptr, element_size)

# 3. Accessing Data
# To access voxel at (i, j, k):
i, j, k = 100, 200, 300

# Option A: Generic access (uses layout_type from init)
index = zoGetIndex(zo_struct_ptr, i, j, k)

# Option B: Z-order specific access (bypasses generic check)
# index = zoGetIndexZOrder(zo_struct_ptr, i, j, k)

# Access the value
value = z_order_data_ptr[index]
```

**Formulas:**
The Z-order index $I$ for coordinates $(x, y, z)$ is computed by interleaving the bits of the coordinates. For a 3D grid, if $x, y, z$ are represented in binary as $x = \sum x_m 2^m$, $y = \sum y_m 2^m$, $z = \sum z_m 2^m$, then:
$$ I = \sum_{m=0}^{M-1} (x_m 2^{2m} + y_m 2^{2m+1} + z_m 2^{2m+2}) $$
where $M$ is the number of bits required to represent the maximum dimension [S2, implied by Morton-order definition, unofficial].

**Parameters:**
*   `ZORDER_LAYOUT` / `AORDER_LAYOUT`: Enumerators passed to initialization. Determines the indexing logic for the generic `zoGetIndex` routine [S2, unofficial].
*   `element_size`: Byte size of array elements, required for correct memory copying during conversion [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Power-of-Two Dimensions:** The Z-order layout is optimized for dimensions that are even powers of two. If dimensions are not powers of two, padding is applied, inflating the memory footprint by 1.5× to 4× depending on the deviation [S2, unofficial].
*   **Memory Allocation:** The conversion step `zoConvertArrayOrderToZOrder` always allocates a new array. There is no zero-copy in-place conversion. Data must be loaded in Z-order format initially if memory allocation is a constraint [S2, unofficial].

## Subtopics

### Morton-Order Curve
The mathematical basis for Z-order, involving bit-interleaving. It is the standard implementation of Z-order in many graphics libraries. [Link to sibling if available, otherwise general reference].

### Hilbert Curve
An alternative space-filling curve that offers better locality than Z-order but is significantly more expensive to compute. Z-order is preferred in volume rendering for its speed [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Index computation is $O(1)$ with precomputed tables or $O(\log N)$ bit operations [S2, unofficial]. Conversion is $O(N)$ where $N$ is the number of voxels [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for the data array. Additional overhead for `ZOrderStruct` tables is negligible [S2, unofficial]. Memory footprint can increase by 1.5×–4× if dimensions are not powers of two [S2, unofficial].
*   **Performance:**
    *   In a 2048³ grid raycasting scenario, Z-order flattening increased frame rates from 12 FPS to 102 FPS (over 8× speedup) compared to standard flattening [S1, unofficial].
    *   Performance gains vary by access pattern and hardware, ranging from negligible to 800% faster runtime on Intel Ivy Bridge and Knight's Corner architectures [S2, unofficial].
*   **Strengths:**
    *   Improves GPU cache locality for ray traversal (neighboring voxels are in cache) [S1, unofficial].
    *   Improves locality for multiple rays accessing nearby memory [S1, unofficial].
    *   Low computational cost for index generation compared to Hilbert curves [S2, unofficial].
*   **Limitations:**
    *   Requires data conversion (copying) if input is in row-major order [S2, unofficial].
    *   Memory inflation for non-power-of-two dimensions [S2, unofficial].
    *   Less locality preservation than Hilbert curves for certain access patterns, though the trade-off favors Z-order for its speed [S2, unofficial].

## Background
Z-order curve flattening, also known as Morton-order, was developed to address the index-space locality problem in multidimensional array storage [S2, unofficial]. Early research by Lam et al. (1991) focused on blocking and tiling to improve cache utilization, but these methods required intrusive code modifications [S2, unofficial]. Space-filling curves were proposed as a non-intrusive alternative. The Z-order curve was selected for widespread use in scientific computing and graphics due to its balance of locality and computational efficiency [S2, unofficial]. It is commonly used in raycasting volume rendering, bilateral smoothing filters, and data-intensive codes with structured or unstructured memory access patterns [S2, unofficial].

## Sources
[S1] This Tiny Algorithm Can Render BILLIONS of Voxels in Real Time, https://www.youtube.com/watch?v=ztkh1r1ioZo, fetched 2026-10-25 (unofficial)
[S2] zorder-lib: Library API for Z-Order Memory Layout, https://doi.org/10.2172/1460301, fetched 2026-10-26 (unofficial)
