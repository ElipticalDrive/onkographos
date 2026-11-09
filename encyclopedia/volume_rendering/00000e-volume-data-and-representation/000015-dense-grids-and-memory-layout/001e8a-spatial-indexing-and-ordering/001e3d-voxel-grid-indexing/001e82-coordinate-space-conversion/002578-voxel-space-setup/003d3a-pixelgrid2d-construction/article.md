# PixelGrid2D Construction
Also called: 2D Grid Initialization from BitArray, Grid2D<BitArray, bool> Construction

## Overview
PixelGrid2D Construction initializes a 2D grid structure (`Grid2D<BitArray, bool>`) to store voxel occupancy or scalar data projected onto a 2D plane. It solves the problem of mapping a 3D voxel space's bounding box and resolution into a compact 2D memory layout suitable for rendering or processing. It differs from its sibling [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md) by using a Cartesian bounding box rather than spherical coordinates, and from [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md) by performing a static initialization rather than dynamic coordinate transformation.

## How it works
**Inputs:**
- `boundingBox`: A 3D bounding box defining the spatial extent.
- `pixelDimensions`: A `Point3d` (X, Y, Z) representing the physical size of a single voxel/pixel in each dimension.
- `bitArray`: A `BitArray` containing the initial voxel data.

**Outputs:**
- A `PixelGrid2D` instance containing a 2D grid of `BitArray` elements, each storing boolean values.

**Data Structures:**
- `Grid2D<T, U>`: A 2D array structure. In this implementation, `T` is `BitArray` and `U` is `bool`.
- `BitArray`: A dynamic array of bits.
- `Point3i`: An integer vector (X, Y, Z).
- `SelfProduct`: A method on `Point3i` returning the product $X \times Y \times Z$.

**Steps:**
1.  **Initialize Bounding Box and Pixel Size:** Extract the bounding box dimensions and pixel dimensions from the input parameters [S1, unofficial].
2.  **Calculate X Dimension Size:** Compute the number of pixels along the X-axis by dividing the bounding box X-length by the pixel X-dimension and flooring the result:
    $$ N_x = \lfloor \frac{\text{BoundingBox.XLength}}{\text{PixelDimensions.X}} \rfloor $$
3.  **Calculate Y Dimension Size:** Compute the number of pixels along the Y-axis by dividing the bounding box Y-length by the pixel Y-dimension and flooring the result:
    $$ N_y = \lfloor \frac{\text{BoundingBox.YLength}}{\text{PixelDimensions.Y}} \rfloor $$
4.  **Set Grid Size UV:** Create a `Point3i` (or equivalent 2D size structure) with dimensions $(N_x, N_y, 1)$ or $(N_x, N_y)$ depending on internal representation, representing the grid's width and height [S1, unofficial].
5.  **Inject BitArray:** Inject the provided `BitArray` directly into the grid structure without copying, if the implementation supports direct reference injection [S1, unofficial].
6.  **Create Default BitArray (Fallback/Initialization):** If direct injection is not performed or for internal consistency, create a new `BitArray` with size equal to the product of the grid dimensions ($N_x \times N_y$), initialized to `false` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class PixelGrid2D:
    def __init__(self, bounding_box, pixel_dimensions, bit_array):
        # Step 1: Initialize bounding box and pixel size
        # bounding_box has properties: XLength, YLength
        # pixel_dimensions is a Point3d with properties: X, Y, Z
        
        # Step 2: Calculate X dimension size
        # Floor the division of bounding box X length by pixel X dimension
        self.grid_width = floor(bounding_box.XLength / pixel_dimensions.X)
        
        # Step 3: Calculate Y dimension size
        # Floor the division of bounding box Y length by pixel Y dimension
        self.grid_height = floor(bounding_box.YLength / pixel_dimensions.Y)
        
        # Step 4: Set the grid size UV
        # Create a Point3i or similar structure for grid dimensions
        # Note: Z dimension is typically 1 or ignored for 2D grid indexing
        self.grid_size_uv = Point3i(self.grid_width, self.grid_height, 1)
        
        # Step 5: Inject the provided BitArray directly into the grid without copying
        # Implementation detail: The grid is a Grid2D<BitArray, bool>
        # This implies the grid stores BitArrays.
        # If the source BitArray fits the total size, it is injected.
        # If not, or if a new structure is needed:
        
        # Step 6: Create a new BitArray with size equal to the product of the grid dimensions
        total_size = self.grid_width * self.grid_height
        self.grid_data = Grid2D(BitArray, bool)
        
        # Initialize the grid with BitArrays
        # Each cell in the 2D grid might hold a BitArray, or the grid itself is a flat BitArray
        # Based on "Grid2D<BitArray, bool>", it suggests a 2D array of BitArrays.
        # However, "Inject the provided BitArray directly" suggests a flat structure or a single BitArray backing.
        # Given the ambiguity of "Grid2D<BitArray, bool>", we assume a 2D array of BitArrays is created,
        # or the BitArray is mapped to the 2D indices.
        
        # Re-reading [S1, unofficial]: "Inject the provided BitArray directly into the grid without copying."
        # This suggests the grid's internal storage is the BitArray itself, or a reference to it.
        # "Create a new BitArray with size equal to the product of the grid dimensions... initialized to false."
        # This step seems to be an alternative or a fallback if injection fails or for initialization.
        
        # Let's assume the standard construction:
        # The grid is initialized with dimensions (grid_width, grid_height).
        # The data is stored in a BitArray of size grid_width * grid_height.
        
        self.bit_array = bit_array if bit_array else BitArray(total_size, default=False)
        
        # Mapping 2D coordinates (u, v) to 1D index in BitArray
        # index = v * grid_width + u
```

**Parameters:**
- `boundingBox`: Not given in the sources.
- `pixelDimensions`: Not given in the sources.
- `bitArray`: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point division followed by flooring can lead to off-by-one errors if the bounding box dimensions are not exact multiples of the pixel dimensions.
- **Boundaries:** If the bounding box is smaller than the pixel size, the resulting grid dimension will be 0.
- **Degenerate Input:** If `pixelDimensions` contains zero, division by zero occurs.

## Subtopics

### PixelGrid2D SetValue
- **What it is:** A method to set a boolean value at a specific index in the internal grid array.
- **What sets it apart:** It provides direct access to modify the voxel state after construction.
- **When to choose it:** When updating specific voxel values in the 2D grid.
- **Link:** [PixelGrid2D Construction](3521-pixelgrid2d-construction.md)
- **Step:** Set the value in the internal Grid array at index $i_{UV}$ to the provided boolean value [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for initialization if the BitArray size is known and pre-allocated. $O(N)$ if the BitArray needs to be created and initialized, where $N$ is the number of pixels ($N_x \times N_y$).
- **Memory Complexity:** $O(N)$ bits, where $N$ is the number of pixels.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of pixels.
- **Strengths:** Compact memory usage due to BitArray; direct injection avoids copying overhead.
- **Limitations:** Limited to boolean data; fixed 2D layout.
- **When to use it:** When a compact 2D representation of voxel data is needed for rendering or processing.
- **When not:** When 3D data is required, or when non-boolean data is needed.

## Background
- **Origin:** The method is part of the `voxeltool` library by Arend van der Werff [S1, unofficial].
- **History:** The library is a collection of voxel-related utilities. The specific implementation details are found in the source code of the library.
- **Usage:** Used in voxel rendering and processing applications where 2D grids are needed for projection or slicing.

## Sources
[S1] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
