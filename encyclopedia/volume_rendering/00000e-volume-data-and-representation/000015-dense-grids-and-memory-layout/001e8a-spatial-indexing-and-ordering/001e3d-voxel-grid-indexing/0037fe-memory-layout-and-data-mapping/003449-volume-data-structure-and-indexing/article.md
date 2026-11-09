# Volume Data Structure and Indexing

Also called: IJK to RAS coordinate mapping, Volume class

## Overview
This method defines the core data structure for storing and accessing 3D scalar volume data in a dense grid format, specifically implementing the mapping between voxel indices (IJK) and physical world coordinates (RAS). It solves the problem of efficiently storing volumetric data in contiguous memory while providing the necessary geometric metadata to transform topological indices into physical space. It differs from sparse representations (like [Multi-Material Cell Map Indexing & Access](4886-multi-material-cell-map-indexing-access.md)) by using a fixed-size dense array, which simplifies indexing but requires explicit bounds checking and memory management by the caller.

## How it works
**Inputs and Outputs:**
*   **Input:** A typed array buffer (e.g., `Float32Array`, `Uint8Array`) containing voxel values, and dimensions $N_x, N_y, N_z$.
*   **Output:** A `Volume` object instance that provides indexed access to data and coordinate transformations.

**Data Structures:**
The `Volume` object maintains the following fields:
*   `data`: The underlying `TypedArray` holding the scalar values.
*   `xLength`, `yLength`, `zLength`: Integers representing the grid dimensions $N_x, N_y, N_z$.
*   `spacing`: An array of 3 numbers `[sx, sy, sz]`, default `[1, 1, 1]`, representing voxel dimensions in physical units.
*   `offset`: An array of 3 numbers `[ox, oy, oz]`, default `[0, 0, 0]`, representing the physical origin.
*   `matrix`: A $3 \times 3$ matrix (identity by default) for rotation/scaling.
*   `inverseMatrix`: The inverse of the `matrix`.
*   `segmentation`: A boolean flag, default `false`, indicating if the data represents segmentation labels.
*   `sliceList`: An array of `VolumeSlice` objects associated with this volume.
*   `min`, `max`: Numbers representing the minimum and maximum values in the data (computed on construction or update).
*   `lowerThreshold`, `upperThreshold`: Numbers for filtering voxels in slices. Default `-Infinity` and `Infinity` respectively. Changing these triggers `geometryNeedsUpdate = true` on all associated slices.

**Indexing Formulas:**
The storage follows row-major layout where the $i$-axis (x) is the innermost dimension, $j$ (y) is middle, and $k$ (z) is outermost [S1, unofficial].

1.  **Forward Index (IJK to Linear):**
    Given voxel coordinates $(i, j, k)$ where $0 \le i < N_x$, $0 \le j < N_y$, $0 \le k < N_z$:
    $$
    idx = k \cdot N_x \cdot N_y + j \cdot N_x + i
    $$
    The value is accessed as `data[idx]` [S1, unofficial].

2.  **Reverse Index (Linear to IJK):**
    Given a linear index `idx`:
    $$
    k = \lfloor idx / (N_x \cdot N_y) \rfloor
    $$
    $$
    j = \lfloor (idx \pmod{N_x \cdot N_y}) / N_x \rfloor
    $$
    $$
    i = idx \pmod{N_x}
    $$
    [S1, unofficial].

**Coordinate Transformation (IJK to RAS):**
To map voxel index $(i, j, k)$ to physical RAS coordinates $(x, y, z)$:
1.  Apply spacing: $v_x = i \cdot s_x, v_y = j \cdot s_y, v_z = k \cdot s_z$.
2.  Apply rotation/scaling via `matrix` $M$:
    $$
    \begin{bmatrix} x' \\ y' \\ z' \end{bmatrix} = M \begin{bmatrix} v_x \\ v_y \\ v_z \end{bmatrix}
    $$
3.  Apply offset:
    $$
    \begin{bmatrix} x \\ y \\ z \end{bmatrix} = \begin{bmatrix} x' \\ y' \\ z' \end{bmatrix} + \begin{bmatrix} o_x \\ o_y \\ o_z \end{bmatrix}
    $$
    [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Volume:
    def __init__(self, data: TypedArray, x_length: int, y_length: int, z_length: int,
                 spacing: List[float] = [1.0, 1.0, 1.0],
                 offset: List[float] = [0.0, 0.0, 0.0],
                 matrix: Matrix3 = IdentityMatrix3()):
        """
        Initialize Volume object.
        Args:
            data: TypedArray (Float32Array, Uint8Array, etc.)
            x_length, y_length, z_length: Dimensions Nx, Ny, Nz
            spacing: [sx, sy, sz]
            offset: [ox, oy, oz]
            matrix: 3x3 rotation/scaling matrix
        """
        # Validate buffer size
        expected_size = x_length * y_length * z_length
        if len(data) != expected_size:
            raise Error("Error in Volume constructor, lengths are not matching arrayBuffer size")
        
        # Validate type (JS TypedArrays do not support 64-bit ints)
        if data.constructor.name in ["BigInt64Array", "BigUint64Array"]:
            raise Error("Error in Volume constructor : this type is not supported in JavaScript")

        self.data = data
        self.xLength = x_length
        self.yLength = y_length
        self.zLength = z_length
        self.spacing = spacing
        self.offset = offset
        self.matrix = matrix
        self.inverseMatrix = matrix.inverse()
        self.segmentation = False
        self.sliceList = []
        self.lowerThreshold = -Infinity
        self.upperThreshold = Infinity
        
        # Compute min/max, ignoring NaN
        self.min = Infinity
        self.max = -Infinity
        for val in data:
            if not isnan(val):
                if val < self.min: self.min = val
                if val > self.max: self.max = val

    def get_linear_index(self, i: int, j: int, k: int) -> int:
        """Calculate linear index from IJK coordinates."""
        return k * self.xLength * self.yLength + j * self.xLength + i

    def get_ijk_from_index(self, idx: int) -> Tuple[int, int, int]:
        """Calculate IJK coordinates from linear index."""
        k = floor(idx / (self.xLength * self.yLength))
        remainder = idx % (self.xLength * self.yLength)
        j = floor(remainder / self.xLength)
        i = remainder % self.xLength
        return (i, j, k)

    def get_ras_from_ijk(self, i: int, j: int, k: int) -> Vector3:
        """Transform IJK voxel index to RAS physical coordinates."""
        # 1. Apply spacing
        vx = i * self.spacing[0]
        vy = j * self.spacing[1]
        vz = k * self.spacing[2]
        
        # 2. Apply matrix (rotation/scaling)
        # Assuming matrix is 3x3 stored as row-major or column-major consistent with math library
        # Here we assume standard vector-matrix multiplication M * v
        v_vec = Vector3(vx, vy, vz)
        transformed = self.matrix.multiply_vector(v_vec)
        
        # 3. Apply offset
        return Vector3(
            transformed.x + self.offset[0],
            transformed.y + self.offset[1],
            transformed.z + self.offset[2]
        )

    def map(self, callback: Callable[[float, int, int, int], float]):
        """Apply a callback to every voxel in place.
        callback(value, i, j, k) -> new_value
        """
        for k in range(self.zLength):
            for j in range(self.yLength):
                for i in range(self.xLength):
                    idx = self.get_linear_index(i, j, k)
                    val = self.data[idx]
                    new_val = callback(val, i, j, k)
                    self.data[idx] = new_val
```

**Edge Cases and Pitfalls:**
*   **Buffer Size Mismatch:** If the input `arrayBuffer` size does not equal $N_x \cdot N_y \cdot N_z$, an error is thrown [S1, unofficial].
*   **Unsupported Types:** JavaScript `TypedArray` does not support 64-bit integers (`int64`/`uint64`). Attempting to use these types throws an error [S1, unofficial].
*   **NaN Handling:** Min/max calculations must explicitly ignore `NaN` values to avoid incorrect bounds [S1, unofficial].
*   **Threshold Updates:** Changing `lowerThreshold` or `upperThreshold` requires setting `geometryNeedsUpdate = true` on all associated `VolumeSlice` objects to ensure visual consistency [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Indexing is $O(1)$. Computing min/max is $O(N_x N_y N_z)$. Coordinate transformation is $O(1)$ (constant number of arithmetic operations).
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the data array. Additional overhead for metadata (spacing, offset, matrix) is $O(1)$.
*   **Strengths:** Simple, cache-friendly contiguous memory layout; fast linear indexing; straightforward coordinate transformation pipeline.
*   **Limitations:** Dense storage is inefficient for sparse data; no built-in compression; bounds checking is the caller's responsibility.
*   **When to use:** Ideal for dense volumetric data where random access is frequent and memory bandwidth is a concern. Not suitable for highly sparse volumes where sparse structures (e.g., octrees) would be more efficient.

## Background
This structure is derived from the `Volume` class implementation in the `webgl_volume_demo` project by PeterC-DLS [S1, unofficial]. It reflects common practices in WebGL-based volume rendering where typed arrays are used for efficient GPU/CPU data transfer. The IJK-to-RAS mapping is a standard convention in medical imaging (DICOM), adapted here for general 3D scalar field visualization.

## Sources
[S1] webgl_volume_demo, https://github.com/PeterC-DLS/webgl_volume_demo, fetched 2026-10-27 (unofficial)
