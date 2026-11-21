# Octree-based Voxel Terrain Voxelization

Also called: Recursive Octree Voxelization for Terrain, Surface-Intersecting Octree Terrain Voxelization

## Overview
This method constructs a voxel terrain representation by recursively subdividing an octree based on surface intersection tests. It solves the problem of efficiently generating a sparse voxel grid for procedural terrain by only allocating memory for nodes that contain or intersect the terrain surface, rather than filling a dense volume. It differs from dense voxelization by using adaptive depth, though this introduces specific edge cases regarding noise-induced gaps that require careful handling of subdivision criteria [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A terrain bounding box (typically rectangular), a surface definition function (e.g., Perlin noise heightmap), and a target voxel resolution (e.g., 1m cubed).
*   **Output:** An octree structure where leaf nodes represent either solid ground, empty air, or surface-intersecting chunks ready for meshing. Each node contains a `voxel_index` indicating its rendering state.

**Data Structures:**
*   **Octree Node:**
    *   `bounds`: Axis-aligned bounding box (AABB) defining the spatial extent.
    *   `children`: Array of 8 child nodes (null if leaf).
    *   `voxel_index`: Integer flag set by `intersects_surface()`. Values indicate if the node is fully inside, fully outside, or intersecting the surface.
    *   `depth`: Current subdivision level.

**Algorithm Steps:**

1.  **Initialize Root Nodes:**
    *   Determine the terrain dimensions. If the terrain is not a perfect cube with power-of-two dimensions, create multiple root parent nodes to cover the rectangular bounds.
    *   Example: For a $2048 \times 1024 \times 2048$ terrain, create four parent nodes, each $1024 \times 1024 \times 1024$ [S1, unofficial].

2.  **Recursive Subdivision:**
    *   Define a recursive function `Subdivide(node)`:
        1.  Call `intersects_surface(node)`. This function must return `true` if the node intersects the surface and set `node.voxel_index` accordingly.
        2.  If `intersects_surface(node)` returns `true`:
            *   If `node` is not at the maximum depth (target voxel size), create 8 child nodes.
            *   Recursively call `Subdivide(child)` for each child.
        3.  If `intersects_surface(node)` returns `false` (node is fully inside or outside):
            *   Do not create children. Mark the node as a leaf.

3.  **Alternative Bottom-Up Approach (for gap avoidance):**
    *   If gaps are unacceptable, test all nodes at the lowest required depth (LOD limit).
    *   Cache the intersection results.
    *   Work back up to the root: create parent nodes for a collection of eight children only if they represent a surface.
    *   Delete parent nodes that are completely in air or solid ground [S1, unofficial].

**Formulas and Parameters:**
*   **Voxel Size:** Typically 1m cubed, determining the minimum leaf node dimension [S1, unofficial].
*   **Terrain Dimensions:** Example given as $2048 \times 1024 \times 2048$ [S1, unofficial].

**Pitfalls and Edge Cases:**
*   **Noise-Induced Gaps:** When using random noise, a node's corners may all be inside or outside the surface, yet a child node may still intersect the surface due to the random nature of the noise. Standard corner-testing intersection checks may fail to detect this, leading to gaps in the terrain [S1, unofficial].
*   **Memory vs. Accuracy Trade-off:** Subdividing every node until the surface depth is reached avoids gaps but consumes significantly more time and memory compared to selective subdivision [S1, unofficial].
*   **Prediction Difficulty:** Predicting intersection based on noise frequency, amplitude, and octaves without sampling is complex and prone to error [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_terrain(terrain_bounds, surface_func, max_depth):
    """
    Main entry point for octree-based voxel terrain voxelization.
    """
    # Step 1: Initialize root nodes
    # If terrain is not a power-of-two cube, split into multiple roots
    root_nodes = []
    if is_power_of_two(terrain_bounds.width) and \
       is_power_of_two(terrain_bounds.height) and \
       is_power_of_two(terrain_bounds.depth):
        root = Node(terrain_bounds, depth=0)
        root_nodes.append(root)
    else:
        # Create multiple parent nodes to cover rectangular bounds
        # Implementation detail: split bounds into N cubes of power-of-two size
        root_nodes = create_multiple_roots(terrain_bounds)

    # Step 2: Recursive subdivision
    for root in root_nodes:
        subdivide(root, surface_func, max_depth)

    return root_nodes

def subdivide(node, surface_func, max_depth):
    """
    Recursively subdivides the octree node if it intersects the surface.
    """
    # Check intersection and set voxel_index
    # [G] Implementation of intersects_surface depends on the specific surface function.
    # It must return True if the node intersects the surface and set node.voxel_index.
    intersects, voxel_index = intersects_surface(node, surface_func)
    node.voxel_index = voxel_index

    if intersects:
        # If not at max depth, subdivide
        if node.depth < max_depth:
            children = create_children(node)
            for child in children:
                subdivide(child, surface_func, max_depth)
        else:
            # Reached target voxel size
            pass
    else:
        # Node is fully inside or outside; do not subdivide
        pass

def intersects_surface(node, surface_func):
    """
    Determines if a node intersects the terrain surface.
    Returns (bool intersects, int voxel_index).
    [G] This is the critical function. A robust implementation must sample
    the surface function at the node's corners and potentially center to
    detect noise-induced intersections that corner-only tests might miss.
    """
    # Example logic:
    corners = get_node_corners(node)
    values = [surface_func(c) for c in corners]
    
    # Check if surface passes through the node
    # If all values are above or below the surface threshold, no intersection
    # If mixed, intersection exists.
    # [G] To avoid gaps, also check the center or use a more robust bounding volume test
    # against the noise function's amplitude.
    
    if has_mixed_signs(values):
        return True, INTERSECTING
    elif all_above_surface(values):
        return False, INSIDE
    else:
        return False, OUTSIDE
```

## Subtopics

### Voxel Octree Terrain Generation and Meshing
The parent method that manages the overall chunk generation and meshing pipeline. It uses this voxelization step to populate the octree before generating triangle meshes for visible chunks. [Voxel Octree Terrain Generation and Meshing](15190-voxel-octree-terrain-generation-and-meshing.md)

## Operating characteristics
*   **Time Complexity:** Depends heavily on the surface complexity and the subdivision strategy. Selective subdivision is faster but risks gaps. Full subdivision to voxel size is $O(N)$ where $N$ is the number of voxels at the target resolution, which is significantly slower than sparse methods.
*   **Memory Complexity:** Sparse. Only nodes intersecting the surface or below it are allocated. However, if full subdivision is used to avoid gaps, memory usage approaches that of a dense grid.
*   **Strengths:** Efficient for large terrains with sparse surface detail. Adaptive resolution allows for high detail near the surface and coarser representation deeper underground.
*   **Limitations:** Prone to gaps when using high-frequency noise if intersection tests are not robust. Performance degrades if the surface is highly complex or if full subdivision is required.
*   **When to use:** When memory efficiency is critical and the terrain surface is relatively smooth or low-frequency.
*   **When not to use:** When using high-frequency noise and gaps are unacceptable, unless the performance cost of full subdivision is acceptable.

## Background
This method was developed as part of the Voxel Terrain system for the Ultra game engine, specifically for VR optimization. It was documented in a blog post by the developer of "The Seventh World" project. The approach addresses the challenges of generating voxel terrain efficiently for real-time rendering, balancing memory usage with visual fidelity. The specific issues with noise-induced gaps highlight the difficulties of adaptive voxelization with procedural surfaces.

## Sources
[S1] Voxel Terrain Part 1 - Voxelization, https://www.leadwerks.com/community/blogs/entry/2800-voxel-terrain-part-1-voxelization/, fetched 2026-10-29 (unofficial)
