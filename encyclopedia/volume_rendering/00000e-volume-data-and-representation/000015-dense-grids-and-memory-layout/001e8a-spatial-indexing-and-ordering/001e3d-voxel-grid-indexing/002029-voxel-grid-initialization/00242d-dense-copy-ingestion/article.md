# Dense Copy Ingestion
## Overview
Dense Copy Ingestion constructs a sparse VDB grid from a dense 3D array by copying only values that differ from a specified background value [S1, unofficial]. It differs from [Voxel Grid Construction](3085-voxel-grid-construction.md) by assuming the input is already a dense, contiguous array rather than generating voxels from geometric primitives or point clouds. It is the inverse of [Sparse-to-Dense Grid Conversion](#sparse-to-dense-grid-conversion).
## How it works
**Inputs:**
- `array`: A dense 3D array (e.g., `Float32` or `Float64`) of dimensions $W \times H \times D$.
- `background`: The scalar value representing empty space.
- `bbox_min`: The world-space minimum corner of the grid's bounding box.
- `voxel_size`: The physical size of a single voxel (assumed uniform in $x, y, z$).

**Outputs:**
- A VDB grid data structure containing active voxels.

**Data Structures:**
- The output grid uses a sparse tree structure (e.g., VDB's `Grid` and `Tree`).
- Active voxels are stored only where `array[i,j,k] != background`.
- Each active voxel stores its world coordinate and value.

**Steps:**
1. Initialize an empty list `data` to hold active voxel records.
2. Iterate over all indices $(i, j, k)$ in `array`:
   - Let $v = \text{array}[i, j, k]$.
   - If $v \neq \text{background}$:
     - Compute the world coordinate $c$ of the voxel center. Assuming the array index $(0,0,0)$ corresponds to `bbox_min` + half-voxel offset (cell-centered) or `bbox_min` (node-centered), the exact mapping depends on the implementation. The source implies a direct mapping using `bbox_min` [S1, unofficial].
     - Append $(c, v)$ to `data`.
3. Call `build_grid(data, background)` to instantiate the VDB grid with the collected active voxels [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dense_copy_ingestion(array, background, bbox_min, voxel_size):
    """
    array: 3D numpy-like array of shape (W, H, D)
    background: scalar float
    bbox_min: tuple (x, y, z) of world coordinates for the first voxel's corner/center
    voxel_size: float, physical size of one voxel
    """
    active_voxels = []
    W, H, D = array.shape
    
    # Determine if bbox_min is corner or center. 
    # Standard VDB often uses node centers or cell centers. 
    # We assume cell-centered for this generic implementation [G].
    # Offset by half voxel size to get center from corner.
    half_voxel = voxel_size / 2.0
    
    for i in range(W):
        for j in range(H):
            for k in range(D):
                value = array[i, j, k]
                
                # Check against background
                if value != background:
                    # Compute world coordinate of the voxel center
                    # Assuming bbox_min is the minimum corner of the bounding box
                    x = bbox_min[0] + i * voxel_size + half_voxel
                    y = bbox_min[1] + j * voxel_size + half_voxel
                    z = bbox_min[2] + k * voxel_size + half_voxel
                    
                    active_voxels.append({
                        'coord': (x, y, z),
                        'value': value
                    })
    
    # Build the grid structure
    grid = build_vdb_grid(active_voxels, background)
    return grid
```

**Edge Cases:**
- **Precision:** Floating point comparisons `value != background` may fail due to precision errors if the background value was computed rather than literal. Use a tolerance if necessary [G].
- **Empty Grid:** If all values equal the background, the resulting grid has no active voxels.

## Subtopics

### Sparse-to-Dense Grid Conversion
This variant performs the reverse operation: it renders a sparse OpenVDB grid into a dense contiguous float array with LayoutXYZ ordering, then exports it as a legacy VTK STRUCTURED_POINTS binary file [S2, unofficial]. It is chosen when a dense representation is required for legacy tools or specific rendering pipelines that do not support sparse formats [S2, unofficial].
**Steps:**
1. Compute dense grid bounds: $v_{min} = \text{center} - \text{volume\_size}/2$, $v_{max} = \text{center} + \text{volume\_size}/2$ [S2, unofficial].
2. Create a dense grid using `transform->worldToIndexCellCentered` or `worldToIndexNodeCentered` to map world coordinates to indices [S2, unofficial].
3. Copy sparse grid data to the dense grid using `openvdb::tools::copyToDense` [S2, unofficial].
4. Convert dense data to big-endian byte order in parallel using TBB `blocked_range` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(W \times H \times D)$ for the iteration over the dense array [S1, unofficial].
- **Memory Complexity:** $O(N_{active})$ for the output grid, where $N_{active}$ is the number of voxels differing from the background. The input array requires $O(W \times H \times D)$ memory [S1, unofficial].
- **Performance:** The method is I/O bound if reading from disk and memory-bound during the copy phase. It is efficient for sparse data within a dense container but wastes memory if the data is dense [S1, unofficial].
- **Limitations:** Does not handle non-uniform voxel sizes or complex bounding boxes beyond the initial `bbox_min` assumption [S1, unofficial].

## Background
This method is part of the Lyr.jl volume rendering library, which integrates VDB grids for efficient sparse volume rendering [S1, unofficial]. It is also referenced in Mesh2Volume for converting sparse OpenVDB grids to dense formats for VTK export [S2, unofficial].

## Sources
[S1] Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] Mesh2Volume, https://github.com/zzilch/Mesh2Volume, fetched 2026-10-26 (unofficial)
