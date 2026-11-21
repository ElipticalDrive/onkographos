# SDF Ray Marching with Grid Lookup

Also called: Camera ray intersection via ray marching, Intersect kernel, camera ray marching

## Overview
A GPU-based ray marching algorithm that samples a 3D Signed Distance Function (SDF) grid to find the first intersection point of a ray with the zero-level set (surface) [S2, unofficial]. It sets itself apart from [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by using linear interpolation to evaluate the SDF at arbitrary points within the grid bounds, allowing for precise intersection detection without fixed step constraints [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   `grid`: 3D tensor of float SDF values, shape `(grid_res_x, grid_res_y, grid_res_z)` [S2, unofficial].
*   `origins`: 2D tensor of float ray origins, shape `(N, 3)` [S2, unofficial].
*   `directions`: 2D tensor of float ray directions, shape `(N, 3)` [S2, unofficial].
*   `origin_image_distances`: 1D tensor of float distances from camera origin to image plane, shape `(N,)` [S2, unofficial].
*   `pixel_distances`: 1D tensor of float distances from camera origin to pixel locations, shape `(N,)` [S2, unofficial].
*   `bounding_box_min`: 3D vector defining the minimum corner of the SDF grid's axis-aligned bounding box [S2, unofficial].
*   `bounding_box_max`: 3D vector defining the maximum corner of the SDF grid's axis-aligned bounding box [S2, unofficial].
*   `grid_res_x`, `grid_res_y`, `grid_res_z`: Integer resolutions of the SDF grid along each axis [S2, unofficial].
*   `max_steps`: Integer maximum number of marching steps per ray (default: 1000) [S2, unofficial].
*   `voxel_position`: 2D tensor of float, shape `(N, 3)`, storing the voxel coordinates of the intersection [S2, unofficial].
*   `intersection_pos`: 2D tensor of float, shape `(N, 3)`, storing the 3D world coordinates of the intersection [S2, unofficial].
*   `gotten_result`: 1D tensor of int, shape `(N,)`, flag indicating if an intersection was found (1) or not (0) [S2, unofficial].

**Data Structures:**
The SDF grid is stored as a contiguous 3D array [S2, unofficial]. Ray parameters are stored in parallel arrays for batch processing [S2, unofficial].

**Steps:**
1.  Initialize `depth = 0`, `first_time = true`, `gotten_result = 0` for each ray [S2, unofficial].
2.  Loop for `max_steps` [S2, unofficial]:
    a.  Calculate current point: `reached_point_x = origins[i] + depth * directions[i]` (and y, z) [S2, unofficial].
    b.  Call `ValueAt` to get SDF distance at `reached_point` [S2, unofficial].
    c.  Check for exit: If `distance == -1`, set `voxel_position` and `intersection_pos` to -1, set `gotten_result = 1`, and break [S2, unofficial].
    d.  Check for intersection: If `distance < pixel_distances[pixel_index] / origin_image_distances[pixel_index] * depth` AND `distance` is truthy (non-zero), record intersection [S2, unofficial]. Set `voxel_position` to floor of current point (clamped to `grid_res - 1`) [S2, unofficial]. Set `intersection_pos` to current point [S2, unofficial]. Set `gotten_result = 1` and break [S2, unofficial].
    e.  Increment `depth` [S2, unofficial].

**ValueAt (Trilinear Interpolation):**
1.  Compute fractional coordinates `tx`, `ty`, `tz` relative to the grid cell containing the point [S2, unofficial].
2.  Interpolate along X to get `c01, c23, c45, c67` [S2, unofficial].
3.  Interpolate along Y to get `c0, c1` [S2, unofficial].
4.  Interpolate along Z for final result [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def value_at(grid, point, grid_res, bounding_box_min, bounding_box_max):
    # Map world point to grid coordinates
    grid_x = (point[0] - bounding_box_min[0]) / (bounding_box_max[0] - bounding_box_min[0]) * (grid_res[0] - 1)
    grid_y = (point[1] - bounding_box_min[1]) / (bounding_box_max[1] - bounding_box_min[1]) * (grid_res[1] - 1)
    grid_z = (point[2] - bounding_box_min[2]) / (bounding_box_max[2] - bounding_box_min[2]) * (grid_res[2] - 1)

    # Check bounds
    if not (0 <= grid_x < grid_res[0] and 0 <= grid_y < grid_res[1] and 0 <= grid_z < grid_res[2]):
        return -1.0

    # Get integer and fractional parts
    ix = int(grid_x)
    iy = int(grid_y)
    iz = int(grid_z)
    fx = grid_x - ix
    fy = grid_y - iy
    fz = grid_z - iz

    # Clamp indices
    ix = max(0, min(ix, grid_res[0] - 1))
    iy = max(0, min(iy, grid_res[1] - 1))
    iz = max(0, min(iz, grid_res[2] - 1))

    # Trilinear interpolation
    c000 = grid[ix][iy][iz]
    c100 = grid[min(ix + 1, grid_res[0] - 1)][iy][iz]
    c010 = grid[ix][min(iy + 1, grid_res[1] - 1)][iz]
    c110 = grid[min(ix + 1, grid_res[0] - 1)][min(iy + 1, grid_res[1] - 1)][iz]
    c001 = grid[ix][iy][min(iz + 1, grid_res[2] - 1)]
    c101 = grid[min(ix + 1, grid_res[0] - 1)][iy][min(iz + 1, grid_res[2] - 1)]
    c011 = grid[ix][min(iy + 1, grid_res[1] - 1)][min(iz + 1, grid_res[2] - 1)]
    c111 = grid[min(ix + 1, grid_res[0] - 1)][min(iy + 1, grid_res[1] - 1)][min(iz + 1, grid_res[2] - 1)]

    # Interpolate along X
    c01 = c000 * (1 - fx) + c100 * fx
    c23 = c010 * (1 - fx) + c110 * fx
    c45 = c001 * (1 - fx) + c101 * fx
    c67 = c011 * (1 - fx) + c111 * fx

    # Interpolate along Y
    c0 = c01 * (1 - fy) + c23 * fy
    c1 = c45 * (1 - fy) + c67 * fy

    # Interpolate along Z
    return c0 * (1 - fz) + c1 * fz

def ray_marching_with_grid_lookup(grid, origins, directions, origin_image_distances, pixel_distances, grid_res, bounding_box_min, bounding_box_max, max_steps=1000):
    N = len(origins)
    voxel_position = [[-1.0, -1.0, -1.0] for _ in range(N)]
    intersection_pos = [[-1.0, -1.0, -1.0] for _ in range(N)]
    gotten_result = [0] * N

    for i in range(N):
        depth = 0.0
        gotten_result[i] = 0
        for step in range(max_steps):
            # Calculate current point
            reached_point = [
                origins[i][0] + depth * directions[i][0],
                origins[i][1] + depth * directions[i][1],
                origins[i][2] + depth * directions[i][2]
            ]

            # Get SDF distance
            distance = value_at(grid, reached_point, grid_res, bounding_box_min, bounding_box_max)

            # Check for exit
            if distance == -1.0:
                voxel_position[i] = [-1.0, -1.0, -1.0]
                intersection_pos[i] = [-1.0, -1.0, -1.0]
                gotten_result[i] = 1
                break

            # Check for intersection
            threshold = (pixel_distances[i] / origin_image_distances[i]) * depth
            if distance < threshold and distance != 0.0:
                # Record intersection
                floor_point = [float(int(p)) for p in reached_point]
                clamped_point = [
                    max(0.0, min(floor_point[0], grid_res[0] - 1)),
                    max(0.0, min(floor_point[1], grid_res[1] - 1)),
                    max(0.0, min(floor_point[2], grid_res[2] - 1))
                ]
                voxel_position[i] = clamped_point
                intersection_pos[i] = reached_point
                gotten_result[i] = 1
                break

            depth += 1.0

    return voxel_position, intersection_pos, gotten_result
```

## Subtopics

### [Raycasting with Irregular Grid Interpolation](12445-raycasting-with-irregular-grid-interpolation.md)
This method extends SDF ray marching to handle hierarchical voxel grids with varying resolutions. It solves the problem of evaluating the Signed Distance Function (SDF) at arbitrary points near boundaries between different resolution levels, where standard tri-linear interpolation is invalid. It sets itself apart from its parent by replacing the uniform interpolation kernel with a local polynomial fit derived from a linear system, ensuring continuity across resolution boundaries.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] ShadowNeuS: Neural SDF Reconstruction by Shadow Ray Supervision, https://doi.org/10.1109/cvpr52729.2023.00025, fetched 2026-10-26
[S2] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
[S3] PDF Generating Signed Distance Fields from Triangle Meshes using Vulkan ..., https://williamwhitehouse.dev/assets/projects/2023/sdf-gen/gen-sdf-from-meshes-using-vulkan-compute-report.pdf, fetched 2026-10-30 (unofficial)
