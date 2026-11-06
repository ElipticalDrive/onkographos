# Adaptive Resolution Data Generation
Also called: AR Data Generation

## Overview
Adaptive Resolution (AR) Data Generation creates a multiresolution volume representation by recursively subdividing the data space into an octree, where leaf nodes store data at varying resolutions based on local error metrics [S1, unofficial]. It differs from fixed-grid downsampling by pruning low-variance regions, thereby reducing memory and rendering costs while preserving detail in high-variance areas [S1, unofficial].

## How it works
**Inputs:** A uniform scalar volume dataset $V$ of resolution $N \times N \times N$ (where $N=2^k$), and a scalar threshold $\Delta$.
**Outputs:** An octree structure where each node represents a cubic sub-volume. Leaf nodes store the scalar values for their respective resolution level.

**Data Structures:**
The data is stored in an octree. Each node contains:
1.  `is_leaf`: Boolean flag.
2.  `children`: Array of 8 pointers to child nodes (only if `is_leaf` is false).
3.  `data`: Array of scalar values (only if `is_leaf` is true). The size of `data` depends on the node's resolution level $L$. For a node at level $L$, the resolution is $N / 2^L$. The number of vertices is $(N/2^L + 1)^3$.

**Algorithm:**
1.  **Initial Coarsening:** Generate the base coarse levels by sampling. Level 0 is the finest ($N^3$). To generate Level $L+1$ from Level $L$, sample every other data point in each dimension. Level $k$ has resolution $N/2^k$. Level 7 is the root for a $128^3$ dataset [S1, unofficial].
2.  **Recursive Subdivision:** Start with a root node representing the entire volume at the coarsest level (Level 7).
3.  **Error Calculation:** For a current node at level $L$ with resolution $R = N/2^L$:
    *   Retrieve the scalar values of the 8 vertices of the cube represented by this node.
    *   Calculate the difference $\delta = \max(V_{vertices}) - \min(V_{vertices})$.
4.  **Pruning Decision:**
    *   If $\delta < \Delta$, the node is a leaf. Store the data for this resolution level.
    *   If $\delta \ge \Delta$, the node is an internal node. Create 8 children, each representing one octant of the current cube. Recursively apply steps 3-4 to each child.
5.  **Data Storage:** When a node becomes a leaf, store the scalar values for the grid points defined by its resolution level. Note that adjacent leaf nodes at different levels share boundary vertices to ensure continuity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_ar_data(volume, threshold, level=0, max_level=7):
    """
    volume: 3D array of scalars
    threshold: float, delta
    level: int, current resolution level
    max_level: int, root level
    """
    # Base case: reached root or max level
    if level == max_level:
        return create_leaf_node(volume, level)

    # Get current resolution size
    # Assuming volume is passed as the sub-volume for the current node
    # In practice, this requires tracking offsets or passing the full volume with bounds
    current_res = volume.shape[0]
    
    # Calculate max and min of the 8 corners of the current cube
    # The cube vertices are at indices 0 and current_res-1 in each dimension
    corners = [
        volume[0, 0, 0], volume[0, 0, current_res-1],
        volume[0, current_res-1, 0], volume[0, current_res-1, current_res-1],
        volume[current_res-1, 0, 0], volume[current_res-1, 0, current_res-1],
        volume[current_res-1, current_res-1, 0], volume[current_res-1, current_res-1, current_res-1]
    ]
    
    delta = max(corners) - min(corners)
    
    if delta < threshold:
        return create_leaf_node(volume, level)
    else:
        # Subdivide into 8 octants
        half = current_res // 2
        children = []
        
        # Define octant boundaries
        # Octant 0: x[0,half], y[0,half], z[0,half]
        # ... up to Octant 7: x[half,current_res], y[half,current_res], z[half,current_res]
        
        for x in [0, half]:
            for y in [0, half]:
                for z in [0, half]:
                    # Extract sub-volume for child
                    # Note: Indices must be handled carefully to share boundaries
                    child_vol = volume[x:x+half+1, y:y+half+1, z:z+half+1]
                    children.append(generate_ar_data(child_vol, threshold, level + 1, max_level))
        
        return create_internal_node(children)

def create_leaf_node(data, level):
    return Node(is_leaf=True, data=data, level=level)

def create_internal_node(children):
    return Node(is_leaf=False, children=children, level=None)
```

**Parameters:**
*   $\Delta$: Threshold for variance. Typical values depend on the data range. Not given in the sources.
*   $N$: Base resolution. Example: $128^3$ [S1, unofficial].

**Edge Cases:**
*   **Boundary Sharing:** Adjacent nodes at different levels share vertices. Implementation must ensure consistent data retrieval for shared boundaries [S1, unofficial].
*   **Degenerate Input:** If all vertex values are identical, $\delta=0$, and the node is pruned immediately.

## Subtopics

### [Hierarchical Voronoi Diagrams](4112-hierarchical-voronoi-diagrams.md)
An adaptive clustering method for scattered data that builds a hierarchy of Voronoi diagrams. It differs by not requiring a grid structure, making it suitable for unstructured datasets.

## Operating characteristics
*   **Time Complexity:** Generation is $O(N^3)$ in the worst case (no pruning), but typically faster due to pruning. Rendering complexity depends on the number of leaf nodes.
*   **Memory Complexity:** Proportional to the number of leaf nodes and their resolution. Significantly less than full resolution for smooth data [S1, unofficial].
*   **Strengths:** Reduces memory and computation time for redundant data areas. Preserves detail where needed [S1, unofficial].
*   **Limitations:** Requires preprocessing to build the octree. Not given in the sources for specific performance metrics.

## Background
Originated in the context of scientific volume visualization, specifically for IsoSurface Rendering of Adaptive Resolution Data [S1, unofficial]. The method addresses the need for efficient rendering of large volume datasets by exploiting data redundancy.

## Sources
[S1] IsoSurface Rendering of Adaptive Resolution Data, https://www.cs.unh.edu/~rdb/papers/ARiso.pdf, fetched 2026-10-29 (unofficial)
