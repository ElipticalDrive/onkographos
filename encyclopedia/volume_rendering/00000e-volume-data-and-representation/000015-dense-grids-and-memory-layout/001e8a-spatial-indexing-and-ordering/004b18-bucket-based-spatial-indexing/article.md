# Bucket-based Spatial Indexing

Also called: BB index lookup, Bucket-based Spatial Indexing for Primitive Rendering, Grid-based bucket mapping

## Overview
Bucket-based spatial indexing maps 3D primitives to a 3D grid of buckets using bounding box indices, enabling fast spatial queries during rendering by iterating only over relevant buckets [S1, unofficial]. It differs from arithmetic stride mapping by grouping primitives into discrete spatial cells rather than relying on linear memory layout, and from space-filling curves by using axis-aligned bounding box (AABB) discretization instead of bit-interleaved coordinate ordering [S1, unofficial].

## How it works
**Inputs:** A set of 3D primitives, each defined by a bounding box $[minV, maxV]$ where $minV, maxV \in \mathbb{R}^3$, and a spatial grid configuration (implied by the bucket mapping structure).
**Outputs:** A hash map `mapping` where keys are integer bucket indices and values are lists of `UpdatePattern` objects containing rendering data.

**Data Structures:**
*   `UpdatePatternPoint`: A structure containing:
    *   `pos`: `Vec3<float>` representing the 3D position.
    *   `pointDisplayParams`: Metadata for display parameters.
    *   `normal`: `Vec3<float>` representing the surface normal.
    *   Other display metadata fields.
*   `UpdatePattern`: A container (e.g., `std::vector<UpdatePatternPoint>`) holding points associated with a specific bucket.
*   `mapping`: A hash map (`std::unordered_map<int, UpdatePattern>`) keyed by integer bucket indices.

**Algorithm Steps:**
1.  **Compute Bucket Indices:** For each primitive, calculate the 3D bounding box indices corresponding to its spatial extent $[minV, maxV]$. This is done via a function `calculateIndicesFromBB(params, minV, maxV)` which maps the continuous bounding box coordinates to discrete integer grid coordinates $(i, j, k)$ [S1, unofficial].
2.  **Iterate Buckets:** For each computed bucket index $(i, j, k)$:
    a.  Compute the linear or composite key for the bucket.
    b.  Look up the key in the `mapping` hash map.
    c.  If the key is not found (`it == mapping.end()`), skip to the next bucket [S1, unofficial].
    d.  Retrieve the `UpdatePattern` (list of points) associated with this bucket.
3.  **Process Points:** For each `UpdatePatternPoint` in the retrieved bucket:
    a.  Apply geometric clipping or thickness checks against the primitive's bounds.
    b.  If the point is within the valid bounds, add the rendering data to the output `render` list [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bucket_based_spatial_index(primitives, grid_params):
    """
    Maps primitives to buckets and retrieves relevant points for rendering.
    primitives: List of objects with 'minV' (Vec3) and 'maxV' (Vec3)
    grid_params: Configuration for bucket size and offset (implicit in calculateIndicesFromBB)
    """
    mapping = {}  # Hash map: int -> List[UpdatePatternPoint]

    # Step 1: Populate the bucket map
    for primitive in primitives:
        minV = primitive.minV
        maxV = primitive.maxV
        
        # Calculate bucket indices for the primitive's bounding box
        # Returns a list of integer bucket indices covering the primitive's extent
        bucket_indices = calculate_indices_from_bb(grid_params, minV, maxV)
        
        for bucket_idx in bucket_indices:
            if bucket_idx not in mapping:
                mapping[bucket_idx] = []
            
            # Assuming primitive provides points or patterns to store
            # In the source context, UpdatePatternPoints are stored in the bucket
            # This step depends on how primitives are decomposed into points/patterns
            # For the purpose of this index structure, we assume points are added here
            # Note: The source notes imply the mapping stores UpdatePatternPoints
            # directly or via UpdatePattern containers.
            # We add placeholder points for the primitive to the bucket.
            # In a real implementation, this would involve decomposing the primitive
            # into its constituent UpdatePatternPoints.
            points = get_points_for_primitive(primitive)
            for pt in points:
                mapping[bucket_idx].append(pt)

    # Step 2: Query and Render
    render_list = []
    
    # Iterate over all populated buckets (or query specific regions)
    for bucket_idx, points in mapping.items():
        # Step 3: Process points in the bucket
        for pt in points:
            # Apply geometric clipping/thickness checks
            if is_point_valid(pt, primitive.bounds):
                render_list.append(pt)
                
    return render_list

def calculate_indices_from_bb(params, minV, maxV):
    """
    Maps bounding box coordinates to bucket indices.
    params: Grid parameters (cell size, origin, etc.)
    minV, maxV: Bounding box corners
    Returns: List of integer bucket indices
    """
    # Implementation details depend on specific grid discretization
    # Typically involves floor((coord - origin) / cell_size)
    # Returns all integer indices (i, j, k) that the box overlaps
    pass

def is_point_valid(pt, bounds):
    """
    Checks if a point is within the primitive's bounds, considering thickness.
    """
    # Check if point is within outer bounds
    if not (bounds.minV.x < pt.pos.x < bounds.maxV.x and
            bounds.minV.y < pt.pos.y < bounds.maxV.y and
            bounds.minV.z < pt.pos.z < bounds.maxV.z):
        return False
    
    # Apply thickness clipping if applicable
    if bounds.thickness > 0:
        inner_min = bounds.minV + bounds.thickness
        inner_max = bounds.maxV - bounds.thickness
        # Skip if strictly inside inner bounds
        if (inner_min.x < pt.pos.x < inner_max.x and
            inner_min.y < pt.pos.y < inner_max.y and
            inner_min.z < pt.pos.z < inner_max.z):
            return False
            
    return True
```

**Parameters:**
*   `grid_params`: Defines the spatial resolution and origin of the bucket grid. Not given in the sources.
*   `thickness`: Float value defining the wall thickness for hollow cuboids. Typical value: 0.8 (as seen in source example) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Missing Buckets:** Bucket lookup may fail if the primitive's bounding box falls outside pre-populated grid cells or if `calculateIndicesFromBB` produces indices not present in `mapping`. This is handled by checking `if (it == mapping.end()) continue;` [S1, unofficial].
*   **Precision:** Floating-point comparisons for bounds and thickness checks must handle edge cases where points lie exactly on boundaries.

## Subtopics

### Hollow Cuboid Rendering with Thickness Clipping
Renders a cuboid as a hollow box by excluding points that lie strictly inside the inner bounds offset by `thickness` from each face [S1, unofficial]. It is chosen when rendering wireframe-like or shell structures where the interior volume should not be rendered. The key distinction is the additional thickness check: a point is rendered only if it is within the outer bounds $[minV, maxV]$ but *not* strictly inside the inner box $[minV + thickness, maxV - thickness]$ [S1, unofficial]. The `CuboidGeometry` structure contains corners `v1`, `v2` (Vec3), `thickness` (float), and `isWireframe` (bool) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Building the index is $O(N \cdot B)$ where $N$ is the number of primitives and $B$ is the average number of buckets a primitive overlaps. Querying is $O(K)$ where $K$ is the number of points in the queried buckets. Hash map lookups are $O(1)$ on average.
*   **Memory Complexity:** $O(P)$ where $P$ is the total number of points stored in the buckets. The hash map overhead adds constant factor memory usage.
*   **Strengths:** Fast spatial filtering; only relevant buckets are processed. Simple to implement with standard hash maps.
*   **Limitations:** Hash map overhead; potential for cache misses due to non-contiguous memory access in hash maps. Performance degrades if primitives are very large (overlapping many buckets) or if the grid resolution is too fine (many empty buckets).
*   **When to use:** When primitives are sparse and spatial locality is important for culling. When primitives have complex shapes that can be approximated by bounding boxes.
*   **When not to use:** When primitives are dense and cover most of the grid (hash map overhead dominates). When strict spatial coherence is required (use SFCs instead).

## Background
Originating from practical implementations in volumetric display and real-time rendering systems, this method is documented in the `Volumetric-display` repository [S1, unofficial]. It is used in applications requiring efficient rendering of complex 3D primitives, such as Minecraft-like block worlds or volumetric visualization tools, where spatial partitioning is crucial for performance [S1, unofficial].

## Sources
[S1] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
