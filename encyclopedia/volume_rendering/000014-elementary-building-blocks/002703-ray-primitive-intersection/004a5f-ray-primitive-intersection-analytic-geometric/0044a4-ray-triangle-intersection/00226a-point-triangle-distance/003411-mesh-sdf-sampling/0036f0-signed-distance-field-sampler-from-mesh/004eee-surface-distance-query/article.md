# Surface Distance Query

Also called: Minimum Surface Distance Query, Closest Surface Search

## Overview
Surface Distance Query is a method for determining the minimum Euclidean distance from a query point to a collection of geometric surfaces, while optionally identifying the specific surface responsible for that minimum distance [S1, unofficial]. It solves the problem of proximity detection in scenes containing multiple distinct objects or mesh parts, serving as the foundational lookup mechanism for Signed Distance Field (SDF) Samplers built from meshes [Parent]. Unlike single-mesh distance queries, this method aggregates results across a list of surfaces, enabling complex scene representations where the global distance is the minimum of local surface distances [S1, unofficial].

## How it works
**Inputs:**
- `point`: A 3D coordinate $(x, y, z)$ in world space.
- `surfaces`: A list or array of surface objects. Each surface object must provide a method to compute its distance at a given point.

**Outputs:**
- `min_dist`: The scalar minimum distance found.
- `closest_surface`: The reference to the surface object that yielded `min_dist`.

**Data Structures:**
- `surfaces`: An iterable collection of surface handles.
- `min_dist`: Initialized to a large constant representing the maximum search radius.
- `closest_surface`: Initialized to `null` or `None`.

**Algorithm Steps:**
1. Initialize `min_dist` to `MAX_DISTANCE` (a value larger than any expected scene dimension, e.g., 2000.0) [S1, unofficial].
2. Initialize `closest_surface` to `null` [S1, unofficial].
3. Iterate through each `surface` in the `surfaces` list:
    a. Query the surface for its distance at the point [S1, unofficial].
    b. If the returned distance is less than `min_dist`:
        i. Set `min_dist` to the returned distance [S1, unofficial].
        ii. Set `closest_surface` to the current surface [S1, unofficial].
4. Return `min_dist` and `closest_surface` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_distance_query(point, surfaces, max_distance=2000.0):
    """
    Finds the minimum distance from point to any surface in the list.
    
    Args:
        point: Tuple or array of (x, y, z).
        surfaces: List of objects with a distance computation method.
        max_distance: Upper bound for initialization.
        
    Returns:
        min_dist: Float, the minimum distance.
        closest_surface: Object, the surface yielding the minimum distance.
    """
    min_dist = max_distance
    closest_surface = None
    
    for surface in surfaces:
        # Query the surface for its distance at the point
        d = surface.distance(point)
        
        if d < min_dist:
            min_dist = d
            closest_surface = surface
            
    return min_dist, closest_surface
```

**Parameters:**
- `max_distance`: Typical value depends on scene scale. In the reference implementation, `MAX_DISTANCE` is set to `2000f` [S1, unofficial].
- `MIN_DISTANCE`: A lower bound threshold used in ray marching contexts to terminate early if the distance is sufficiently small (e.g., `0.2f`) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Empty List:** If `surfaces` is empty, the function returns `max_distance` and `null` [G]. The caller must handle this case to avoid dereferencing a null surface [G].
- **Precision:** Floating-point comparison `d < min_dist` is standard. For signed distances, negative values (inside geometry) will correctly update the minimum if they are smaller than the current positive minimum [G].
- **Performance:** This is a linear scan $O(N)$ where $N$ is the number of surfaces. For large $N$, spatial partitioning (BVH, Grid) is required before querying individual surfaces, but the query logic itself remains a reduction [G].

## Subtopics

### [3D Empty Space Supervision](14034-3d-empty-space-supervision.md)
**Definition:** A method for regularizing the non-occupied regions of a learned radiance field by supervising the "object code" using estimated surface distances. It solves the problem of ambiguous empty space in neural rendering, where the model may incorrectly assign object semantics to background voxels. It sets itself apart from standard volume rendering losses by explicitly computing a "surfaceness" and "emptiness" score for every sample point along a ray, allowing for targeted supervision of the object field's background channel.

### Two-Sided Distance Field Generation
**Definition:** A configuration for high-density meshes (e.g., vegetation with masked materials like leaves/branches) where surfaces contain many holes and cannot be treated as solid opaque boundaries [S2]. It generates distance fields for both sides of the geometry [S2].
**When to choose:** Use for semi-transparent or highly perforated geometry like trees and grass where standard one-sided distance fields fail to capture the interior/exterior relationship correctly [S2].
**Pitfall:** Enabling this increases ray marching cost because the surface becomes semi-transparent in the distance field representation, requiring more steps to find the mesh intersection compared to opaque surfaces [S2].

### Geodesic Distance Rendering
**Definition:** A technique that renders a surface colored by the geodesic (mesh-path) distance from a query point, computed via Dijkstra's algorithm on the surface mesh [S3, unofficial].
**Steps:** Find the closest mesh vertex to the query point using a spatial locator [S3, unofficial]. Run Dijkstra's algorithm on the surface mesh from the closest vertex to a dummy end vertex (0) to compute cumulative edge weights [S3, unofficial]. Extract cumulative weights as a numpy array and replace uncomputed/infinite distances with -1.0 [S3, unofficial]. Normalize distances based on `clim_max` (default 'ptile95'): supports 'max', 'mean', 'median', 'ptileN' (percentile), or a float; divide by `clim_max`, scale to 0-255, clip, and mark negatives as -1 [S3, unofficial]. Construct a 257-entry LUT: index 0 is white with specified alpha; indices 1-256 map to the colormap [S3, unofficial]. Create a polydata mapper, set input, override scalar range to (-1.0, max_data_range), and bind the LUT [S3, unofficial].
**Parameters:** `clim_max`: Normalization target, default 'ptile95' [S3, unofficial].
**Layout:** `vtk.vtkLookupTable`: 257 entries [S3, unofficial].
**Pitfall:** Unreachable mesh regions or uncomputed Dijkstra weights exceed `VTK_DOUBLE_MAX/10.0` and must be explicitly replaced with -1.0 to avoid rendering artifacts [S3, unofficial].

### Closest Point Query with FCPW Comparison
**Definition:** The closest point query (CPQ) finds the nearest surface point to a given query position [S4].
**Steps:** Tree construction uses top-down recursive partitioning with binned surface area heuristic (SAH) splits: primitives are distributed into spatial buckets along each axis, and sweep operations compute partition costs to select optimal splits [S4]. Distance-based node sorting is implemented, which visits children in an order determined by proximity to the query point [S4]. Queries are issued as 100,000 randomly sampled queries within each scene’s bounding volume, executed single-threaded [S4].
**Parameters:** Query count: 100,000 randomly sampled queries per scene [S4].
**Pitfall:** Scion checks for SENTINEL nodes because some layouts may have invalid children, whereas FCPW assumes sentinels do not exist and elides this check, causing a performance difference [S4].
**Numbers:** On ARM, Scion is 4% slower than FCPW (geomean) [S4]. On x86, Scion is 5% faster than FCPW (geomean) [S4]. Worst case performance is 0.89x on the white-oak scene [S4].

### Point-to-Mesh Distance
**Definition:** A metric that computes the displacements between each point in a compared point cloud and the nearest facet of a reference mesh. It provides signed distances if mesh normals are considered [S5].
**Steps:** For each point in the compared point cloud, search for the nearest triangle in the reference mesh [S5]. Calculate the nearest Euclidean distance between the point and the triangle facet [S5]. Optionally assign a sign to the distance based on the triangle normal [S5].
**Parameters:** Reference mesh: A high-accuracy ground truth mesh (e.g., from Structured Light Imaging) used as the baseline for distance calculations [S5].

## Operating characteristics
- **Time Complexity:** $O(N)$ per query, where $N$ is the number of surfaces in the list [S1, unofficial]. Each surface distance calculation is $O(1)$ for analytic primitives or $O(M)$ for mesh queries depending on the surface implementation [G].
- **Memory Complexity:** $O(N)$ to store the list of surface references [S1, unofficial]. No additional temporary memory is required beyond scalar variables [G].
- **Strengths:** Simple to implement and agnostic to the underlying surface representation as long as a distance method exists [G].
- **Limitations:** Linear scan does not scale to scenes with thousands of surfaces without spatial acceleration structures [G]. Does not handle self-occlusion or visibility; it returns the geometric minimum, not the visible minimum [G].
- **When to use:** Small to medium-sized scenes, or as a backend for SDF samplers where the number of constituent meshes is manageable [Parent, S1].
- **When not to use:** Large-scale open worlds with millions of objects; use spatial hashing or BVH-based distance queries instead [G].

## Background
The method originates from basic ray-marching shader implementations where multiple surface types are queried per pixel to determine the closest hit [S1, unofficial]. It is a core component of the "Signed Distance Field Sampler from Mesh" pipeline, allowing a single sampler function to abstract over a `THREE.Mesh` instance composed of multiple geometries [Parent]. The specific implementation details (MAX_DISTANCE=2000f, MIN_DISTANCE=0.2f) come from a Java-based ray marching engine [S1, unofficial]. The variant "Geodesic Distance Rendering" utilizes VTK's Dijkstra implementation for path-based distance visualization [S3, unofficial].

## Sources
[S1] conradplake/ray-marching-shader, https://github.com/conradplake/ray-marching-shader, fetched 2026-10-27 (unofficial)
[S2] 虚幻引擎网格体距离场, https://dev.epicgames.com/documentation/unreal-engine/mesh-distance-fields-in-unreal-engine?lang=zh-CN, fetched 2026-10-30
[S3] Volume Renderer with VTK - yuta-hi, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
[S4] Decoupling Data Layouts from Bounding Volume Hierarchies, https://arxiv.org/pdf/2511.15028, fetched 2026-10-29
[S5] GEOMETRIC ACCURACY ANALYSIS BETWEEN NEURAL RADIANCE FIELDS (NERFS) AND TERRESTRIAL LASER SCANNING (TLS), https://doi.org/10.5194/isprs-archives-xlviii-1-w3-2023-153-2023, fetched 2026-10-29
