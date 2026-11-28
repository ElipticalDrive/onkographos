# Dense Copy Extraction

Also called: OpenVDB to Dense Array Conversion, SDF to Dense Volume Conversion

## Overview
Dense Copy Extraction extracts a spatial region defined by a bounding box from a sparse grid into a contiguous dense 3D array [S1, unofficial][S2, unofficial]. It converts hierarchical sparse data structures into uniform grids suitable for dense rendering or processing [S2, unofficial]. Unlike sibling methods that query individual points or gradients, this method performs a bulk conversion by initializing the output with a background value and overwriting it with interpolated values for all coordinates within the bounds [S1, unofficial].

## How it works
**Inputs:**
- A sparse grid with an associated transform [S2, unofficial].
- A bounding box (`bbox`) defining the region of interest [S1, unofficial][S2, unofficial].
- A background value (default from the tree) [S1, unofficial].
- Optional flag `flag_cube` to force a cubic bounding box (default: `true`) [S2, unofficial].

**Outputs:**
- A dense 3D array containing the sampled values [S1, unofficial][S2, unofficial].

**Data Structures:**
- **Bounding Box:** Defined by minimum and maximum coordinates in world space [S1, unofficial][S2, unofficial].
- **Dense Array:** A contiguous 3D buffer with dimensions derived from the bounding box and grid spacing [S1, unofficial][S2, unofficial].
- **Transform:** Maps world coordinates to voxel indices [S2, unofficial].

**Numbered Steps:**
1. Calculate dimensions `dx, dy, dz` from `bbox` and grid spacing [S1, unofficial].
2. Initialize `result` array filled with `tree.background` [S1, unofficial].
3. Compute center and dense bounding box: `center = bbox.getCenter()`, `bbox_dense = BBoxd(center - length/2, center + length/2)` [S2, unofficial].
4. Create dense container with `transform->worldToIndexNodeCentered(bbox_dense)` and call `copyToDense(*grid, dense)` [S2, unofficial].
5. Triple nested loop over `ix, iy, iz` [S1, unofficial].
6. Compute `c = Coord(bbox.min.x + ix, bbox.min.y + iy, bbox.min.z + iz)` [S1, unofficial].
7. Set `result[ix + 1, iy + 1, iz + 1] = get_value(tree, c)` [S1, unofficial].

**Formulas:**
- `center = (bbox.min + bbox.max) / 2` [S2, unofficial].
- `bbox_dense = [center - length/2, center + length/2]` where `length = max(bbox.max - bbox.min)` [S2, unofficial].
- `dx = round((bbox.max.x - bbox.min.x) / spacing.x)`, `dy = round((bbox.max.y - bbox.min.y) / spacing.y)`, `dz = round((bbox.max.z - bbox.min.z) / spacing.z)` [S1, unofficial].

**Parameters:**
- `flag_cube`: Forces bounding box to be cubic. Default: `true` [S2, unofficial].

**Edge Cases and Pitfalls:**
- Floating-point errors in coordinate calculations may lead to off-by-one errors in dimension calculation [S1, unofficial][S2, unofficial].
- Bounding box alignment with grid voxel boundaries affects sampling accuracy [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Coord:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

def dense_copy_extraction(grid, bbox, background_value, flag_cube=True):
    # Step 1: Calculate dimensions dx, dy, dz from bbox
    spacing = grid.get_spacing()
    dx = int(round((bbox.max.x - bbox.min.x) / spacing.x))
    dy = int(round((bbox.max.y - bbox.min.y) / spacing.y))
    dz = int(round((bbox.max.z - bbox.min.z) / spacing.z))

    # Step 2: Initialize result array filled with tree.background
    result = [[[background_value for _ in range(dz)] for _ in range(dy)] for _ in range(dx)]

    # Step 3: Compute center and dense bounding box
    center_x = (bbox.min.x + bbox.max.x) / 2
    center_y = (bbox.min.y + bbox.max.y) / 2
    center_z = (bbox.min.z + bbox.max.z) / 2
    length = max(bbox.max.x - bbox.min.x, bbox.max.y - bbox.min.y, bbox.max.z - bbox.min.z)
    if flag_cube:
        bbox.min = Coord(center_x - length / 2, center_y - length / 2, center_z - length / 2)
        bbox.max = Coord(center_x + length / 2, center_y + length / 2, center_z + length / 2)

    # Step 4: Create dense container with transform->worldToIndexNodeCentered(bbox_dense)
    # (Library-specific initialization; iteration proceeds below)

    # Step 5, 6, 7: Triple nested loop over ix, iy, iz
    for ix in range(dx):
        for iy in range(dy):
            for iz in range(dz):
                c = Coord(bbox.min.x + ix, bbox.min.y + iy, bbox.min.z + iz)
                value = grid.get_value(c.x, c.y, c.z)
                result[ix + 1][iy + 1][iz + 1] = value

    return result
```

## Operating characteristics
- Time Complexity: Proportional to the number of voxels in the bounding box, as every coordinate is visited in a triple nested loop [S1, unofficial].
- Memory Complexity: Proportional to the volume of the bounding box for the output dense array [S1, unofficial][S2, unofficial].
- Performance: Depends on the sparse grid lookup and interpolation method [S2, unofficial].
- Strengths: Produces a standard dense format suitable for downstream processing [S1, unofficial][S2, unofficial].
- Limitations: Memory usage scales with the bounding box volume regardless of sparse occupancy [S1, unofficial].
- When to Use: When a dense representation is required for rendering or machine learning inputs [S2, unofficial].
- When Not to Use: When memory is constrained or the region is extremely sparse [S1, unofficial].

## Background
Dense Copy Extraction converts sparse hierarchical grids into contiguous dense arrays with known origin and spacing [S2, unofficial]. It is used in mesh-to-volume pipelines and SDF rendering workflows to bridge sparse representations with dense processing requirements [S2, unofficial].

## Sources
[S1] Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] Mesh2Volume, https://github.com/zzilch/Mesh2Volume, fetched 2026-10-26 (unofficial)
