# Voxel Data Structure Layout

Also called: Array2, Array2<T>, Structure Field Alignment, Voxel Field Ordering

## Overview
Voxel Data Structure Layout defines the memory organization of individual voxel elements within a dense grid, solving the problem of efficient storage and retrieval of per-voxel attributes such as density, color, normals, and occupancy [S1, unofficial][S2][S3, unofficial]. It differs from sibling structures like [Voxel Data Layout](4076-voxel-data-layout.md) (which focuses on array indexing and coordinate mapping) and [Paged Block Array](12102-paged-block-array.md) (which manages chunk allocation) by defining the actual payload content and memory footprint of a single voxel element [S1, unofficial][S2]. It serves as the base layer for [Voxel Material Packing](4907-voxel-material-packing.md) and [Voxel Representation with Gaussian Lobes](4128-voxel-representation-with-gaussian-lobes.md) [G].

## How it works
**Inputs and Outputs:**
- Input: A set of voxel attributes (e.g., scalar value, normal vector, gradient magnitude) and their data types.
- Output: A contiguous memory block representing a 2D or 3D grid of voxels, where each voxel is an instance of a defined structure.

**Data Structures:**
The fundamental container is a 2D array structure `Array2<T>`, which wraps a 1D contiguous array. The layout is defined as follows [S1, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
template <typename T, typename ArrayT = std::vector<T>>
struct Array2 {
    int ni; // Width (number of columns)
    int nj; // Height (number of rows)
    ArrayT a; // Contiguous storage for elements
};
```

For a specific voxel payload, the structure `Voxel` is defined by the fields required for rendering and classification [S2]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
typedef unsigned char Scalar;
typedef unsigned short Normal;
typedef unsigned char Gradient;

typedef struct {
    Normal normal;   // 16-bit surface normal
    Scalar scalar;   // 8-bit scalar value (e.g., density)
    Gradient gradient; // 8-bit gradient magnitude
} Voxel;
```

**Indexing Formula:**
Elements in the `Array2` are mapped to the 1D array `a` using row-major ordering. The 2D element at coordinates `(i, j)` maps to the 1D index `idx` [S1, unofficial]:

$$
idx = i + ni \times j
$$

Where:
- $i$ is the column index ($0 \le i < ni$).
- $j$ is the row index ($0 \le j < nj$).
- $ni$ is the width of the array.

**Step-by-Step Implementation:**

1.  **Define Voxel Structure:** Create a `struct` containing all per-voxel attributes. Order fields to satisfy machine alignment restrictions (e.g., align 16-bit fields to 2-byte boundaries) and place shading-critical fields first for cache efficiency [S2].
2.  **Initialize Container:** Instantiate `Array2<T>` with dimensions $ni$ and $nj$. Assert that $ni \ge 0$ and $nj \ge 0$ [S1, unofficial].
3.  **Allocate Memory:** Resize the underlying array `a` to size $ni \times nj$ [S1, unofficial].
4.  **Access Element:** To access voxel $(i, j)$, calculate the linear index $idx = i + ni \times j$. Perform bounds checking: assert $0 \le i < ni$ and $0 \le j < nj$ [S1, unofficial].
5.  **Verify Size:** Use `sizeof(Voxel)` to ensure the structure size matches expectations, accounting for any compiler-inserted padding [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Array2:
    def __init__(self, ni: int, nj: int, data_type: type):
        """
        Initialize a 2D voxel array.
        :param ni: Width (number of columns)
        :param nj: Height (number of rows)
        :param data_type: The type of voxel data (e.g., Voxel struct)
        """
        assert ni >= 0 and nj >= 0, "Dimensions must be non-negative"
        self.ni = ni
        self.nj = nj
        self.a = [None] * (ni * nj)

    def resize(self, ni: int, nj: int):
        """Update dimensions and resize the underlying array."""
        assert ni >= 0 and nj >= 0
        self.ni = ni
        self.nj = nj
        self.a = [None] * (ni * nj)

    def clear(self):
        """Empty the array and reset dimensions."""
        self.a = []
        self.ni = 0
        self.nj = 0

    def get(self, i: int, j: int):
        """
        Get the voxel at (i, j).
        :param i: Column index
        :param j: Row index
        :return: The voxel data
        """
        assert 0 <= i < self.ni, f"i out of bounds: {i} not in [0, {self.ni})"
        assert 0 <= j < self.nj, f"j out of bounds: {j} not in [0, {self.nj})"
        idx = i + self.ni * j
        return self.a[idx]

    def set(self, i: int, j: int, value):
        """
        Set the voxel at (i, j).
        :param i: Column index
        :param j: Row index
        :param value: The voxel data to store
        """
        assert 0 <= i < self.ni, f"i out of bounds: {i} not in [0, {self.ni})"
        assert 0 <= j < self.nj, f"j out of bounds: {j} not in [0, {self.nj})"
        idx = i + self.ni * j
        self.a[idx] = value

    def swap(self, other: 'Array2'):
        """Exchange contents with another Array2 instance."""
        self.ni, other.ni = other.ni, self.ni
        self.nj, other.nj = other.nj, self.nj
        self.a, other.a = other.a, self.a

# Example Voxel Structure Definition
class Voxel:
    def __init__(self, normal: int, scalar: int, gradient: int):
        self.normal = normal    # 16-bit
        self.scalar = scalar    # 8-bit
        self.gradient = gradient # 8-bit

    def __repr__(self):
        return f"Voxel(n={self.normal}, s={self.scalar}, g={self.gradient})"
```

**Parameters and Typical Values:**
- `ni`, `nj`: Grid dimensions. Typical values depend on scene resolution (e.g., 128x128, 256x256).
- `Voxel` field sizes: `Normal` (16-bit), `Scalar` (8-bit), `Gradient` (8-bit) [S2].

**Edge Cases and Pitfalls:**
- **Alignment Padding:** Compilers may insert padding between structure fields to satisfy alignment requirements (e.g., 16-bit fields aligned to 2-byte boundaries). Use `sizeof()` to verify the actual memory footprint [S2].
- **Bounds Checking:** Accessing indices outside $[0, ni)$ or $[0, nj)$ triggers assertion failures or undefined behavior if not checked [S1, unofficial].
- **Field Ordering:** Placing non-shading fields before shading fields reduces cache performance for fast rendering algorithms. Shading fields should be placed first [S2].

## Subtopics

### [Voxel Data Structure](3982-voxel-data-structure.md)
The Voxel Data Structure is the fundamental memory unit for storing per-voxel attributes in dense grid-based volume rendering and voxel engines. It solves the problem of efficient storage and retrieval of geometric and material properties (such as density, color, normals, and occupancy) for individual voxels, which are often the primary memory bottleneck in large-scale volumetric scenes. It differs from sibling structures like [Voxel Data Layout](4076-voxel-data-layout.md) (which focuses on array indexing) and [Paged Block Array](12102-paged-block-array.md) (which manages chunk allocation) by defining the actual payload content and memory footprint of a single voxel element.

### [Voxel Data Layout](4076-voxel-data-layout.md)
Voxel Data Layout defines the fundamental memory organization for dense voxel grids, mapping 3D spatial coordinates to a linear or tensor-based storage structure. It solves the problem of efficient spatial indexing and boundary validation in GPU or CPU-based volume rendering by replacing multi-dimensional array overhead with linear memory access. Unlike hierarchical approaches like [Paged Block Array](12102-paged-block-array.md) which optimize for sparse data, or [Voxel Data Planar De-interleaving](6794-voxel-data-planar-de-interleaving.md) which optimizes for cache locality, this method focuses on the raw data representation and coordinate mapping. It serves as the base layer for [Voxel Material Packing](4907-voxel-material-packing.md) and [Voxel Representation with Gaussian Lobes](4128-voxel-representation-with-gaussian-lobes.md) [G].

### [Voxel Data Planar De-interleaving](6794-voxel-data-planar-de-interleaving.md)
Voxel Data Planar De-interleaving converts an Array of Structures (AoS) of voxel data into a Structure of Arrays (SoA) planar format for compression and data reduction. It serves as the inverse operation of planar re-interleaving, restoring the AoS layout from planar buffers when required by downstream systems. This method builds on [Voxel Data Structure Layout](3584-voxel-data-structure-layout.md).

### [Paged Block Array](12102-paged-block-array.md)
The Paged Block Array is a dynamic array implementation that stores elements in fixed-size blocks (pages) to optimize memory allocation and cache locality, avoiding the need for contiguous reallocation of the entire array. It differs from standard dynamic arrays by using fixed-size blocks instead of contiguous reallocation, reducing allocation overhead and fragmentation at the cost of slightly higher pointer indirection and fixed block sizes.

### [Voxel Memory Layout & Stride Calculation](13519-voxel-memory-layout-stride-calculation.md)
This method defines the mapping between 3D voxel coordinates $(i, j, k)$ and their linear offset in a contiguous 1D memory buffer, enabling efficient random access to volume data. It establishes the row-major storage order where the $i$-coordinate varies fastest, followed by $j$, then $k$, which is the standard layout for dense grid representations. Unlike planar de-interleaving which separates channels, this method assumes a single contiguous block of memory for the scalar or vector field values.

### SerializedArray Binary Layout
The binary layout for SerializedArray structures, including name, dtype, shape, and data [S3, unofficial].
**Layout:**
- 8 bytes: `uint64` name_size
- N bytes: `char` name (where N = name_size)
- 4 bytes: `uint32` dtype
- 8 bytes: `uint64` shape_dim_count
- M*8 bytes: `uint64` shape dimensions (where M = shape_dim_count)
- 8 bytes: `uint64` data_byte_count
- K bytes: `uint8` data (where K = data_byte_count) [S3, unofficial]

### Buffer View Linkage
The `voxelVolume.primitives.bufferView` field holds an integer index pointing into a separate `bufferViews` array. Each entry in `bufferViews` contains an index into a `buffers` array, which ultimately stores the raw voxel data bytes [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Element access is $O(1)$ given valid coordinates, involving a single multiplication and addition for index calculation [S1, unofficial].
- **Memory Complexity:** $O(ni \times nj \times \text{sizeof(Voxel)})$ for a 2D array. For 3D, it scales with the total voxel count [S1, unofficial][S2].
- **Performance:** Placing shading-critical fields first improves cache performance for rendering algorithms [S2]. Proper alignment minimizes padding waste [S2].
- **Strengths:** Simple implementation, efficient random access, cache-friendly for contiguous traversal.
- **Limitations:** Fixed size requires pre-allocation; resizing can be costly. Not suitable for highly sparse data without additional sparse indexing structures.
- **When to Use:** Dense voxel grids where random access is frequent and memory overhead of sparsity structures is unjustified.
- **When Not to Use:** Highly sparse volumes where [Paged Block Array](12102-paged-block-array.md) or hierarchical structures are more efficient.

## Background
The concept of structured voxel arrays originates from early volume rendering systems like VolPack, which standardized the layout of voxel data for efficient shading and classification [S2]. The `Array2` template structure is commonly used in C++ implementations for its simplicity and compatibility with standard libraries like `std::vector` [S1, unofficial]. Modern extensions include binary serialization formats for network transmission and GPU buffer linkage via formats like OpenVoxel [S3, S4, unofficial].

## Sources
[S1] czq142857/NMC, https://github.com/czq142857/NMC, fetched 2026-10-26 (unofficial)
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S3] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
[S4] Voxel data format, https://github.com/enkisoftware/OpenVoxelFileFormat/issues/4, fetched 2026-10-27 (unofficial)
