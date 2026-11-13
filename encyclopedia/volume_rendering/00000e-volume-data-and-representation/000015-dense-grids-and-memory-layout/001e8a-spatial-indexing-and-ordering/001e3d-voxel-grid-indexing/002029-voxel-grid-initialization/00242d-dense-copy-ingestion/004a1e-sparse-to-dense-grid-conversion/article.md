# Sparse-to-Dense Grid Conversion
Also called: VTK export

## Overview
This method converts a sparse OpenVDB volume grid into a dense, contiguous float array with `LayoutXYZ` ordering, subsequently exporting the result as a legacy VTK `STRUCTURED_POINTS` binary file [S1, unofficial]. It bridges the gap between memory-efficient sparse representations and renderers requiring dense, contiguous memory layouts. Unlike its parent [Dense Copy Ingestion](3212-dense-copy-ingestion.md), which ingests dense arrays into sparse structures, this method performs the inverse transformation for export. It differs from child [OpenVDB to Yocto Volume Conversion](14106-openvdb-to-yocto-volume-conversion.md) by targeting the legacy VTK format rather than the Yocto `.vol` format, and by not explicitly handling the coordinate translation to `(0,0,0)` required by the Yocto ecosystem [S1, unofficial].

## How it works
**Inputs:**
- A sparse OpenVDB grid (typically containing density and temperature fields).
- A transformation matrix defining the grid's world-space position and scale.
- The desired `volume_size` (dimensions of the output dense grid).
- The `center` (world-space center of the volume).

**Outputs:**
- A dense, contiguous float array with `LayoutXYZ` ordering.
- A legacy VTK `STRUCTURED_POINTS` binary file.

**Data Structures:**
- **Sparse Grid:** OpenVDB `Grid` object with active voxels.
- **Dense Grid:** A dense 3D array (e.g., `openvdb::math::DenseMatrix` or similar dense container) with dimensions `volume_size`.
- **VTK File:** Binary file containing header and raw float data.

**Steps:**
1. **Compute Dense Grid Bounds:**
   Calculate the minimum and maximum world-space coordinates for the dense grid:
   $$v_{min} = center - \frac{volume\_size}{2}$$
   $$v_{max} = center + \frac{volume\_size}{2}$$
   Create a dense grid representation using the transformation matrix to map world coordinates to indices. Use `transform->worldToIndexCellCentered` or `worldToIndexNodeCentered` depending on the grid's centering [S1, unofficial].

2. **Copy Sparse to Dense:**
   Populate the dense grid with values from the sparse grid. Use `openvdb::tools::copyToDense` to efficiently transfer active voxel values into the corresponding positions in the dense array. Background values are filled with the grid's background value [S1, unofficial].

3. **Convert Byte Order:**
   Convert the dense float data to big-endian byte order to ensure cross-platform compatibility for the VTK binary format. This conversion is performed in parallel using TBB `blocked_range` for performance [S1, unofficial].

4. **Export VTK File:**
   Write the big-endian float array to a file using the legacy VTK `STRUCTURED_POINTS` format. This includes writing the appropriate VTK header followed by the raw binary data [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import openvdb
import openvdb.tools
import numpy as np
import struct
import tbb

def sparse_to_dense_vtk(sparse_grid, transform, volume_size, center):
    # Step 1: Compute bounds and create dense grid
    vmin = center - volume_size / 2
    vmax = center + volume_size / 2
    
    # Create a dense grid with the specified size
    # The transform maps world coordinates to dense grid indices
    dense_grid = openvdb.math.DenseMatrix(volume_size, volume_size, volume_size)
    
    # Map world coordinates to indices
    # Note: worldToIndexCellCentered or worldToIndexNodeCentered depends on grid centering
    index_transform = transform.worldToIndexCellCentered()
    
    # Step 2: Copy sparse to dense
    # copyToDense fills the dense grid with values from the sparse grid
    openvdb.tools.copyToDense(sparse_grid, dense_grid, index_transform)
    
    # Extract the dense data as a numpy array (LayoutXYZ)
    dense_data = dense_grid.array()
    
    # Step 3: Convert to big-endian byte order in parallel
    # Flatten the array for byte conversion
    flat_data = dense_data.flatten()
    
    # Use TBB for parallel byte order conversion
    def convert_chunk(start, end):
        for i in range(start, end):
            # Convert float to big-endian bytes
            val = struct.unpack('>f', struct.pack('<f', flat_data[i]))[0]
            flat_data[i] = val
            
    # Divide work among TBB threads
    num_threads = tbb.num_threads()
    chunk_size = len(flat_data) // num_threads
    tasks = []
    for i in range(num_threads):
        start = i * chunk_size
        end = start + chunk_size if i < num_threads - 1 else len(flat_data)
        tasks.append(tbb.task.spawn(lambda s=start, e=end: convert_chunk(s, e)))
    
    # Wait for all tasks to complete
    for task in tasks:
        task.wait()
        
    # Step 4: Export to VTK STRUCTURED_POINTS binary file
    # Write VTK header
    with open("output.vtk", "wb") as f:
        f.write(b"# vtk DataFile Version 2.0\n")
        f.write(b"Sparse to Dense Export\n")
        f.write(b"BINARY\n")
        f.write(b"DATASET STRUCTURED_POINTS\n")
        f.write(f"DIMENSIONS {volume_size} {volume_size} {volume_size}\n".encode())
        f.write(f"ORIGIN {vmin[0]} {vmin[1]} {vmin[2]}\n".encode())
        f.write(f"SPACING {1.0} {1.0} {1.0}\n".encode()) # Assuming unit spacing
        f.write(b"POINT_DATA " + str(len(flat_data)).encode() + b"\n")
        f.write(b"SCALARS density float 1\n")
        f.write(b"LOOKUP_TABLE default\n")
        
        # Write big-endian float data
        for val in flat_data:
            f.write(struct.pack('>f', val))

```

**Parameters:**
- `volume_size`: Dimensions of the output dense grid. Typical values depend on the desired resolution and scene scale.
- `center`: World-space center of the volume.
- `transform`: OpenVDB transformation matrix defining the grid's world-space position and scale.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision may be lost during the conversion to big-endian byte order if not handled carefully. Use standard library functions for byte order conversion to ensure correctness [S1, unofficial].
- **Boundaries:** Ensure that the `volume_size` and `center` are chosen such that the sparse grid's active voxels fall within the dense grid's bounds. Voxels outside the bounds will be clipped [S1, unofficial].
- **Degenerate Input:** If the sparse grid is empty, the output dense grid will be filled with the background value. If the transformation matrix is invalid, the index mapping may fail [S1, unofficial].

## Subtopics

### [OpenVDB to Yocto Volume Conversion](14106-openvdb-to-yocto-volume-conversion.md)
Converts sparse OpenVDB grids to dense Yocto `.vol` files. Choose this when targeting the Yocto/Quasar renderer ecosystem. It handles coordinate translation to `(0,0,0)` and uses the `yovdbload` toolchain, unlike the VTK export.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of active voxels in the sparse grid. The copy operation and byte order conversion are linear in the number of active voxels.
- **Memory Complexity:** $O(V)$, where $V$ is the total number of voxels in the dense grid ($volume\_size^3$). The dense grid requires contiguous memory for all voxels, regardless of sparsity.
- **Measured Performance:** Performance depends on the sparsity of the input grid and the size of the output dense grid. For highly sparse grids, the conversion may be faster than expected due to the efficient `copyToDense` implementation, but memory usage remains high due to the dense output [S1, unofficial].
- **Strengths:** Simple, efficient conversion from sparse to dense; produces a widely supported VTK format; parallel byte order conversion improves performance.
- **Limitations:** High memory usage for large dense grids; not suitable for highly sparse data where memory efficiency is critical; VTK `STRUCTURED_POINTS` is a legacy format.
- **When to Use:** When a dense, contiguous memory layout is required for rendering or further processing, and the VTK format is acceptable.
- **When Not to Use:** When memory efficiency is paramount, or when the target renderer requires a different format (e.g., Yocto `.vol`).

## Background
This method is part of the Mesh2Volume pipeline, which converts 3D meshes into volume representations suitable for rendering [S1, unofficial]. The sparse-to-dense conversion step is crucial for exporting volumes to formats that do not support sparse data structures natively. The use of OpenVDB tools ensures efficient handling of sparse data, while the VTK export provides compatibility with a wide range of visualization and rendering tools.

## Sources
[S1] Mesh2Volume, https://github.com/zzilch/Mesh2Volume, fetched 2026-10-26 (unofficial)
