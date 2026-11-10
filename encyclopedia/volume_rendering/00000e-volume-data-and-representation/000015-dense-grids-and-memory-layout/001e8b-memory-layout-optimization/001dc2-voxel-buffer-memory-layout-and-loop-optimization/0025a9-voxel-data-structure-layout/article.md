# Voxel Data Structure Layout

Also called: Array2, Array2<T>, Structure Field Alignment, Voxel Field Ordering

## Overview
Voxel Data Structure Layout defines the memory organization of per-voxel attributes within a dense 3D grid, addressing bandwidth and capacity constraints by optimizing field ordering and alignment [S2]. It differs from sibling structures like [Voxel Data Layout](4076-voxel-data-layout.md) by offering variants that trade attribute richness for size, such as bit-packed formats or specific floating-point density fields [S1, unofficial][S2][S3, unofficial].

## How it works
The method organizes voxel data into compact structures where field placement determines memory efficiency and cache performance.

**Inputs and Outputs:**
- Input: A set of voxel attributes (e.g., scalar, normal, gradient) and their data types.
- Output: A contiguous memory layout (struct or array) optimized for rendering or classification access patterns.

**Data Structures:**
The core container is a 2D array abstraction `Array2<T>` which manages the underlying storage [S1, unofficial].
- **Layout:** `struct Array2<T, ArrayT=std::vector<T>>` contains fields:
  - `int ni`: Width (number of columns).
  - `int nj`: Height (number of rows).
  - `ArrayT a`: The underlying contiguous array (default `std::vector<T>`).
- **Indexing:** A 2D element at `(i, j)` maps to the 1D index `i + ni * j` in the underlying array `a` [S1, unofficial].

**Voxel Structure Definition:**
Voxels are defined as structs containing multiple fields. An example layout for a voxel includes:
- `Normal normal`: 16-bit unsigned short, used for shading.
- `Scalar scalar`: 8-bit unsigned char, the primary density/value.
- `Gradient gradient`: 8-bit unsigned char, precomputed gradient magnitude for classification [S2].

**Numbered Steps:**
1. **Define Voxel Fields:** Select the attributes required for the rendering pipeline (e.g., shading) and classification (e.g., boundary detection) [S2].
2. **Order Fields for Alignment:** Arrange fields in the struct to satisfy machine alignment restrictions. For example, place 16-bit quantities on 2-byte boundaries to avoid compiler-inserted padding [S2].
3. **Order Fields for Access Pattern:** Place fields required for the fast rendering algorithm (shading) first, followed by fields used only for classification. This allows the renderer to access only the necessary subset of data, improving cache performance [S2].
4. **Verify Size:** Use `sizeof()` to ensure the total size of the voxel struct matches expectations and contains no unexpected padding [S2].
5. **Initialize Buffer:** Create the `Array2<T>` instance with dimensions `ni` and `nj` (representing the 2D slice or flattened 3D grid dimensions) [S1, unofficial].
6. **Access Elements:** Retrieve or modify voxel data using the index formula `idx = i + ni * j` with bounds checking [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Array2:
    def __init__(self, ni: int, nj: int, data=None):
        """Initialize Array2 with dimensions ni, nj and optional data."""
        if ni < 0 or nj < 0:
            raise ValueError("Dimensions must be non-negative")
        self.ni = ni
        self.nj = nj
        if data is not None:
            self.a = list(data)
        else:
            self.a = [0] * (ni * nj)

    def resize(self, ni: int, nj: int):
        """Update ni, nj and resize the underlying array a to ni*nj."""
        if ni < 0 or nj < 0:
            raise ValueError("Dimensions must be non-negative")
        self.ni = ni
        self.nj = nj
        self.a = [0] * (ni * nj)

    def clear(self):
        """Empty a and set ni, nj to 0."""
        self.a = []
        self.ni = 0
        self.nj = 0

    def swap(self, other: 'Array2'):
        """Exchange ni, nj, and a with another Array2 instance."""
        self.ni, other.ni = other.ni, self.ni
        self.nj, other.nj = other.nj, self.nj
        self.a, other.a = other.a, self.a

    def get(self, i: int, j: int):
        """Get element at (i, j) with bounds checking."""
        if not (0 <= i < self.ni and 0 <= j < self.nj):
            raise IndexError(f"Index ({i}, {j}) out of bounds for size ({self.ni}, {self.nj})")
        return self.a[i + self.ni * j]

    def set(self, i: int, j: int, value):
        """Set element at (i, j) with bounds checking."""
        if not (0 <= i < self.ni and 0 <= j < self.nj):
            raise IndexError(f"Index ({i}, {j}) out of bounds for size ({self.ni}, {self.nj})")
        self.a[i + self.ni * j] = value

    def swap_with(self, other: 'Array2'):
        """Swap contents with another Array2."""
        self.swap(other)

# Example Voxel Struct Logic (Conceptual)
# In C++, this would be:
# struct Voxel {
#     unsigned short normal; // 2 bytes
#     unsigned char scalar;  // 1 byte
#     unsigned char gradient;// 1 byte
# }; // Total 4 bytes, aligned properly
```

**Parameters:**
- `ni`: Width of the 2D grid. Must be $\ge 0$ [S1, unofficial].
- `nj`: Height of the 2D grid. Must be $\ge 0$ [S1, unofficial].
- `T`: Data type of the elements (e.g., `float`, `unsigned char`) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Bounds Checking:** Accessing `i` or `j` outside `[0, ni)` or `[0, nj)` triggers an assertion failure or index error [S1, unofficial].
- **Alignment Padding:** Incorrect field ordering can cause the compiler to insert wasted space to satisfy alignment restrictions (e.g., 2-byte quantities aligned to 2-byte boundaries). This increases memory usage and reduces cache efficiency [S2].
- **Size Verification:** Failure to verify `sizeof(Voxel)` may lead to incorrect memory strides if padding is present [S2].

## Subtopics

### [Voxel Data Structure](3982-voxel-data-structure.md)
The Voxel Data Structure stores per-voxel attributes in a compact memory layout to address bandwidth and capacity constraints in dense volume rendering and voxel engines. It differs from sibling structures like [Voxel Data Layout](4076-voxel-data-layout.md) by offering variants that trade attribute richness for size, such as bit-packed formats or floating-point density fields.

### [Voxel Data Layout](4076-voxel-data-layout.md)
Voxel Data Layout refers to the fundamental memory organization of voxel data in a dense 3D grid, typically implemented as a contiguous array or tensor indexed by world coordinates $[x, y, z]$. It solves the problem of mapping 3D spatial positions to linear memory addresses for efficient access during rendering or processing [G]. Unlike hierarchical structures (e.g., [Voxel Data Structure](3982-voxel-data-structure.md)) which use pointers or bitmasks to skip empty space, this layout stores every voxel in a fixed-size slot, making it simple to implement but potentially memory-inefficient for sparse scenes [G]. It serves as the base layer for optimizations like [Voxel Material Packing](4907-voxel-material-packing.md) and [Voxel Renderer with Top-Level Grid and Bricks](8114-voxel-renderer-with-top-level-grid-and-bricks.md) [G].

### [Voxel Data Planar De-interleaving](6794-voxel-data-planar-de-interleaving.md)
Voxel Data Planar De-interleaving converts an Array of Structures (AoS) of voxel data into a Structure of Arrays (SoA) planar format to optimize memory access patterns and enable data reduction techniques such as slice reuse. It serves as the inverse operation of planar re-interleaving, restoring the AoS layout from planar buffers when required by rendering or simulation pipelines.

### SerializedArray Binary Layout
The binary layout for SerializedArray structures, including name, dtype, shape, and data [S3, unofficial].
- **Layout:**
  - 8 bytes: `uint64` name_size
  - N bytes: `char` name (N = name_size)
  - 4 bytes: `uint32` dtype
  - 8 bytes: `uint64` shape_dim_count
  - M*8 bytes: `uint64` shape dimensions (M = shape_dim_count)
  - 8 bytes: `uint64` data_byte_count
  - K bytes: `uint8` data (K = data_byte_count) [S3, unofficial]

## Operating characteristics
- **Time Complexity:** Element access is $O(1)$ given the index [S1, unofficial].
- **Memory Complexity:** $O(N \times M \times S)$ where $N, M$ are dimensions and $S$ is the size of the voxel struct [S1, unofficial][S2].
- **Performance:** Ordering shading fields first improves cache performance for fast rendering algorithms by allowing the renderer to skip classification fields [S2].
- **Strengths:** Simple implementation, predictable memory access, cache-friendly for dense data [S2].
- **Limitations:** Memory inefficient for sparse scenes compared to hierarchical structures [G].
- **When to Use:** Dense volume rendering where all voxels are accessed or where cache locality is critical [S2].
- **When Not to Use:** Sparse scenes where most voxels are empty, or when extreme memory compression is required [G].

## Background
The concept of structuring voxel data for efficient access originates from early volume rendering systems like VolPack, which emphasized field ordering for shading and classification [S2]. The `Array2` abstraction is commonly used in modern implementations to manage 2D slices or flattened 3D grids, providing a simple interface for memory management [S1, unofficial]. The SerializedArray binary layout is used in sparse voxel contexts for serialization [S3, unofficial].

## Sources
[S1] czq142857/NMC, https://github.com/czq142857/NMC, fetched 2026-10-26 (unofficial)
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S3] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
