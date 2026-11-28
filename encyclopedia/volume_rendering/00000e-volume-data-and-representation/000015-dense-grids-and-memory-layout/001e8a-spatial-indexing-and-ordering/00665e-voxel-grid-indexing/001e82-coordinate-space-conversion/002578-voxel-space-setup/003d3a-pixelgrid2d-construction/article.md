# PixelGrid2D Construction

Also called: 2D Grid Initialization from BitArray, Grid2D<BitArray, bool> Construction

## Overview
Constructs a 2D grid structure from a bounding box, voxel dimensions, and a pre-computed bit array [S1, unofficial]. It maps spatial coordinates to boolean occupancy flags initialized from a flat bit stream, differing from dynamic occupancy methods like [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md) by using a static pre-calculated set of active voxels [G]. It provides the spatial index foundation for [Voxel Center Precomputation](10731-voxel-center-precomputation.md) [G].

## How it works
**Inputs:**
1. `boundingBox`: A 3D bounding box defining the spatial extent [S1, unofficial].
2. `pixelDimensions`: A 3D vector $(P_x, P_y, P_z)$ representing the physical size of a single voxel [S1, unofficial].
3. `bitArray`: A `BitArray` containing the occupancy status of voxels [S1, unofficial].

**Outputs:**
1. 2D grid structure mapping coordinates to boolean occupancy flags [S1, unofficial].

**Data Structures:**
* `Point3i`: Represents integer dimensions $(X, Y, Z)$ [S1, unofficial].
* `BitArray`: A dynamic array of bits [S1, unofficial].
* 2D grid container: Stores the grid dimensions and references the bit array [S1, unofficial].

**Steps:**
1. Initialize the bounding box and pixel size from the input parameters [S1, unofficial].
2. Calculate the X dimension size by dividing the bounding box X length by the pixel X dimension and flooring the result [S1, unofficial].
3. Calculate the Y dimension size by dividing the bounding box Y length by the pixel Y dimension and flooring the result [S1, unofficial].
4. Set the grid size UV to the calculated dimensions [S1, unofficial].
5. Inject the provided `bitArray` directly into the grid without copying [S1, unofficial].
6. Create a new `BitArray` with size equal to the product of the grid dimensions (SelfProduct), initialized to `false` [S1, unofficial].

**Formulas:**
$$ N_x = \left\lfloor \frac{B_x}{P_x} \right\rfloor $$
where $B_x$ is the length of the bounding box in the X dimension and $P_x$ is the pixel size in X [S1, unofficial].
$$ N_y = \left\lfloor \frac{B_y}{P_y} \right\rfloor $$
where $B_y$ is the length of the bounding box in the Y dimension and $P_y$ is the pixel size in Y [S1, unofficial].
$$ S = N_x \times N_y $$
where $S$ is the SelfProduct size for the new `BitArray` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class BitArray:
    def __init__(self, size, initial_value=False):
        self.size = size
        self.data = [initial_value] * size

class Grid2D:
    def __init__(self, width, height):
        self.width = width
        self.height = height
        self.bit_array = None

def construct_pixel_grid_2d(bounding_box, pixel_dimensions, input_bit_array):
    # bounding_box: dict with 'min_x', 'max_x', 'min_y', 'max_y'
    # pixel_dimensions: dict with 'x', 'y', 'z'
    box_x = bounding_box['max_x'] - bounding_box['min_x']
    box_y = bounding_box['max_y'] - bounding_box['min_y']
    px = pixel_dimensions['x']
    py = pixel_dimensions['y']

    grid_width = int(box_x // px)
    grid_height = int(box_y // py)

    grid = Grid2D(width=grid_width, height=grid_height)
    grid.bit_array = input_bit_array

    self_product_size = grid_width * grid_height
    internal_bit_array = BitArray(size=self_product_size, initial_value=False)

    return grid
```

**Parameters:**
* Integer division (floor) for dimension calculation [S1, unofficial].
* `BitArray(size, initial_value)` constructor [S1, unofficial].

**Edge Cases:**
* Non-integer dimensions: If the bounding box dimensions are not exact multiples of the pixel dimensions, the floor operation discards the partial voxel [S1, unofficial].
* BitArray size mismatch: The sources do not explicitly define how the input `bitArray` size relates to the 2D grid dimensions or a potential 3D volume [S1, unofficial].

## Subtopics

### PixelGrid2D SetValue
Sets the value in the internal grid array at index $i_{UV}$ to a provided boolean value [S1, unofficial]. The index is calculated from 2D coordinates $(u, v)$ and grid width as $i_{UV} = u \times \text{width} + v$ [G].

### TraceGrid Construction
Constructs a 2D grid mapping inline/xline coordinates to trace indices, enabling spatial access to seismic traces [S2, unofficial]. It handles sparse or irregular trace layouts by initializing empty cells with -1 [S2, unofficial].
1. Iterate over all `TraceMeta` objects in `metaList`, adding inline values to a `HashSet<Integer>` `inlines` and xline values to a `HashSet<Integer>` `xlines` [S2, unofficial].
2. Convert the `inlines` HashSet to a sorted int array `inlineValues` and the `xlines` HashSet to a sorted int array `xlineValues` [S2, unofficial].
3. Create a `HashMap<Integer, Integer>` `inlineIndex` mapping each inline value to its index in `inlineValues`, and a `HashMap<Integer, Integer>` `xlineIndex` mapping each xline value to its index in `xlineValues` [S2, unofficial].
4. Initialize a 2D int array `grid` of size `[inlineValues.length][xlineValues.length]` and fill all entries with -1 [S2, unofficial].
5. Iterate over all `TraceMeta` objects again, look up inline index `ii` and xline index `jj` using the maps, and set `grid[ii][jj] = tm.traceIndex` if both indices are found [S2, unofficial].
**Pitfall:** If a trace's inline or xline value is missing from the sorted arrays, the map lookup returns null; the implementation must check for null before accessing the grid to avoid `NullPointerException` [S2, unofficial].

## Operating characteristics
* **Time Complexity:** $O(1)$ if the bit array injection is a direct reference transfer, otherwise $O(N)$ for mapping or copying, where $N$ is the number of bits [S1, unofficial]. TraceGrid construction runs in $O(T \log T)$ time due to sorting unique coordinates, where $T$ is the number of traces [S2, unofficial].
* **Memory Complexity:** $O(N)$ for the bit array grid storage [S1, unofficial]. TraceGrid uses $O(I \times X)$ space for the 2D array, where $I$ and $X$ are the counts of unique inlines and xlines [S2, unofficial].
* **Strengths:** Efficient storage of sparse data using bit arrays and fast spatial lookup for seismic traces [S1, unofficial][S2, unofficial].
* **Limitations:** Requires a specific bit array format and traversal order; TraceGrid requires two full passes over the trace metadata [S1, unofficial][S2, unofficial].

## Background
The PixelGrid2D construction originates from the `voxeltools` library [S1, unofficial]. The TraceGrid construction originates from the `SeismicDataCompressor` project [S2, unofficial]. Both methods address spatial indexing for voxel occupancy and seismic trace data respectively [S1, unofficial][S2, unofficial].

## Sources
[S1] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
[S2] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
