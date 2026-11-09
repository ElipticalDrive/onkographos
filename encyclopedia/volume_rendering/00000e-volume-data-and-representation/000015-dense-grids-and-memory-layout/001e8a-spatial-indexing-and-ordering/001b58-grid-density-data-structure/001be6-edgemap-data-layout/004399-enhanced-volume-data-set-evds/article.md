# Enhanced Volume Data Set (EVDS)
## Overview
An Enhanced Volume Data Set (EVDS) is a volumetric representation that augments standard voxel grids with Boolean edge labels to encode local connectivity between vertices [S1, unofficial]. It solves the problem of separating segmentation from geometric modeling by providing a topological map of which vertices belong to the same underlying object [S1, unofficial]. Unlike standard dense grids, EVDS explicitly tracks whether two adjacent vertices share an active connection, enabling efficient topological analysis and hierarchical compression via L-blocks [S1, unofficial]. It builds on [EdgeMap Data Layout](1109-edgemap-data-layout.md) by defining the specific data structure for storing these edge states.
## How it works
### Inputs and outputs
- **Input**: A 3D volumetric grid of vertices with associated scalar values (voxel data) and a segmentation decision function.
- **Output**: An Enhanced Volume Data Set consisting of vertex data and Boolean edge labels, optionally organized into L-blocks.

### Data structures
**Vertex Structure**:
Each vertex in the grid stores:
1.  **Voxel Value(s)**: The scalar intensity or material property.
2.  **Edge Label Vector**: A Boolean vector indicating the activity status of edges emanating from the vertex. The size of this vector depends on the chosen connectivity level (see Parameters).

**L-block Structure** (Optional Hierarchical Layout):
An L-block is a 3-dimensional iso-rectangular block of enhanced vertex information [S1, unofficial]. It consists of two parts:
1.  **Header**:
    -   `position`: The $(x, y, z)$ coordinates of the least vertex (minimum corner) of the block.
    -   `template`: A tuple $(l_1, l_2, l_3)$ representing the number of vertices in the $x$, $y$, and $z$ directions, respectively.
2.  **Vertex Block**: Contains the enhanced vertex information (voxel values and edge labels) for all vertices within the isorectangular block defined by the header [S1, unofficial].

### Parameters
-   **Connectivity Level**: Defines the number of edges emanating from a vertex.
    -   **3-connectivity**: Edges along axis directions only. Vertex $(i,j,k)$ connects to $(i+1,j,k)$, $(i,j+1,k)$, and $(i,j,k+1)$. [S1, unofficial]
    -   **9-connectivity**: Includes 3-connectivity plus connections across the edges of the axis-aligned cube surrounding the vertex. [S1, unofficial]
    -   **13-connectivity**: Includes 9-connectivity plus connections across the corners of the axis-aligned cube. [S1, unofficial]
-   **Threshold Test**: Used to determine "whitespace." A vertex is considered whitespace if it does not satisfy a threshold test, often equivalent to checking if any edges leading to or from the vertex are active [S1, unofficial].

### Steps
1.  **Segmentation**: Apply a decision function to the raw voxel data to determine connectivity. This function assigns a Boolean label (1 for active, 0 for inactive) to selected edges between vertices [S1, unofficial].
    -   *Note*: The decision function is essentially the segmentation process. Conservative labeling functions are often used initially to segment a superset of critical data, allowing for later adjustment [S1, unofficial].
2.  **Edge Labeling**: For each vertex, evaluate the edges emanating from it based on the chosen connectivity level.
    -   If an edge connects two vertices that belong to the same underlying object (as determined by the segmentation), label it 1 (active).
    -   Otherwise, label it 0 (inactive).
3.  **Boundary Handling**: Vertices at the boundary of the grid lack some edges. Treat these missing edges as inactive (0) [S1, unofficial].
4.  **Storage**: Store the voxel value and the Boolean edge label vector for each vertex.
5.  **L-block Construction** (Optional): Group vertices into iso-rectangular blocks.
    -   Identify connected regions of vertices.
    -   Pack contiguous blocks of vertices into L-blocks.
    -   Store the header (position and template) and the vertex block data.
    -   Store connected L-blocks in a hierarchical data structure called an L-block covering (LBC), similar to an AABB-tree [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_evds(grid_values, connectivity_level=3):
    """
    Creates an Enhanced Volume Data Set from a 3D grid of values.
    
    Args:
        grid_values: 3D array of voxel values.
        connectivity_level: 3, 9, or 13.
        
    Returns:
        evds: A list of Vertex objects or L-blocks.
    """
    dims = grid_values.shape
    vertices = []
    
    # Define connectivity offsets based on level
    if connectivity_level == 3:
        # Only positive axis directions to avoid double counting, 
        # but storage is per-vertex, so we store all emanating edges.
        # For 3-connectivity, a vertex has 3 outgoing edges in a standard grid 
        # if we consider directed edges, or 6 if undirected. 
        # The source implies "edges emanating from a vertex". 
        # We assume storage of all incident edges or a canonical set.
        # Based on "Lego block" analogy and 3-connectivity definition:
        # (i,j,k) connects to (i+1,j,k), (i,j+1,k), (i,j,k+1).
        # This suggests storing 3 boolean flags per vertex for outgoing edges.
        edge_offsets = [(1,0,0), (0,1,0), (0,0,1)]
    elif connectivity_level == 9:
        # 3-axis + 3 edge diagonals
        edge_offsets = [
            (1,0,0), (0,1,0), (0,0,1),
            (1,1,0), (1,0,1), (0,1,1)
        ]
    elif connectivity_level == 13:
        # 3-axis + 3 edge diagonals + 3 corner diagonals
        edge_offsets = [
            (1,0,0), (0,1,0), (0,0,1),
            (1,1,0), (1,0,1), (0,1,1),
            (1,1,1)
        ]
    else:
        raise ValueError("Connectivity level must be 3, 9, or 13")

    for i in range(dims[0]):
        for j in range(dims[1]):
            for k in range(dims[2]):
                value = grid_values[i, j, k]
                labels = []
                
                for dx, dy, dz in edge_offsets:
                    ni, nj, nk = i + dx, j + dy, k + dz
                    
                    # Check boundary
                    if 0 <= ni < dims[0] and 0 <= nj < dims[1] and 0 <= nk < dims[2]:
                        # Decision function: determine if edge is active
                        # This is the segmentation step.
                        # Conservative labeling: assume active if both endpoints meet threshold?
                        # Source says: "decision function... is essentially the segmentation process"
                        # We assume a function `is_connected(v1, v2)` exists.
                        active = is_connected(value, grid_values[ni, nj, nk])
                        labels.append(active)
                    else:
                        # Boundary edges are inactive
                        labels.append(False)
                
                vertices.append(Vertex(value, labels))
                
    return vertices

def is_connected(val1, val2):
    """
    Placeholder for the segmentation decision function.
    In practice, this uses statistical analysis or manual editing thresholds.
    """
    # Example: simple threshold
    return val1 > 0.5 and val2 > 0.5

class Vertex:
    def __init__(self, value, edge_labels):
        self.value = value
        self.edge_labels = edge_labels # List of Booleans

class LBlock:
    def __init__(self, position, template, vertices):
        self.position = position # (x, y, z)
        self.template = template # (l1, l2, l3)
        self.vertices = vertices # List of Vertex objects
```

### Edge cases and pitfalls
-   **Boundary Vertices**: Vertices at the grid boundary lack edges. These missing edges must be treated as inactive (0) [S1, unofficial].
-   **Decision Function Quality**: The quality of the segmentation (decision function) directly determines the faithfulness of the reconstruction. Poor segmentation leads to incorrect connectivity [S1, unofficial].
-   **Whitespace Definition**: Whitespace is defined as vertices that do not satisfy a threshold test. This test is often equivalent to checking if any edges leading to or from the vertex are active. Misinterpretation of this definition can lead to incorrect isolation of objects [S1, unofficial].
-   **Connectivity Choice**: Higher connectivity levels (9, 13) capture more complex shapes but increase storage and computational cost. 3-connectivity is sufficient for axis-aligned structures (Lego-like) [S1, unofficial].

## Subtopics

### L-block (LB)
An L-block is a 3-dimensional iso-rectangular block of enhanced vertex information, defined by a template $(l_1, l_2, l_3)$ representing the number of vertices in the $x$, $y$, and $z$ directions [S1, unofficial]. It is used to compress the EVDS by grouping connected vertices. A single voxel is a $(1,1,1)LB$ [S1, unofficial]. L-blocks sharing an active edge are assumed to be connected and are stored in a hierarchical structure called an L-block covering (LBC) [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: Depends on the segmentation decision function. If the decision function is $O(1)$ per edge, the construction is $O(N \cdot C)$ where $N$ is the number of vertices and $C$ is the connectivity level.
-   **Memory Complexity**: $O(N \cdot C)$ bits for edge labels plus $O(N)$ for voxel values. L-blocks add overhead for headers but reduce storage for large homogeneous regions if the LBC is used.
-   **Strengths**: Provides explicit topological connectivity, enabling efficient topological analysis and hierarchical compression. Separates segmentation from visualization.
-   **Limitations**: Relies heavily on the quality of the segmentation decision function. Higher connectivity levels increase storage.
-   **When to use**: When topological analysis or hierarchical compression of volumetric data is required, especially for filamentary or structured data.
-   **When not to use**: For simple visualization where connectivity is not needed, or when the segmentation is unreliable.

## Background
The Enhanced Volume Data Set (EVDS) was introduced by McCormick et al. in the context of compression, segmentation, and modeling of filamentary volumetric data [S1, unofficial]. It builds on earlier work in digital topology [KR89] and advanced methods for describing digital topology [EHV*03], but focuses on basic local connectivity. The concept of L-blocks and L-block covering (LBC) was developed to efficiently store and process these enhanced datasets [S1, unofficial].

## Sources
[S1] Compression, Segmentation, and Modeling of Filamentary Volumetric Data, https://doi.org/10.2312/sm.20041411, fetched 2026-10-26 (unofficial)
