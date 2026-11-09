# Voxel Data Structure Layout

Also called: Array2, Array2<T>, Structure Field Alignment, Voxel Field Ordering

## Overview
Voxel Data Structure Layout defines the memory organization of a dense 3D grid where each cell (voxel) stores scalar or vector properties such as opacity, color, or material type [S2]. It solves the problem of mapping 3D spatial coordinates to contiguous memory addresses for efficient ray-marching or surface extraction [G]. It differs from hierarchical structures like [Voxel Data Structure](3982-voxel-data-structure.md) by using a flat, fixed-size array without internal node pointers or occupancy masks, making it suitable for dense volumes or as a base representation before compression [G].

## How it works
### Inputs and Outputs
- **Inputs**: Dimensions of the grid ($n_i, n_j$) and a contiguous memory buffer for the data [S1, unofficial].
- **Outputs**: A data structure providing $O(1)$ access to any voxel at coordinates $(i, j)$ via a 1D index [S1, unofficial].

### Data Structures
The core layout is a 2D array abstraction `Array2<T>` which wraps a 1D container [S1, unofficial].

**Layout of `Array2<T>`:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
template <typename T, typename ArrayT = std::vector<T>>
struct Array2 {
    int ni; // Width (dimension i)
    int nj; // Height (dimension j)
    ArrayT a; // Contiguous storage of size ni * nj
};
```
- `ni` and `nj` are integers representing the grid dimensions [S1, unofficial].
- `a` is a contiguous array (e.g., `std::vector<T>`) storing the voxel data [S1, unofficial].
- Total capacity is $n_i \times n_j$ elements of type $T$ [S1, unofficial].

**Voxel Field Layout (Structure of Arrays vs Array of Structures):**
When storing multiple attributes per voxel (e.g., Normal, Scalar, Gradient), the layout within the element $T$ (if $T$ is a struct) or the planar arrays must respect alignment [S2].

**Example Voxel Struct (Array of Structures):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
typedef unsigned char Scalar;
typedef unsigned short Normal;
typedef unsigned char Gradient;
typedef struct {
    Normal normal;   // 16-bit
    Scalar scalar;   // 8-bit
    Gradient gradient; // 8-bit
} Voxel;
```
- **Field Ordering**: Fields required for shading (e.g., `normal`) should be placed first, followed by fields used only for classification (e.g., `gradient`) [S2]. This allows storing just the shading fields for fast rendering algorithms, saving memory and improving cache performance [S2].
- **Alignment**: Fields must be ordered to satisfy machine alignment restrictions (e.g., 16-bit quantities aligned to 2-byte boundaries) [S2]. The compiler may insert padding if fields are not ordered carefully. Use `sizeof()` to verify the struct size matches expectations [S2].

### Numbered Steps
1. **Initialization**: Construct `Array2<T>` with dimensions $n_i, n_j$. Assert $n_i \ge 0$ and $n_j \ge 0$ [S1, unofficial]. Allocate underlying array `a` of size $n_i \times n_j$ [S1, unofficial].
2. **Coordinate Mapping**: To access the element at 2D coordinates $(i, j)$, compute the 1D index:
   $$ \text{index} = i + n_i \times j $$
   This assumes row-major ordering where $i$ is the innermost dimension [S1, unofficial]. Note: The parent article [Voxel Buffer Memory Layout and Loop Optimization] specifies row-major ordering where $x$ is innermost. For a 2D slice or the `Array2` abstraction, $i$ corresponds to the innermost stride [G].
3. **Access**: Retrieve value: `val = a[index]`. Bounds checking: Assert $0 \le i < n_i$ and $0 \le j < n_j$. Accessing outside these ranges triggers an assertion failure [S1, unofficial].
4. **Resizing**: Call `resize(ni_new, nj_new)`. Update `ni` and `nj`. Resize `a` to size $n_i \times n_j$ [S1, unofficial].
5. **Clearing**: Call `clear()`. Empty array `a`. Set $n_i = 0, n_j = 0$ [S1, unofficial].
6. **Swapping**: Call `swap(Array2<T>& x)`. Exchange `ni`, `nj`, and `a` with another instance $x$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Array2:
    def __init__(self, ni: int, nj: int, data: list = None):
        assert ni >= 0 and nj >= 0, "Dimensions must be non-negative"
        self.ni = ni
        self.nj = nj
        if data is not None:
            assert len(data) == ni * nj, "Data length must match dimensions"
            self.a = list(data)
        else:
            self.a = [None] * (ni * nj)

    def resize(self, ni: int, nj: int):
        assert ni >= 0 and nj >= 0
        self.ni = ni
        self.nj = nj
        self.a = [None] * (ni * nj)

    def clear(self):
        self.a = []
        self.ni = 0
        self.nj = 0

    def swap(self, other: 'Array2'):
        self.ni, other.ni = other.ni, self.ni
        self.nj, other.nj = other.nj, self.nj
        self.a, other.a = other.a, self.a

    def get(self, i: int, j: int):
        assert 0 <= i < self.ni, f"i out of bounds: {i} not in [0, {self.ni})"
        assert 0 <= j < self.nj, f"j out of bounds: {j} not in [0, {self.nj})"
        index = i + self.ni * j
        return self.a[index]

    def set(self, i: int, j: int, value):
        assert 0 <= i < self.ni, f"i out of bounds: {i} not in [0, {self.ni})"
        assert 0 <= j < self.nj, f"j out of bounds: {j} not in [0, {self.nj})"
        index = i + self.ni * j
        self.a[index] = value
```

### Parameters and Typical Values
- **Data Types**: Common typedefs include `Array2d` (double), `Array2f` (float), `Array2i` (int), `Array2ui` (unsigned int), `Array2s` (short), `Array2us` (unsigned short), `Array2c` (char), `Array2uc` (unsigned char), `Array2ll` (long long), `Array2ull` (unsigned long long) [S1, unofficial].
- **Voxel Size**: In the VolPack example, a Voxel struct is 32 bits (4 bytes) if packed tightly, but may be larger due to alignment padding depending on the platform [S2].

### Edge Cases and Pitfalls
- **Alignment Padding**: If fields are not ordered correctly, the compiler may insert padding bytes. For example, a struct with `unsigned short` (2 bytes) followed by `unsigned char` (1 byte) might be padded to 4 bytes total on some architectures. Always verify `sizeof(Voxel)` [S2].
- **Bounds Checking**: Accessing $i \ge n_i$ or $j \ge n_j$ causes assertion failure [S1, unofficial].

## Subtopics

### [Voxel Data Structure](3982-voxel-data-structure.md)
The Voxel Data Structure stores per-voxel attributes in a compact memory layout to address bandwidth and capacity constraints in dense volume rendering and voxel engines. It differs from sibling structures like [Voxel Data Layout](4076-voxel-data-layout.md) by offering variants that trade attribute richness for size, such as bit-packed formats or floating-point density fields.

### [Voxel Data Layout](4076-voxel-data-layout.md)
Voxel Data Layout refers to the fundamental memory organization of voxel data in a dense 3D grid, typically implemented as a contiguous array or tensor indexed by world coordinates $[x, y, z]$. It solves the problem of mapping 3D spatial positions to linear memory addresses for efficient access during rendering or processing. Unlike hierarchical structures (e.g., [Voxel Data Structure](3982-voxel-data-structure.md)) which use pointers or bitmasks to skip empty space, this layout stores every voxel in a fixed-size slot, making it simple to implement but potentially memory-inefficient for sparse scenes. It serves as the base layer for optimizations like [Voxel Material Packing](4907-voxel-material-packing.md) and [Voxel Renderer with Top-Level Grid and Bricks](8114-voxel-renderer-with-top-level-grid-and-bricks.md).

### SerializedArray Binary Layout
The binary layout for SerializedArray structures, including name, dtype, shape, and data [S3, unofficial].
- **Layout**:
  1. 8 bytes: `uint64` `name_size`
  2. N bytes: `char` `name` (where $N = \text{name\_size}$)
  3. 4 bytes: `uint32` `dtype`
  4. 8 bytes: `uint64` `shape_dim_count`
  5. $M \times 8$ bytes: `uint64` `shape` dimensions (where $M = \text{shape\_dim\_count}$)
  6. 8 bytes: `uint64` `data_byte_count`
  7. K bytes: `uint8` `data` (where $K = \text{data\_byte\_count}$) [S3, unofficial]

### Voxel Data Planar De-interleaving
Converts an Array of Structures (AoS) of `FVoxelData` into a Structure of Arrays (SoA) planar format for compression, and vice versa [S4, unofficial].
- **AoS to SoA (De-interleaving)**:
  - Input: `TArray<FVoxelData> Data` of size $N$.
  - Output: `TArray<uint8> OutPlanes` of size $N \times 4$.
  - Layout: First $N$ bytes are MaterialID, next $N$ are Density, next $N$ are BiomeID, last $N$ are Metadata [S4, unofficial].
  - Steps:
    1. Set $N = \text{Data.Num()}$ [S4, unofficial].
    2. Reinterpret `Data` as `uint8* Src` [S4, unofficial].
    3. Set pointers `Mat`, `Den`, `Bio`, `Meta` in `OutPlanes` to the start of each plane [S4, unofficial].
    4. Loop $i$ from $0$ to $N-1$:
       - `Mat[i] = Src[i*4 + 0]`
       - `Den[i] = Src[i*4 + 1]`
       - `Bio[i] = Src[i*4 + 2]`
       - `Meta[i] = Src[i*4 + 3]` [S4, unofficial]
- **SoA to AoS (Interleaving)**:
  - Input: `uint8* Planes` (Mat, Den, Bio, Meta contiguous blocks of `NumVoxels` size) [S4, unofficial].
  - Output: `TArray<FVoxelData> OutData` [S4, unofficial].
  - Steps:
    1. Set `Mat`, `Den`, `Bio`, `Meta` pointers [S4, unofficial].
    2. Reinterpret `OutData` as `uint8* Dst` [S4, unofficial].
    3. Loop $i$ from $0$ to `NumVoxels-1`:
       - `Dst[i*4 + 0] = Mat[i]`
       - `Dst[i*4 + 1] = Den[i]`
       - `Dst[i*4 + 2] = Bio[i]`
       - `Dst[i*4 + 3] = Meta[i]` [S4, unofficial]

## Operating characteristics
- **Time Complexity**: Access is $O(1)$ given valid coordinates. Resizing is $O(n_i \times n_j)$ due to reallocation [S1, unofficial].
- **Memory Complexity**: $O(n_i \times n_j \times \text{sizeof}(T))$. For the VolPack Voxel example, this is 4 bytes per voxel if tightly packed, but potentially more with alignment padding [S2].
- **Performance**: Cache performance is improved by ordering fields for shading first, allowing smaller subsets of data to be loaded for rendering [S2]. Planar de-interleaving (SoA) improves compression efficiency and vectorization potential compared to AoS [S4, unofficial].
- **Strengths**: Simple implementation, low overhead, predictable memory usage, efficient for dense volumes.
- **Limitations**: Not suitable for sparse volumes (wastes memory on empty space). Alignment padding can increase memory usage. Bounds checking overhead if not optimized out.

## Background
The `Array2` abstraction and its variants are commonly used in graphics programming for 2D slices of 3D data or 2D textures [S1, unofficial]. The VolPack User's Guide provides specific recommendations for voxel field ordering to optimize shading and classification performance [S2].

## Sources
[S1] czq142857/NMC, https://github.com/czq142857/NMC, fetched 2026-10-26 (unofficial)
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S3] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
[S4] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
