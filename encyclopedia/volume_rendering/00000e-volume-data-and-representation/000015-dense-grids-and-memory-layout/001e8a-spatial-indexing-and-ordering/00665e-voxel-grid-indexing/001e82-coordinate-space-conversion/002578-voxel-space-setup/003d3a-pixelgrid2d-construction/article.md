# PixelGrid2D Construction

Also called: 2D Grid Initialization from BitArray, Grid2D<BitArray, bool> Construction

## Overview
Constructs a 2D grid structure (`Grid2D<BitArray, bool>`) that maps voxel coordinates to boolean occupancy flags, initialized from a bounding box and a pre-computed bit array. It differs from [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md) by assuming a static, pre-calculated set of active voxels provided as a flat bit stream, rather than computing occupancy dynamically during ray traversal. It serves as the foundational data structure for [Voxel Center Precomputation](10731-voxel-center-precomputation.md) by providing the spatial index for active cells.

## How it works
**Inputs:**
1. `boundingBox`: A 3D bounding box defining the spatial extent (min/max coordinates).
2. `pixelDimensions`: A 3D vector $(P_x, P_y, P_z)$ representing the physical size of a single voxel.
3. `bitArray`: A `BitArray` containing the occupancy status of voxels, ordered in a specific traversal order (typically Z-Y-X or similar, consistent with the parent `Voxel Space Setup`).

**Outputs:**
1. `Grid2D<BitArray, bool>`: A 2D grid where each cell contains a `BitArray` representing a column of voxels (or a slice, depending on the specific implementation of `Grid2D`). *Note: The source [S1, unofficial] describes the result as `Grid2D<BitArray, bool>`, but the construction steps imply a direct mapping from a 1D bit array to a 2D grid structure. The exact internal layout of `Grid2D` is not fully detailed in the provided snippets, but it is initialized with dimensions derived from the bounding box.*

**Data Structures:**
*   `Point3i`: Represents integer dimensions $(X, Y, Z)$.
*   `BitArray`: A dynamic array of bits.
*   `Grid2D<T, U>`: A generic 2D grid container. In this context, it stores `BitArray` objects.

**Steps:**
1.  **Initialize Bounding Box and Pixel Size:** Extract the bounding box dimensions and pixel sizes from the input parameters.
2.  **Calculate X Dimension Size:** Compute the number of voxels in the X direction:
    $$ N_x = \lfloor \frac{B_x}{P_x} \rfloor $$
    where $B_x$ is the length of the bounding box in the X dimension, and $P_x$ is the pixel size in X.
3.  **Calculate Y Dimension Size:** Compute the number of voxels in the Y direction:
    $$ N_y = \lfloor \frac{B_y}{P_y} \rfloor $$
    where $B_y$ is the length of the bounding box in the Y dimension, and $P_y$ is the pixel size in Y.
4.  **Set Grid Size UV:** Set the grid dimensions to $(N_x, N_y)$.
5.  **Inject BitArray:** Inject the provided `BitArray` directly into the grid structure. The source [S1, unofficial] states "Inject the provided BitArray directly into the grid without copying," implying a reference transfer or direct memory mapping if possible.
6.  **Create SelfProduct BitArray:** Create a new `BitArray` with size equal to the product of the grid dimensions:
    $$ S = N_x \times N_y $$
    This new bit array is initialized to `false`. *Note: The purpose of this step in the context of construction is ambiguous. It may be for internal buffering or a specific implementation detail of the `Grid2D` class where the `BitArray` is stored internally. Given the instruction to inject the provided BitArray, this step might be creating an internal representation or a mask.*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_pixel_grid_2d(bounding_box, pixel_dimensions, input_bit_array):
    # Extract bounding box lengths
    box_x = bounding_box.max_x - bounding_box.min_x
    box_y = bounding_box.max_y - bounding_box.min_y
    
    # Extract pixel dimensions
    px = pixel_dimensions.x
    py = pixel_dimensions.y
    
    # Calculate grid dimensions
    # Floor the result as per source [S1, unofficial]
    grid_width = floor(box_x / px)
    grid_height = floor(box_y / py)
    
    # Initialize the 2D grid structure
    # The source specifies Grid2D<BitArray, bool>
    # We assume a structure that can hold BitArrays at each (x, y) coordinate
    grid = Grid2D(width=grid_width, height=grid_height)
    
    # Inject the provided BitArray directly
    # Note: The exact mechanism of "injection" into a 2D grid from a 1D bit array
    # depends on the traversal order defined in Voxel Space Setup.
    # For implementation, we assume the bit array is mapped to the grid cells.
    # Since the source says "without copying", we might just assign the reference
    # if the grid structure supports it, or populate it.
    
    # The source also mentions creating a new BitArray with size SelfProduct
    self_product_size = grid_width * grid_height
    internal_bit_array = BitArray(size=self_product_size, initial_value=False)
    
    # Inject input_bit_array into the grid
    # This step is abstract in the source. We assume the grid stores the bit array.
    grid.bit_array = input_bit_array
    
    return grid
```

**Parameters:**
*   `floor`: Standard floor function.
*   `BitArray(size, initial_value)`: Constructor for a bit array.

**Edge Cases:**
*   **Non-integer Dimensions:** If the bounding box dimensions are not exact multiples of the pixel dimensions, the floor operation discards the partial voxel. This may lead to a slight mismatch between the physical bounding box and the grid coverage.
*   **BitArray Size Mismatch:** The source [S1, unofficial] does not explicitly state how the size of the `input_bit_array` relates to $N_x \times N_y \times N_z$. If the input bit array is for a 3D volume, the `Grid2D` construction might only be handling a specific slice or projection, or the `Grid2D` is a misnomer for a 3D grid in the source context. However, based on the name "PixelGrid2D", it is assumed to be 2D. If the input bit array is 3D, the mapping to 2D is not defined in the provided notes. *Assumption: The input bit array corresponds to the 2D grid dimensions or the method is part of a larger 3D construction where this step handles a specific layer.*

## Subtopics

### PixelGrid2D SetValue
Sets the value in the internal Grid array at index $i_{UV}$ to the provided boolean value. This allows for dynamic modification of the occupancy flags after construction. The index $i_{UV}$ is calculated based on the 2D coordinates $(u, v)$ using the grid width: $i_{UV} = u \times \text{width} + v$.

### TraceGrid Construction
Constructs a 2D grid mapping inline/xline coordinates to trace indices, enabling spatial access to seismic traces. It handles sparse or irregular trace layouts by initializing empty cells with -1.
**Steps:**
1.  Iterate over all `TraceMeta` objects in `metaList`. For each, add its inline value to a `HashSet<Integer>` `inlines` and its xline value to a `HashSet<Integer>` `xlines`.
2.  Convert the `inlines` HashSet to a sorted int array `inlineValues`. Convert the `xlines` HashSet to a sorted int array `xlineValues`.
3.  Create a `HashMap<Integer, Integer>` `inlineIndex` mapping each inline value to its index in `inlineValues`. Create a `HashMap<Integer, Integer>` `xlineIndex` mapping each xline value to its index in `xlineValues`.
4.  Initialize a 2D int array `grid` of size `[inlineValues.length][xlineValues.length]`. Fill all entries with -1.
5.  Iterate over all `TraceMeta` objects again. For each, look up its inline index `ii` and xline index `jj` using the maps. If both are non-null, set `grid[ii][jj] = tm.traceIndex`.
**Pitfall:** If a trace's inline or xline value is not present in the respective sorted arrays (should not happen if `metaList` is consistent), the map lookup returns null. The code checks for null before accessing the grid to avoid `NullPointerException`.

## Operating characteristics
*   **Time Complexity:** $O(1)$ for the construction if the bit array is injected by reference. If the bit array needs to be copied or mapped, it is $O(N)$ where $N$ is the number of bits. The `TraceGrid` construction is $O(T \log T)$ due to sorting, where $T$ is the number of traces.
*   **Memory Complexity:** $O(N)$ for the bit array, where $N$ is the number of voxels/traces. The `TraceGrid` uses $O(I \times X)$ space for the grid, where $I$ and $X$ are the number of unique inlines and xlines.
*   **Strengths:** Efficient storage of sparse data using bit arrays. Fast spatial lookup for seismic traces using the `TraceGrid`.
*   **Limitations:** The `PixelGrid2D` construction assumes a specific bit array format. The `TraceGrid` construction requires two passes over the trace metadata.

## Background
The `PixelGrid2D` construction is part of the `voxeltools` library [S1, unofficial], which provides utilities for voxel-based data structures. The `TraceGrid` construction is part of the `SeismicDataCompressor` project [S2, unofficial], which deals with seismic data storage and retrieval.

## Sources
[S1] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
[S2] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
