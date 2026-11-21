# Interactive Large Iso-Surface Ray Tracing (DeMarle et al.)
Also called: DeMarle's approach, Out-of-core iso-surface ray tracing

## Overview
An out-of-core iso-surface ray tracing engine designed to render volume datasets exceeding single-machine main memory limits by leveraging a PC cluster. It employs a hierarchical grid acceleration structure with a three-level bricking approach to minimize memory overhead, enabling interactive frame rates for terabyte-scale data. It differs from its parent [Ray-marching reference (min-max octree accelerated)](1554-ray-marching-reference-min-max-octree-accelerated.md) by using a rectilinear hierarchical grid instead of an octree, and from [Object-Order Iso-Surface Rendering with Macro-Cells](1757-object-order-iso-surface-rendering-with-macro-cells.md) by relying on image-order ray traversal and distributed out-of-core data loading rather than object-order macro-cell processing.

## How it works
**Inputs:**
- Volume dataset $V$ with dimensions $D_x \times D_y \times D_z$ and voxel resolution.
- Iso-surface threshold value $T$.
- Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$).
- Cluster configuration (number of nodes, network topology).

**Outputs:**
- Rendered image tiles distributed across cluster nodes.

**Data Structures:**
1.  **Hierarchical Grid Acceleration Structure:**
    -   Three-level hierarchy.
    -   Leaf level: Macro-cells of size $8^3$ voxels.
    -   Internal nodes: Store min/max scalar values of their children.
    -   Memory overhead: Minimal; for a $2048 \times 2048 \times 1920$ dataset (7.5 GB), the structure requires only 8.5 MB per time step [S1].
2.  **Brick Cache (DataServer):**
    -   Local memory cache on each cluster node.
    -   Stores loaded voxel bricks (sub-volumes) requested by rays.
    -   Managed via shared memory and semaphores between processors on the same node [S1].

**Algorithm Steps:**

1.  **Initialization:**
    -   Master node divides the image into rectangular tiles.
    -   Tiles are distributed to render nodes. If a node has multiple processors, tiles are further divided into scan-lines [S1].

2.  **Hierarchical Ray Traversal:**
    -   Start at the top level of the hierarchical grid.
    -   Perform incremental grid walking to find the first macro-cell intersected by the ray [S1].
    -   **Empty Space Skipping:** Check the min/max values stored in the current node. If the iso-surface threshold $T$ is not within $[min, max]$, skip the entire subtree (empty space) [S1].
    -   **Recursion:** If the ray intersects a macro-cell containing the isosurface, descend to the next lower hierarchy level. Restart traversal from the top of that sub-grid [S1].
    -   Repeat until reaching the leaf level (macro-cells of $8^3$ voxels) or the ray exits the volume [S1].

3.  **Iso-Surface Intersection:**
    -   At the leaf level, the ray intersects a brick of $8^3$ voxels.
    -   Perform analytic iso-surface intersection tests within the voxel grid.
    -   Solve for intersections using Cardano's formula for cubic polynomials, assuming trilinear or linear interpolation within voxels [S1].
    -   [G] Standard trilinear interpolation yields a cubic equation for the ray parameter $t$. Cardano's formula provides the exact roots.

4.  **Data Loading (Out-of-Core):**
    -   When a ray enters a brick not present in the local DataServer cache:
        -   Request the brick from the DataServer.
        -   DataServer loads the brick from the network storage if not in memory.
        -   Cache the brick in local shared memory for subsequent rays [S1].

5.  **Rendering:**
    -   Compute shading and color at intersection points.
    -   Accumulate results into the image tile buffer.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_tile(tile, camera, iso_threshold, hierarchy, data_server):
    """
    Renders a single image tile using hierarchical ray traversal.
    """
    image = empty_image(tile.width, tile.height)
    
    for y in range(tile.y_start, tile.y_end):
        for x in range(tile.x_start, tile.x_end):
            # 1. Generate ray for pixel (x, y)
            ray = camera.generate_ray(x, y)
            
            # 2. Hierarchical Traversal
            # Start at root of hierarchy
            current_node = hierarchy.root
            
            while current_node is not None:
                # Check if ray intersects current node's bounding box
                if not ray_intersects_bbox(ray, current_node.bbox):
                    current_node = None # Ray misses this branch
                    break
                
                # Empty space check using min/max values
                if not (current_node.min_val <= iso_threshold <= current_node.max_val):
                    current_node = None # Skip empty space
                    break
                
                # If leaf node, perform intersection test
                if current_node.is_leaf:
                    # Load brick if not in cache
                    brick = data_server.get_brick(current_node.brick_id)
                    
                    # Analytic intersection using Cardano's formula
                    # Assuming linear interpolation within the 8^3 macro-cell
                    intersections = solve_cardano_intersections(
                        ray, brick, iso_threshold
                    )
                    
                    if intersections:
                        # Pick closest intersection
                        t = min(intersections)
                        point = ray.origin + t * ray.direction
                        color = compute_shading(point, ray, brick)
                        image.set_pixel(x, y, color)
                    
                    current_node = None # Done with this ray
                    break
                
                # Descend to children
                # Find the child that the ray enters first
                children = current_node.children
                next_node = None
                min_t = infinity
                
                for child in children:
                    if ray_intersects_bbox(ray, child.bbox):
                        t = ray_t_intersect(ray, child.bbox)
                        if t < min_t:
                            min_t = t
                            next_node = child
                
                current_node = next_node

    return image
```

**Parameters:**
-   Macro-cell size: $8^3$ voxels at leaf level [S1].
-   Hierarchy depth: 3 levels [S1].
-   Iso-surface threshold: User-defined scalar value [S1].

**Edge Cases and Pitfalls:**
-   **Precision:** Analytic solution via Cardano's formula requires careful handling of floating-point precision near degenerate cases (e.g., ray tangent to surface) [S1].
-   **Boundary Conditions:** Ray exit from the volume must be explicitly checked to avoid infinite traversal [S1].
-   **Network Latency:** Out-of-core loading introduces latency; efficient bricking and caching strategies are critical to maintain performance [S1].

## Operating characteristics
-   **Time Complexity:** Depends on the number of macro-cells traversed. Empty space skipping reduces average case complexity. Intersection test is $O(1)$ per leaf cell due to analytic solution [S1].
-   **Memory Complexity:** Very low for acceleration structure (8.5 MB for 7.5 GB dataset) [S1]. DataServer cache size depends on working set of bricks.
-   **Measured Performance:**
    -   Visible Female (428 MB, 512x512 resolution): ~1 fps on 1 node, up to 22 fps on 32 nodes (linear scaling) [S1].
    -   LLNL (7.5 GB, 512x512 resolution): 2.2 to 6.7 fps on 32 nodes, depending on iso-surface complexity [S1].
-   **Strengths:**
    -   Handles datasets too large for single-machine memory.
    -   Low memory overhead for acceleration structure.
    -   Linear scalability with cluster size [S1].
-   **Limitations:**
    -   Slower than kd-tree based approaches (e.g., Wald's) for comparable hardware (5x slower than Wald's 8.1 fps on single PC for Visible Female) [S1].
    -   Performance sensitive to iso-surface complexity and network latency [S1].
-   **When to use:** Large-scale scientific visualization where data exceeds single-node memory and cluster resources are available.
-   **When not to use:** Single-machine environments with sufficient memory, or when higher frame rates are required on limited hardware compared to kd-tree methods [S1].

## Background
-   **Origin:** Presented by DeMarle et al. in 2003 [S1].
-   **Development:** Built upon the *-Ray ray tracing engine by Parker et al., originally designed for parallel shared memory SGI Onyx 2000 systems [S1]. Extended to out-of-core rendering on PC clusters.
-   **Usage:** Used for interactive visualization of large-scale scientific simulations, such as the LLNL Richtmyer-Meshkov instability simulation [S1].
-   **Comparison Context:** Compared against Wald's kd-tree approach and Neubauer's hierarchical grid method [S1].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
