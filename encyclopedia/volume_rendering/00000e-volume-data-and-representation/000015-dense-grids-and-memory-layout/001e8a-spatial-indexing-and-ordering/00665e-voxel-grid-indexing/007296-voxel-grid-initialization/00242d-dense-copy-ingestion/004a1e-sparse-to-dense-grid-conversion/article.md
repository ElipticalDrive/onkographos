# Sparse-to-Dense Grid Conversion
Also called: VTK export
## Overview
This method converts a sparse OpenVDB volume grid into a dense, contiguous float array with `LayoutXYZ` ordering, then exports it as a legacy VTK `STRUCTURED_POINTS` binary file [S1, unofficial]. It bridges memory-efficient sparse representations and renderers requiring dense, contiguous memory layouts [G].
## How it works
**Inputs:**
- A sparse OpenVDB grid [S1, unofficial].
- A transformation matrix defining the grid's world-space position and scale [G].
- The desired `volume_size` (dimensions of the output dense grid) [S1, unofficial].
- The `center` (world-space center of the volume) [S1, unofficial].

**Outputs:**
- A dense, contiguous float array with `LayoutXYZ` ordering [S1, unofficial].
- A legacy VTK `STRUCTURED_POINTS` binary file [S1, unofficial].

**Data Structures:**
- **Sparse Grid:** OpenVDB `Grid` object containing active voxels [G].
- **Dense Grid:** A dense 3D array with dimensions `volume_size` [G].
- **VTK File:** Binary file containing a text header and raw float data [G].

**Steps:**
1. **Compute Dense Grid Bounds:** Calculate the minimum and maximum world-space coordinates for the dense grid:
   $$v_{min} = center - \frac{volume\_size}{2}$$
   $$v_{max} = center + \frac{volume\_size}{2}$$
   Create a dense grid using `transform->worldToIndexCellCentered` or `worldToIndexNodeCentered` depending on the grid's centering [S1, unofficial].

2. **Copy Sparse to Dense:** Transfer active voxel values from the sparse grid into the corresponding positions in the dense array using `openvdb::tools::copyToDense` [S1, unofficial].

3. **Convert Byte Order:** Convert the dense float data to big-endian byte order to ensure cross-platform compatibility for the VTK binary format [S1, unofficial]. This conversion is performed in parallel using TBB `blocked_range` [S1, unofficial].

4. **Export VTK File:** Write the big-endian float array to a file using the legacy VTK `STRUCTURED_POINTS` format [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct
from concurrent.futures import ThreadPoolExecutor

def sparse_to_dense_vtk(sparse_grid, transform, volume_size, center):
    # Step 1: Compute bounds and create dense grid
    vmin = center - volume_size / 2
    vmax = center + volume_size / 2
    dense_size = volume_size ** 3
    dense_grid = [0.0] * dense_size

    # Step 2: Copy sparse to dense using OpenVDB's tool
    # openvdb::tools::copyToDense maps active voxels to the dense grid
    # based on the transform's worldToIndexCellCentered/NodeCentered logic
    copy_to_dense_impl(sparse_grid, dense_grid, transform)

    # Step 3: Convert to big-endian byte order in parallel
    # Simulates TBB blocked_range parallelization
    def convert_chunk(start, end):
        for i in range(start, end):
            # Pack as little-endian float, unpack as big-endian float
            dense_grid[i] = struct.unpack('>f', struct.pack('<f', dense_grid[i]))[0]

    num_threads = 4
    chunk_size = dense_size // num_threads
    with ThreadPoolExecutor(max_workers=num_threads) as executor:
        futures = []
        for i in range(num_threads):
            start = i * chunk_size
            end = start + chunk_size if i < num_threads - 1 else dense_size
            futures.append(executor.submit(convert_chunk, start, end))
        for f in futures:
            f.result()

    # Step 4: Export to VTK STRUCTURED_POINTS binary file
    with open("output.vtk", "wb") as f:
        f.write(b"# vtk DataFile Version 2.0\n")
        f.write(b"Sparse to Dense Export\n")
        f.write(b"BINARY\n")
        f.write(b"DATASET STRUCTURED_POINTS\n")
        f.write(f"DIMENSIONS {volume_size} {volume_size} {volume_size}\n".encode())
        f.write(f"ORIGIN {vmin[0]} {vmin[1]} {vmin[2]}\n".encode())
        f.write(b"POINT_DATA " + str(dense_size).encode() + b"\n")
        f.write(b"SCALARS density float 1\n")
        f.write(b"LOOKUP_TABLE default\n")
        for val in dense_grid:
            f.write(struct.pack('>f', val))

def copy_to_dense_impl(sparse_grid, dense_grid, transform):
    # Implements openvdb::tools::copyToDense behavior
    for voxel in sparse_grid.active_voxels():
        idx = transform.world_to_index(voxel.world_pos)
        if 0 <= idx < len(dense_grid):
            dense_grid[idx] = voxel.value
```

**Parameters:**
- `volume_size`: Dimensions of the output dense grid [S1, unofficial].
- `center`: World-space center of the volume [S1, unofficial].
- `transform`: OpenVDB transformation matrix defining the grid's world-space position and scale [G].

**Edge Cases and Pitfalls:**
- **Boundaries:** Ensure that the `volume_size` and `center` are chosen such that the sparse grid's active voxels fall within the dense grid's bounds [G].
- **Precision:** Floating-point precision must be preserved during the big-endian byte order conversion [G].
- **Empty Input:** If the sparse grid contains no active voxels, the dense grid will be filled with the background value [G].

## Subtopics          

### [OpenVDB to Yocto Volume Conversion](14106-openvdb-to-yocto-volume-conversion.md)
Converts sparse OpenVDB grids to dense Yocto `.vol` files. Choose this when targeting the Yocto/Quasar renderer ecosystem. It handles coordinate translation to `(0,0,0)` and uses the `yovdbload` toolchain, unlike the VTK export [G].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of active voxels in the sparse grid. The copy operation and byte order conversion are linear in the number of active voxels [G].
- **Memory Complexity:** $O(V)$, where $V$ is the total number of voxels in the dense grid ($volume\_size^3$). The dense grid requires contiguous memory for all voxels, regardless of sparsity [G].
- **Measured Performance:** Not given in the sources [G].
- **Strengths:** Simple, efficient conversion from sparse to dense; produces a widely supported VTK format; parallel byte order conversion improves performance [G].
- **Limitations:** High memory usage for large dense grids; not suitable for highly sparse data where memory efficiency is critical; VTK `STRUCTURED_POINTS` is a legacy format [G].
- **When to Use:** When a dense, contiguous memory layout is required for rendering or further processing, and the VTK format is acceptable [G].
- **When Not to Use:** When memory efficiency is paramount, or when the target renderer requires a different format (e.g., Yocto `.vol`) [G].

## Background
Not given in the sources.

## Sources
[S1] zzilch/Mesh2Volume, https://github.com/zzilch/Mesh2Volume, fetched 2026-10-26 (unofficial)
