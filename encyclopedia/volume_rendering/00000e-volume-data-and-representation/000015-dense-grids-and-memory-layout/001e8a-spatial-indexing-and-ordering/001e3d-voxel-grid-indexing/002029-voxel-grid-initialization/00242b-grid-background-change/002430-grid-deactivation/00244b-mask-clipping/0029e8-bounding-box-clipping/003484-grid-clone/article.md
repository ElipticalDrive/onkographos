# Grid Clone
Also called: PixelGrid2D Clone
## Overview
Grid Clone creates a new grid instance by copying the spatial metadata and all voxel values from a source grid [S1, unofficial]. It preserves the full logical structure and resolution of the original data without spatial filtering [S1, unofficial].
## How it works
**Inputs:** A source grid object containing a bounding box, pixel size, and a 1D array of voxel values [S1, unofficial].
**Outputs:** A new grid instance with identical metadata and a deep copy of the voxel data [S1, unofficial].
**Data Structures:** The grid stores a bounding box (`Box`), pixel dimensions (`Point3d`), and a 1D array for values sized by the total voxel count [S1, unofficial].
**Steps:**
1. Initialize a new grid object using the source's bounding box and pixel size [S1, unofficial].
2. Allocate a data array matching the source's total voxel count [S1, unofficial].
3. Iterate from index $0$ to $Count$ (exclusive) and assign each source value to the corresponding index in the new grid [S1, unofficial].
**Formula:** $\text{newGrid.Grid}[i] = \text{sourceGrid.Grid}[i] \quad \forall i \in [0, \text{Count})$ [S1, unofficial].
**Parameters:** Not given in the sources.
**Edge Cases:** If the source grid has zero voxels, the loop range is empty and the new grid retains valid metadata with an empty data array [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_clone(source_bbox, source_pixel_size, source_voxel_data):
    # Step 1: Initialize new grid with source metadata
    new_bbox = source_bbox
    new_pixel_size = source_pixel_size
    
    # Step 2: Allocate data array
    count = len(source_voxel_data)
    new_voxel_data = [0.0] * count
    
    # Step 3: Copy values
    for i in range(count):
        new_voxel_data[i] = source_voxel_data[i]
        
    return {'bbox': new_bbox, 'pixel_size': new_pixel_size, 'voxel_data': new_voxel_data}
```
## Subtopics

### PixelGrid2D CloneEmpty
Returns a new `PixelGrid2D` object initialized with the current bounding box and pixel size, without copying any voxel data [S1, unofficial]. Not given in the sources whether the data array is allocated or left null.

## Operating characteristics
**Time Complexity:** $O(N)$, where $N$ is the total number of voxels, due to a single sequential pass over the data array [G].
**Memory Complexity:** $O(N)$, as a new array of identical size is allocated alongside the source [G].
**Strengths:** Preserves exact spatial metadata and resolution; simple sequential memory access pattern [G].
**Limitations:** Requires contiguous memory allocation for the full volume; does not skip inactive or empty regions [G].
## Background
This method originates from the `voxeltool` repository, an unofficial C# implementation for voxel manipulation [S1, unofficial]. It provides basic grid initialization and duplication operations for 3D scalar and pixel grids.
## Sources
[S1] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
