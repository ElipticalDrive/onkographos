# Ray-marching reference (min-max octree accelerated)

Also called: Empty Space Skipping with Min/Max Cells, Interactive Isosurface Ray Tracing of Large Octree Volumes, Min-Max Octree Ray Casting, Min-Max Octree Ray-Marching, Min-Max Octree Ray-Marching Reference, Octree-accelerated ray-marching

## Overview
This method is an accelerated direct isosurface visualization technique that uses a min-max octree to facilitate empty space skipping during ray marching [S1, S2][S3, unofficial]. It serves as a performance baseline for general volumes, offering negligible memory overhead for the input volume and low auxiliary data requirements [S3, unofficial]. Unlike Sphere Tracing, which requires precomputed signed distance functions, or Segment Tracing, which is limited to hierarchical skeletal implicit surfaces, this approach is applicable to a wide range of scalar volumes [S2][S3, unofficial]. It guarantees continuous isosurfaces that locally interpolate the input data, unlike mesh extraction techniques (e.g., marching cubes) which may produce topologically different piecewise linear meshes [S4, unofficial].

## How it works
**Inputs:**
- A 3D scalar volume dataset [G].
- An isovalue threshold for isosurface extraction [G].
- Camera parameters (origin and direction for each pixel) [G].

**Outputs:**
- A rendered image of the isosurface with shading [G].

**Data Structures:**
1. **Min-Max Texture:** A texture where each texel stores the minimum and maximum scalar values of a corresponding block or cell in the volume, used to determine block occupancy [S1][S3, unofficial][S5].
2. **Min-Max Octree:** Constructed from the min-max texture via a MIP texture hierarchy [S1][S3, unofficial]. The octree is built on a base level with equal size along all three cardinal axes, and volume dimensions are padded to the nearest power of two [S1][S3, unofficial]. Internal nodes store min/max values to enable empty space skipping [S4, unofficial][S5].
3. **Scalar Data:** The original volume data, which can be losslessly compressed into the octree representation to occupy less than one quarter of the original 3D array size [S4, unofficial].

**Steps:**
1. **Construct the Min-Max Octree:** Create a MIP texture from the min-max texture used for block occupancy determination [S1][S3, unofficial]. Build the octree on a base level with equal size along all three cardinal axes [S1][S3, unofficial]. Pad volume dimensions to the nearest power of two for straightforward implementation [S1].
2. **Compute Entry and Exit Points:** Instead of rendering one cube enclosing the whole volume, render cubes corresponding to one level of the min/max octree [S5]. All vertices of a cube share the same 2D texture coordinates containing the min/max values [S5]. To compute exit positions, render the cubes with `glCullFace(GL FRONT)` and `glDepthFunc(GL LESS)` [S5]. To compute entry positions, render the cubes again with `glCullFace(GL BACK)` and `glDepthFunc(GL GREATER)` [S5]. A fragment shader discards fragments if the cube is empty or does not contain the isosurface level [S5].
3. **Ray Marching:** March rays through the volume using a step size of $0.5 \times \text{cell edge length}$ [S1][S3, unofficial]. Use the min-max octree to skip empty space during traversal [S3, S4, unofficial].
4. **Isosurface Refinement:** Refine the isosurface intersection position using exactly three iterations of a binary search algorithm [S1][S3, unofficial].
5. **Shading:** Compute surface normals on the fly using the central difference method upon finding an isosurface intersection [S1][S3, unofficial]. Apply Gooch shading using the computed normals [S1][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_min_max_octree(volume, isovalue, camera, octree, cell_edge_length):
    height, width = camera.height, camera.width
    image = [[None] * width for _ in range(height)]
    step_size = 0.5 * cell_edge_length

    for y in range(height):
        for x in range(width):
            ray_origin = camera.origin(x, y)
            ray_direction = camera.direction(x, y)

            entry_point, exit_point = compute_entry_exit_points(ray_origin, ray_direction, octree, isovalue)
            if entry_point is None or exit_point is None:
                continue

            current_point = entry_point
            while distance(current_point, exit_point) > 1e-6:
                if is_empty(current_point, octree, isovalue):
                    current_point += ray_direction * step_size
                    continue

                value = sample_volume(volume, current_point)
                if value >= isovalue:
                    intersection = binary_search_refinement(current_point, ray_direction, volume, isovalue, cell_edge_length)
                    normal = compute_normal(volume, intersection, cell_edge_length)
                    image[y][x] = apply_gooch_shading(normal)
                    break

                current_point += ray_direction * step_size
    return image

def compute_entry_exit_points(ray_origin, ray_direction, octree, isovalue):
    # Conceptual CPU traversal; sources describe this as a GPU rendering pass [S5]
    entry_point = None
    exit_point = None
    t = 0.0
    for _ in range(10000):
        point = ray_origin + ray_direction * t
        if not is_empty(point, octree, isovalue):
            if entry_point is None:
                entry_point = point
            exit_point = point
        t += 0.5
    return entry_point, exit_point

def is_empty(point, octree, isovalue):
    node = octree.find_node_containing(point)
    return node.min_value > isovalue or node.max_value < isovalue

def sample_volume(volume, point):
    ix, iy, iz = int(point.x), int(point.y), int(point.z)
    ix = max(0, min(ix, volume.width - 1))
    iy = max(0, min(iy, volume.height - 1))
    iz = max(0, min(iz, volume.depth - 1))
    return volume[ix, iy, iz]

def binary_search_refinement(point, direction, volume, isovalue, cell_edge_length):
    low = point
    high = point + direction * cell_edge_length
    for _ in range(3):
        mid = low + (high - low) * 0.5
        value = sample_volume(volume, mid)
        if value < isovalue:
            low = mid
        else:
            high = mid
    return low + (high - low) * 0.5

def compute_normal(volume, point, cell_edge_length):
    eps = cell_edge_length / 2.0
    dx = sample_volume(volume, point + vec3(eps, 0, 0)) - sample_volume(volume, point - vec3(eps, 0, 0))
    dy = sample_volume(volume, point + vec3(0, eps, 0)) - sample_volume(volume, point - vec3(0, eps, 0))
    dz = sample_volume(volume, point + vec3(0, 0, eps)) - sample_volume(volume, point - vec3(0, 0, eps))
    return normalize(vec3(dx, dy, dz))

def apply_gooch_shading(normal):
    light_dir = normalize(vec3(1, 1, 1))
    diff = max(dot(normal, light_dir), 0.0)
    return vec3(0.5 + 0.5 * diff)
```

**Parameters:**
- Binary search iterations for isosurface refinement: 3 [S1][S3, unofficial].
- Ray step size: $0.5 \times \text{cell edge length}$ [S1][S3, unofficial].
- Number of min/max cells: $32^3$ resulted in the best performance in experiments [S5].

**Edge Cases and Pitfalls:**
- A large number of small cells induces a tight bound of the isosurface but comes with a large overhead due to the primitives [S5].
- A small number of large cells reduces the overhead but bounds the isosurface loosely [S5].
- Precision issues may arise during binary search if the step size is too large relative to the volume gradient [G].
- Boundary conditions must be handled when sampling near the volume edges [G].

## Subtopics

### [Object-Order Iso-Surface Rendering with Macro-Cells](1757-object-order-iso-surface-rendering-with-macro-cells.md)
An iso-surface rendering algorithm based on object-order ray casting that subdivides data into macro-cells to build a min/max octree, reducing traversal steps by rasterizing hexagonal footprints. It differs from its parent method [Ray-marching reference (min-max octree accelerated)](4392-hierarchical-and-block-based-data-structures.md) by processing visible volume cells directly rather than pixels, and from its sibling [Object-Order Ray Casting with Min/Max Octrees](1814-object-order-ray-casting-with-min-max-octrees.md) by relying on pre-computed hierarchical data structures to skip empty regions.

## Operating characteristics
- **Time Complexity:** Depends on the number of rays and the sparsity of the volume [G]. Empty space skipping reduces the number of samples per ray [S1, S2][S3, unofficial][S4, unofficial][S5].
- **Memory Complexity:** Auxiliary octree data memory requirement is 1 to 4 megabytes for octrees with a depth of 7 or more [S3, unofficial]. The lossless compression into an octree volume occupies less than one quarter the size of the original 3D array [S4, unofficial].
- **Measured Performance:** On an Intel Core Duo 2.16 GHz laptop with 2 GB RAM, frame rates were 2.4, 1.3, and 3.3 fps for three test scenes (LLNL Richtmyer-Meshkov, Utah CSAFE heptane, and a closer view) [S4, unofficial]. On a 16-node NUMA 2.4 GHz Opteron workstation, frame rates were 17.9, 9.8, and 22.0 fps for the same three test scenes [S4, unofficial]. Performance gain up to ≈ 4.95-fold using fine-grained min/max cells for efficient empty space skipping [S5].
- **Scaling:** The method scales well to large data, particularly when scene complexity is high relative to the number of rays that must be cast to fill a frame [S4, unofficial].
- **Strengths:** Applicable to a wide range of volumes [S2][S3, unofficial]. Low auxiliary data memory overhead [S3, unofficial]. Guarantees continuous isosurfaces [S4, unofficial].
- **Limitations:** Outperformed by raster occlusion culling approaches at high resolutions (e.g., 3840×2160) [S1]. Less efficient than the proposed rasterization approach for most resolutions, except for sparse datasets like Chameleon at low-to-medium resolutions [S1][S3, unofficial]. Dependent on rendering resolution for the ray-marching part, unlike the proposed rasterization approach which works largely in volume space [S3, unofficial].
- **When to use:** Suitable for general accelerated direct isosurface rendering, especially when memory overhead must be minimized and the volume is sparse [S3, unofficial].
- **When not to use:** Not suitable for very high resolutions where raster occlusion culling is more efficient [S1]. Not suitable if signed distance functions are available and Sphere Tracing is preferred [S2].

## Background
- **Origin:** The method was introduced as a reference technique for evaluating direct isosurface rendering pipelines [S1, S2][S3, unofficial]. It is based on the concept of using min-max octrees for empty space skipping [S1, S2][S3, unofficial].
- **History:** The technique has been used as a baseline for comparing new rendering approaches [S3, unofficial]. It has been implemented in various research projects, including those involving large compressed structured volumes [S4, unofficial] and FCC datasets [S5].
- **Usage:** It is used as a baseline reference for direct isosurface visualization performance in evaluations [S3, unofficial].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://doi.org/10.1111/cgf.14670, fetched 2026-10-27
[S2] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S3] Efficient Direct Isosurface Rasterization of Scalar Volumes - EG, https://diglib.eg.org/items/8a65e0f0-c048-4fce-9d0b-5acf07452485, fetched 2026-10-30 (unofficial)
[S4] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S5] PDF GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
