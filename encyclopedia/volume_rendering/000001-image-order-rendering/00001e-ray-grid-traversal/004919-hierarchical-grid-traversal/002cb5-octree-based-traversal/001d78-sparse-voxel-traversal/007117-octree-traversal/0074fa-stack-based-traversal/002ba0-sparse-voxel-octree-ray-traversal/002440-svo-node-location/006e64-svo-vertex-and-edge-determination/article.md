# SVO Vertex and Edge Determination

Also called: SVO Node Bounding Box Vertex/Edge Computation

## Overview
This method computes the 8 vertices and 12 edges of each Sparse Voxel Octree (SVO) node's bounding box and identifies the neighboring node indices for each geometric primitive to facilitate mesh generation [S1, unofficial]. It differs from sibling methods like [Octree Voxel Color Retrieval](5015-octree-voxel-color-retrieval.md) by focusing on geometric topology and neighbor resolution rather than data access, and from [Octree Node Lookup by Voxel](5120-octree-node-lookup-by-voxel.md) by operating on node-level bounding boxes rather than individual voxel coordinates.

## How it works
**Inputs and Outputs:**
*   **Input:** The SVO node array (containing Morton codes or node data), node origin, and node width.
*   **Output:** Deduplicated arrays of vertices (`nodeVertexArray`) and edges (`nodeEdgeArray`), where each entry stores the geometric coordinate(s) and the index of the neighboring node sharing that vertex or edge.

**Data Structures:**
*   **Lookup Tables:**
    *   `d_vertSharedLUT`: Size 64 entries. Maps `(vertex_index, neighbor_direction)` to a neighbor index. `vertex_index` $\in [0, 7]$, `neighbor_direction` $\in [0, 7]$.
    *   `d_edgeSharedLUT`: Size 48 entries. Maps `(edge_index, neighbor_direction)` to a neighbor index. `edge_index` $\in [0, 11]$, `neighbor_direction` $\in [0, 3]$.
*   **Vertex Data Layout:** `thrust::pair<Eigen::Vector3f, uint32_t>`. The first element is the 3D position $(x, y, z)$, the second is the neighbor node index.
*   **Edge Data Layout:** `thrust::pair<thrust_edge, uint32_t>`. `thrust_edge` is `thrust::pair<Eigen::Vector3f, Eigen::Vector3f>` representing start and end points. The second element is the neighbor node index.

**Numbered Steps:**

1.  **Initialization:** Create two CUDA streams for parallel execution. Stream 0 is assigned to vertex determination, Stream 1 to edge determination.
2.  **Vertex Determination Kernel (`determineNodeVertex`):**
    *   Launch kernel with `numTreeNodes` threads. Each thread `tid_x` handles one node.
    *   Retrieve the node's `origin` and `width`.
    *   Loop `i` from 0 to 7 to compute the 8 corners of the bounding box.
    *   Decode the Morton code `i` to obtain local coordinates $(x, y, z)$ using `morton3D_32_decode(i, x, y, z)`.
    *   Compute the corner position: $P = \text{origin} + \text{width} \cdot (x, y, z)$.
    *   Identify the neighbor node index:
        *   Iterate `j` from 0 to 7.
        *   Look up the candidate neighbor index in `d_vertSharedLUT[i * 8 + j]`.
        *   Select the neighbor with the minimum Morton code among these candidates.
    *   Store the pair $(P, \text{neighbor\_idx})$ in `d_nodeVertArray[tid_x * 8 + i]`.
3.  **Edge Determination Kernel (`determineNodeEdge`):**
    *   Launch kernel with `numTreeNodes` threads on Stream 1.
    *   Define the 12 edges of the node's bounding box as pairs of 3D points.
    *   Loop `i` from 0 to 11 to process each edge.
    *   Identify the neighbor node index:
        *   Iterate `j` from 0 to 3.
        *   Look up the candidate neighbor index in `d_edgeSharedLUT[i * 4 + j]`.
        *   Select the neighbor with the minimum Morton code among these candidates.
    *   Store the pair $(\text{edge\_points}, \text{neighbor\_idx})$ in `d_nodeEdgeArray[tid_x * 12 + i]`.
4.  **Deduplication:**
    *   Sort `d_nodeVertArray` by position.
    *   Apply `thrust::unique` with a comparator `uniqueVert` that checks if the 3D positions are equal.
    *   Resize `d_nodeVertArray` to the new count `numVerts` and copy to host memory `nodeVertexArray`.
    *   Sort `d_nodeEdgeArray` by edge endpoints.
    *   Apply `thrust::unique` with a comparator `uniqueEdge` that checks if edge endpoints are identical (order-independent).
    *   Resize `d_nodeEdgeArray` to the new count `numEdges` and copy to host memory `nodeEdgeArray`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper: Morton 3D Decode (Standard implementation assumed available or implemented)
def morton3D_32_decode(m, x, y, z):
    # Extract bits for x, y, z from 32-bit Morton code m
    # x = (m & 0x49249249) >> 0
    # y = (m & 0x49249249) >> 1
    # z = (m & 0x49249249) >> 2
    # This is a placeholder for the specific bit-packing logic used in the source.
    # The source uses a lookup table based decode or bit manipulation.
    # For implementation, standard Morton decoding is required.
    pass

def determine_node_vertex(node_origin, node_width, morton_code, d_vertSharedLUT):
    """
    Computes the 8 vertices for a single node and finds neighbor indices.
    Returns a list of 8 tuples: (position, neighbor_index)
    """
    vertices = []
    for i in range(8):
        # Decode local coordinates from Morton index i
        # Note: The source decodes 'i' (0-7) to get x,y,z. 
        # This implies 'i' is treated as a local Morton code for the 8 corners.
        x, y, z = morton3D_32_decode(i, 0, 0, 0) # Simplified: i is the local Morton code
        
        # Compute corner position
        # Eigen::Vector3f((float) x, (float) y, (float) z)
        local_pos = (float(x), float(y), float(z))
        corner = node_origin + node_width * local_pos
        
        # Find neighbor with minimum Morton code
        # d_vertSharedLUT[i * 8 + j] gives candidate neighbor indices
        min_morton = infinity
        best_neighbor_idx = -1
        
        for j in range(8):
            candidate_idx = d_vertSharedLUT[i * 8 + j]
            # Get Morton code of candidate_idx from the SVO node array
            candidate_morton = get_node_morton_code(candidate_idx)
            
            if candidate_morton < min_morton:
                min_morton = candidate_morton
                best_neighbor_idx = candidate_idx
                
        vertices.append((corner, best_neighbor_idx))
    return vertices

def determine_node_edge(node_origin, node_width, d_edgeSharedLUT):
    """
    Computes the 12 edges for a single node and finds neighbor indices.
    Returns a list of 12 tuples: ((start, end), neighbor_index)
    """
    # Define 12 edges as pairs of local coordinates (0 or 1)
    # Edge definitions depend on the specific bounding box vertex ordering.
    # Standard 12 edges of a cube:
    edges_local = [
        ((0,0,0), (1,0,0)), ((0,0,0), (0,1,0)), ((0,0,0), (0,0,1)),
        ((1,1,1), (0,1,1)), ((1,1,1), (1,0,1)), ((1,1,1), (1,1,0)),
        ((1,0,0), (1,1,0)), ((1,0,0), (1,0,1)),
        ((0,1,0), (0,1,1)), ((0,1,0), (0,0,1)),
        ((0,0,1), (0,1,1)), ((1,1,0), (1,0,1)) # Adjust based on actual geometry
    ]
    
    edges = []
    for i in range(12):
        start_local, end_local = edges_local[i]
        start_pos = node_origin + node_width * start_local
        end_pos = node_origin + node_width * end_local
        
        # Find neighbor with minimum Morton code
        min_morton = infinity
        best_neighbor_idx = -1
        
        for j in range(4):
            candidate_idx = d_edgeSharedLUT[i * 4 + j]
            candidate_morton = get_node_morton_code(candidate_idx)
            
            if candidate_morton < min_morton:
                min_morton = candidate_morton
                best_neighbor_idx = candidate_idx
                
        edges.append(((start_pos, end_pos), best_neighbor_idx))
    return edges

def unique_vert(a, b):
    return a.position == b.position

def unique_edge(a, b):
    # Check if endpoints are identical in either order
    return (a.start == b.start and a.end == b.end) or (a.start == b.end and a.end == b.start)
```

**Parameters:**
*   `d_vertSharedLUT`: 64 entries.
*   `d_edgeSharedLUT`: 48 entries.
*   `numTreeNodes`: Total number of nodes in the SVO.

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Nodes at the boundary of the SVO may have neighbors outside the defined tree. The neighbor lookup logic must handle invalid indices or assign a default sentinel value.
*   **Precision:** Floating-point arithmetic for corner positions may introduce slight discrepancies. Using the same origin/width for all nodes sharing a vertex is critical for deduplication.
*   **Deduplication Order:** The `thrust::unique` operation requires the input array to be sorted. The sorting key must be the geometric coordinate (for vertices) or the edge endpoints (for edges).

## Operating characteristics
*   **Time Complexity:** $O(N)$ for vertex and edge generation, where $N$ is the number of nodes. Deduplication adds $O(N \log N)$ due to sorting.
*   **Memory Complexity:** $O(N)$ for storing vertices and edges before deduplication. After deduplication, memory usage is proportional to the number of unique vertices and edges, which is typically less than $8N$ and $12N$ respectively.
*   **Performance:** Parallel execution on CUDA streams allows overlapping of vertex and edge computation. The use of lookup tables minimizes branching and arithmetic operations during neighbor resolution.
*   **Strengths:** Efficient GPU-based generation of mesh topology. Handles neighbor resolution explicitly.
*   **Limitations:** Requires pre-computed lookup tables. Deduplication requires sorting, which can be a bottleneck for very large scenes.

## Background
This method is part of the GPU-SVO implementation by Alan-Leo-Wong [S1, unofficial]. It is designed for real-time volume rendering using Sparse Voxel Octrees, specifically for generating the mesh representation of the SVO structure for ray marching or ray tracing.

## Sources
[S1] GitHub - Alan-Leo-Wong/GPU-SVO, https://github.com/Alan-Leo-Wong/GPU-SVO, fetched 2026-10-29 (unofficial)
