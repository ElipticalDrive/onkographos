# Terrain Multiresolution Representation

Also called: BTT, Bintree Triangulation, Bintree hierarchy with 4-8 subdivision, Longest Edge Refinement Algorithms, Regular grid connectivity patches, Right-Triangle Multiresolution Bintree

## Overview
Terrain Multiresolution Representation (TMR) is a method for adaptive, view-dependent level-of-detail (LOD) rendering of regular height field terrain using a Binary Triangle Tree (BTT) hierarchy [S1][S2, unofficial]. It solves the performance-quality trade-off in real-time terrain rendering by dynamically refining the mesh density based on the observer's distance, ensuring high resolution near the viewer and lower resolution at a distance [S2, unofficial]. Unlike Quad-Tree approaches which rely on recursive square subdivisions, BTT is inherently triangle-based, splitting each triangle into two right-triangle children via longest-edge bisection, which provides a more natural fit for triangle-strip rendering and continuous mesh generation [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A regular height field terrain data (grid of elevation values), horizontal post spacing ($\Delta x, \Delta y$), vertical scale factor ($\Delta z$), and an adaptive tolerance parameter $t$ dependent on distance to the observer [S1][S2, unofficial].
*   **Output:** A conforming triangular mesh with variable resolution, free of cracks, suitable for rendering [S1][S2, unofficial].

**Data Structures:**
The terrain is divided into $N$ square patches [S2, unofficial]. Each patch is represented by a Bintree hierarchy [S1].
*   **Patch Structure:** Each patch contains two Bintree triangle structures at the coarsest level (initially two triangles) [S2, unofficial].
*   **Triangle Node:** Each triangle node in the Bintree contains pointers to its three neighboring triangles [S2, unofficial].
*   **Vertex Pool:** A finite pool of possible vertices is defined to prevent memory exhaustion. Once a vertex is introduced into the mesh, it is removed from the pool [S2, unofficial].

**Formulas:**
For a patch with $n$ rows of triangles in its interior:
*   Number of vertices: $V = \frac{(n + 1)(n + 2)}{2}$ [S1].
*   Number of triangles: $T = n^2$ [S1].
*   Terrain dimensions are limited to $(2mn + 1)^2$ for non-negative integer $m$ [S1].

**Algorithm Steps:**
1.  **Initialization:** Divide the terrain data into $N$ square patches [S2, unofficial]. Initialize each patch with two Bintree triangle structures at the coarsest level. Assign height coordinates to vertices from the height map [S2, unofficial]. Link all triangles by pointers to their three neighbors [S2, unofficial].
2.  **Split (Refinement):** Each right triangle in the Bintree is split by its longest edge, producing two right-triangle children (left and right) [S2, unofficial]. Adaptive local splitting is performed on couples of triangles of the same Bintree level that share a longest edge [S2, unofficial].
3.  **Propagation (Crack Prevention):** If a target triangle has a longest-edge neighbor of a different level, splitting propagation is required [S2, unofficial]. The algorithm follows the Longest Edge Propagation Path (LEPP) to a terminal edge (a base edge for two adjacent triangles or a mesh border edge) [S2, unofficial]. Only after reaching the terminal edge does the splitting process begin, ensuring mesh conformity [S2, unofficial].
4.  **Adaptive Control:** The decision to refine is based on an adaptive tolerance parameter $t$, which depends on the distance of the point to the observer [S2, unofficial]. Triangles closer to the observer are refined more aggressively [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_terrain(height_map, patch_size):
    # Divide terrain into N square patches
    patches = []
    for each patch in terrain:
        # Initialize with two Bintree triangles at coarsest level
        patch.triangles = [Triangle(v1, v2, v3), Triangle(v4, v5, v6)]
        # Link triangles by pointers to neighbors
        link_neighbors(patch.triangles)
        patches.append(patch)
    return patches

def tessellate_frame(patches, observer_position, tolerance_func):
    for patch in patches:
        # Adaptive view-dependent LOD tessellation
        for triangle in patch.triangles:
            # Calculate distance-based tolerance
            t = tolerance_func(triangle.center, observer_position)
            
            # Check if refinement is needed based on tolerance
            if triangle.error > t:
                # Check for level mismatch with neighbors
                if triangle.has_level_mismatch_neighbor():
                    # Perform splitting propagation via LEPP
                    terminal_edge = find_lepp(triangle)
                    split_at_terminal(terminal_edge)
                
                # Perform local split
                split_triangle(triangle)
                
                # Update neighbor links
                update_neighbor_links(triangle)

def split_triangle(triangle):
    # Split by longest edge into left and right right-triangle children
    longest_edge = triangle.get_longest_edge()
    new_vertex = allocate_from_vertex_pool()
    
    # Create children
    left_child = Triangle(triangle.v1, new_vertex, triangle.v2)
    right_child = Triangle(new_vertex, triangle.v3, triangle.v2) # Adjusted for right-triangle geometry
    
    # Replace parent with children in Bintree
    triangle.replace_with_children(left_child, right_child)
    
    # Remove new_vertex from pool
    vertex_pool.remove(new_vertex)

def find_lepp(triangle):
    # Follow Longest Edge Propagation Path to terminal edge
    current = triangle
    while not is_terminal_edge(current.longest_edge):
        current = current.longest_edge_neighbor
    return current.longest_edge

def is_terminal_edge(edge):
    # Terminal edge is base edge for two adjacent triangles or mesh border
    return edge.is_border() or edge.is_shared_by_two_triangles()
```

**Parameters:**
*   $n$: Number of rows of triangles in a patch interior. Can be any integer, not just power of two [S1].
*   $t$: Adaptive tolerance parameter, dependent on distance to observer [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Exhaustion:** If refinement continues indefinitely, memory may be exhausted [S2, unofficial]. This is mitigated by the finite vertex pool; once the pool is empty, no further refinement is possible [S2, unofficial].
*   **Cracks:** Failure to perform LEPP propagation when neighbors have different levels results in cracks in the mesh [S1][S2, unofficial].

## Subtopics

### [Uncoupled Parallel BTT LOD](12577-uncoupled-parallel-btt-lod.md)
A parallel technique for view-dependent continuous LOD rendering of regular height field terrain using BTT. It solves high CPU overhead by splitting the terrain mesh into two independent halves along the observer's line of sight, allowing simultaneous tessilation on separate processors. Unlike concurrent vertex insertion, it decouples the mesh to avoid race conditions while maintaining continuity through boundary consistency enforcement.

## Operating characteristics
*   **Time Complexity:** Adaptive refinement allows for dynamic control of triangle count, optimizing rendering time based on view distance [S2, unofficial].
*   **Memory Complexity:** Limited by the finite vertex pool and the Bintree structure size [S2, unofficial].
*   **Strengths:** Inherently triangle-based, making it better suited for triangle-strip rendering than Quad-Trees [S2, unofficial]. Supports arbitrary patch interior tessellation with regular grid connectivity [S1].
*   **Limitations:** Memory exhaustion risk if vertex pool is not managed [S2, unofficial]. Requires complex neighbor linking and LEPP propagation for crack-free meshes [S2, unofficial].
*   **When to use:** Real-time terrain rendering where view-dependent LOD is critical and hardware supports dynamic mesh generation [S2, unofficial].
*   **When not to use:** Scenes with fixed resolution requirements or where memory constraints are extremely tight without a managed vertex pool [S2, unofficial].

## Background
The Terrain Multiresolution Representation is based on the right-triangle multiresolution bintree algorithm discussed in Duchaineau et al. (1997) [S2, unofficial]. This algorithm is a special case of longest edge refinement algorithms for general triangulations discussed in Rivara (1984, 1997) [S2, unofficial]. It was developed to address the specific problem of terrain rendering, where the horizon of visibility is much larger than in other real-time rendering scenarios, requiring maximization of triangle density near the observer and reduction at a distance [S2, unofficial].

## Sources
[S1] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S2] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/Papers/2007/20832/, fetched 2026-10-30 (unofficial)
