# Adaptive Marching Cubes

Also called: AR Isosurface Algorithm, Adaptive Marching Cubes on Multiresolution Octree, Adaptive Marching Cubes on Octree

## Overview
Adaptive Marching Cubes is an extension of the standard Marching Cubes algorithm designed to extract isosurfaces from Adaptive Resolution (AR) octree data [S1, unofficial]. It solves the problem of topological inconsistencies and geometric artifacts that arise when adjacent cubes in the mesh have different resolutions [S1, unofficial]. Unlike uniform marching cubes, it handles missing vertices and discontinuities by deferring the processing of coarser nodes that conflict with finer neighbors, ensuring a watertight mesh across resolution boundaries [S1, unofficial].

## How it works
The algorithm performs two passes over the octree to resolve inconsistencies between adjacent cells of differing resolutions [S1, unofficial].

### Inputs and Outputs
*   **Input:** An adaptive resolution octree representing a volume dataset. Each node contains voxel values at its specific resolution level.
*   **Output:** A triangle mesh representing the isosurface.

### Data Structures
Each octree node maintains the following fields:
1.  **Triangle List:** A list of triangles generated for this node. Each triangle references vertices in the global Intersection List.
2.  **Intersection List:** A global vertex array storing edge intersection points. Vertices in this list may be updated later during the second pass [S1, unofficial].
3.  **Second Pass List:** A list of octree nodes that require post-processing. Nodes are added to this list if they have missing vertices relative to finer neighbors [S1, unofficial].

### Algorithm Steps

**First Pass: Standard MC with Conflict Detection**
1.  Traverse the octree in any order (typically depth-first or breadth-first).
2.  For each node, perform standard Marching Cubes using the node's voxel values.
3.  For every edge intersection found:
    *   Check if the adjacent neighbor (if it exists and is coarser) has a corresponding vertex on that edge.
    *   If the coarser neighbor has no vertex on the shared edge (i.e., its coarse vertices are on the same side of the isosurface), but the current finer cube finds an intersection:
        *   Create a new entry in the coarser neighbor's fine edge vertex list (stored in the global Intersection List) [S1, unofficial].
        *   Add the coarser neighbor node to the **Second Pass List** [S1, unofficial].
    *   If the coarser neighbor already has a vertex, use it to ensure consistency.
4.  Generate triangles for the current node using the standard MC lookup table, referencing vertices from the Intersection List.

**Second Pass: Subdivision and Finalization**
1.  Process all nodes in the **Second Pass List**.
2.  For each node in the list:
    *   Check if it has two vertices on the same edge in its fine edge vertex list (indicating a conflict where the coarse resolution failed to capture the isosurface crossing) [S1, unofficial].
    *   If such a conflict exists, **subdivide** the coarser cube. This involves interpolating between the known cube vertex values at the coarser resolution to create finer children, which are then processed using the standard MC algorithm [S1, unofficial].
    *   If no subdivision is needed but the node has facial vertices that were deferred, generate the triangles [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global data structures
intersection_list = []  # Array of vertex positions
second_pass_list = []   # List of octree nodes requiring post-processing

def adaptive_marching_cubes(root_node, isovalue):
    # First Pass
    process_node(root_node, isovalue, is_second_pass=False)
    
    # Second Pass
    while second_pass_list is not empty:
        node = second_pass_list.pop()
        process_node(node, isovalue, is_second_pass=True)

def process_node(node, isovalue, is_second_pass):
    if is_second_pass:
        # Check for conflicts requiring subdivision
        if node.has_edge_conflict():
            # Subdivide the coarse cube
            children = node.subdivide()
            for child in children:
                process_node(child, isovalue, is_second_pass=False)
            return
    
    # Standard Marching Cubes logic
    triangles = []
    for edge in node.edges:
        # Determine if isosurface crosses this edge
        if isovalue_between(node.vertices[edge.v1], node.vertices[edge.v2]):
            # Check for consistency with coarser neighbor
            neighbor = node.get_coarser_neighbor(edge)
            if neighbor is not None:
                if not neighbor.has_vertex_on_edge(edge):
                    # Missing vertex in coarser neighbor
                    # Create new vertex in intersection list
                    vertex = interpolate_intersection(edge, isovalue)
                    neighbor.add_fine_vertex(edge, vertex)
                    # Add coarser neighbor to second pass list if not already there
                    if neighbor not in second_pass_list:
                        second_pass_list.append(neighbor)
                else:
                    vertex = neighbor.get_vertex_on_edge(edge)
            else:
                vertex = interpolate_intersection(edge, isovalue)
            
            # Store vertex reference for triangle generation
            edge.vertex = vertex

    # Generate triangles based on MC case
    triangles = generate_mc_triangles(node, isovalue)
    node.triangle_list.extend(triangles)
```

### Parameters and Edge Cases
*   **Isosurface Value:** The threshold value for extraction.
*   **Edge Cases:**
    *   **Missing Vertices:** Occur when a finer cube intersects an edge that a coarser neighbor does not. Handled by creating deferred vertices and subdividing the coarser node [S1, unofficial].
    *   **Precision:** Floating-point precision issues may cause inconsistent edge crossings. Standard MC robustness techniques should be applied [S1, unofficial].
    *   **Boundary Conditions:** Nodes at the boundary of the volume have no coarser neighbors in certain directions; these are handled as standard uniform nodes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the octree depth and the number of conflicts. In the worst case, many coarser nodes may be subdivided, approaching the cost of uniform high-resolution marching cubes. However, large uninteresting regions are skipped, reducing computation [S1, unofficial].
*   **Memory Complexity:** Higher than uniform MC due to the octree structure and the need to store deferred vertices and the second pass list. The intersection list grows dynamically [S1, unofficial].
*   **Strengths:** Efficiently handles large datasets by focusing resolution where needed; produces watertight meshes across resolution boundaries [S1, unofficial].
*   **Limitations:** The two-pass nature and potential for frequent subdivision can complicate implementation and reduce performance in highly heterogeneous data [S1, unofficial].

## Background
The Adaptive Marching Cubes algorithm was introduced by Westermann et al. as part of their work on isosurface rendering of adaptive resolution data [S1, unofficial]. It addresses the limitations of standard marching cubes when applied to multiresolution data structures, specifically the topological inconsistencies at resolution boundaries [S1, unofficial]. The method has been cited in subsequent works on implicit surface rendering and interactive visualization [S2, unofficial].

## Sources
[S1] IsoSurface Rendering of Adaptive Resolution Data, https://www.cs.unh.edu/~rdb/papers/ARiso.pdf, fetched 2026-10-29 (unofficial)
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
