# 2^n-Tree Hierarchical Representation

Also called: 2^n-tree splatting, Generalized object space splatting, Hierarchical Multiresolution Splatting, Multiresolution Splatting

## Overview
The 2^n-tree hierarchical representation is a data structure and rendering algorithm designed for the efficient visualization of high-dimensional (n-dimensional) datasets, such as hyper-volumes [S1]. It generalizes the octree-based acceleration of 3D splatting to arbitrary dimensions by organizing data into a hierarchy where groups of $2^n$ adjacent voxels are merged [S1]. It sets itself apart from its siblings by explicitly addressing the storage and computational scaling challenges of $n > 3$ dimensions, leveraging the fact that the relative storage overhead of the full hierarchy decreases exponentially as the dimension $n$ increases [S1].

## How it works
**Inputs:** An n-dimensional scalar field sampled on a regular grid with $k$ samples per dimension, resulting in $M = k^n$ total samples [S1]. A user-specified error bound $\epsilon$ [S1].
**Outputs:** A set of splat primitives rendered to the image, selected based on the error bound [S1].

**Data Structures:**
The hierarchy is stored as a complete array representing the full $2^n$-tree [S1].
*   **Node:** Represents a group of $2^n$ adjacent cells from the previous level.
*   **Attributes:** Each node stores the averaged function value of its children and a precomputed error bound [S1].
*   **Layout:** The hierarchy is built bottom-up. Level 0 contains the original $k^n$ voxels. Level $i$ contains nodes formed by merging $2^n$ nodes from Level $i-1$. The number of nodes at level $i$ is $\frac{k^n}{(2^n)^i}$ [S1].

**Algorithm Steps:**

1.  **Build Hierarchy:**
    *   Start with the leaf level (Level 0) containing the raw grid data.
    *   For each level $i$ from 1 to $h$ (where $k=2^h$):
        *   Group the nodes from Level $i-1$ into sets of $2^n$ adjacent nodes.
        *   Compute the parent node's value as the average of the $2^n$ children's values.
        *   Compute the error bound for the parent node based on the variance or difference between children and the average [S1].
        *   Store the parent node in the array structure for Level $i$.

2.  **Display Stage (Traversal):**
    *   Perform a Depth First Traversal (DFT) starting from the coarsest level (Level $h$, the root) [S1].
    *   For each node visited:
        *   Check if the node's error bound is less than or equal to the user-specified error bound $\epsilon$ [S1].
        *   If $E_{node} \le \epsilon$:
            *   Select this node as a leaf.
            *   Render the node as a single generalized splat primitive using texture-mapped graphics hardware acceleration [S1].
            *   Do not recurse to its children.
        *   If $E_{node} > \epsilon$:
            *   Recursively visit the $2^n$ children of this node.

**Formulas:**
*   **Total Samples:** For a grid with $k$ samples per direction in $n$ dimensions, total samples $M = k^n$ [S1].
*   **Hierarchy Storage:** Let $m = k = 2^h$. The total storage $M^*$ for the complete $2^n$-tree is:
    $$ M^* = \sum_{i=0}^h \frac{(2^h)^n}{(2^n)^i} = \sum_{i=0}^h (2^n)^{h-i} = \sum_{j=0}^h (2^n)^j $$
    [S1]
*   **Relative Storage Overhead:**
    $$ \frac{M^* - M}{M} < \frac{1}{(2^n - 1)} $$
    [S1]

**Parameters:**
*   **Dimension $n$:** The dimension of the embedding space. Typical values for visualization are $n \ge 4$ [S1].
*   **Error Bound $\epsilon$:** A threshold controlling the trade-off between accuracy and speed. Set by the user via a UI slider [S1].

**Edge Cases and Pitfalls:**
*   **Grid Size:** The derivation assumes $m = 2^h$ for some integer $h$. If the grid dimensions are not powers of 2, padding or truncation may be required [S1].
*   **High Dimensionality:** While storage overhead decreases with $n$, the number of children $2^n$ grows exponentially. For very large $n$, the branching factor becomes large, potentially impacting traversal efficiency despite the storage benefits [S1].
*   **Error Metric:** The specific formula for the error bound is not detailed in the sources, only that it is precomputed and used for comparison [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_hierarchy(grid, n):
    """
    Builds the 2^n-tree hierarchy from an n-dimensional grid.
    grid: n-dimensional array of shape (k, k, ..., k)
    n: dimension of the space
    """
    k = grid.shape[0]
    h = int(log2(k))
    
    # Initialize levels
    # levels[i] will store the nodes at depth i
    # Level 0 is the original grid
    levels = [grid]
    
    current_data = grid
    
    for i in range(1, h + 1):
        # Determine shape of current level data
        # Each node covers 2^n cells from previous level
        # New shape is k / 2^(i-1) in each dimension? 
        # Actually, grouping 2^n adjacent cells reduces count by 2^n total.
        # If we group in a hyper-cube manner, side length reduces by 2^(i-1)?
        # The source says: "number of cells is reduced at each level by a factor of 2^n"
        # So if level i-1 has N cells, level i has N / 2^n cells.
        
        # Let's assume a regular grid structure where we merge 2^n neighbors.
        # For simplicity, assume k is a power of 2 and we merge 2x2x...x2 blocks.
        
        new_shape = tuple(s // 2 for s in current_data.shape)
        new_data = np.zeros(new_shape)
        
        # Compute averages and errors
        # This requires iterating over the grid and grouping 2^n neighbors
        # For n dimensions, we iterate over the new shape and aggregate 2^n values
        
        # Placeholder for aggregation logic as specific implementation details 
        # of the error metric and averaging kernel are not fully specified in sources.
        # [G] Standard averaging: sum of 2^n children / 2^n.
        
        for indices in np.ndindex(new_shape):
            # Generate indices of the 2^n children in current_data
            children_indices = generate_children_indices(indices, current_data.shape)
            children_values = current_data[children_indices]
            
            avg_value = np.mean(children_values)
            error = compute_error(children_values, avg_value) # [G] Placeholder for error metric
            
            new_data[indices] = (avg_value, error)
            
        levels.append(new_data)
        current_data = new_data
        
    return levels

def render(levels, n, epsilon):
    """
    Renders the 2^n-tree using depth-first traversal.
    levels: list of arrays, levels[i] is the data at depth i
    n: dimension
    epsilon: user-specified error bound
    """
    h = len(levels) - 1
    
    # Start from the root (coarsest level)
    render_node(levels, h, (0,) * n, epsilon)

def render_node(levels, level_idx, coords, epsilon):
    """
    Recursively render a node.
    """
    if level_idx == 0:
        # Leaf node (original voxel)
        value, error = levels[0][coords]
        # Render splat for this voxel
        draw_splat(coords, value)
        return

    value, error = levels[level_idx][coords]
    
    if error <= epsilon:
        # Select this node as a leaf
        draw_splat(coords, value)
    else:
        # Recurse to children
        # Children are at level_idx - 1
        # There are 2^n children
        children_coords = get_children_coords(coords, levels[level_idx-1].shape)
        for child_coord in children_coords:
            render_node(levels, level_idx - 1, child_coord, epsilon)

def generate_children_indices(parent_coords, current_shape):
    """
    Returns the indices of the 2^n children in the current level data
    that correspond to the parent node at parent_coords in the next level.
    """
    # This depends on the specific grid layout. 
    # Assuming standard hyper-cube merging:
    # If parent is at (x, y, ...) in level i, 
    # children are at (2x, 2y, ...), (2x+1, 2y, ...), etc. in level i-1.
    indices = []
    for i in range(2**n):
        # Convert integer i to n-dimensional binary index
        child_idx = integer_to_nd_index(i, n)
        # Scale up by 2
        scaled_idx = tuple(c * 2 + d for c, d in zip(parent_coords, child_idx))
        indices.append(scaled_idx)
    return indices

def integer_to_nd_index(i, n):
    """
    Converts an integer i to an n-dimensional index tuple.
    """
    idx = []
    for _ in range(n):
        idx.append(i % 2)
        i //= 2
    return tuple(reversed(idx))

def get_children_coords(parent_coords, child_level_shape):
    """
    Helper for render_node to get child coordinates in the child level array.
    """
    # Similar logic to generate_children_indices but for traversal
    children = []
    for i in range(2**n):
        child_idx = integer_to_nd_index(i, n)
        scaled_idx = tuple(c * 2 + d for c, d in zip(parent_coords, child_idx))
        children.append(scaled_idx)
    return children
```

## Subtopics

### Octree-based Splatting
The 3D special case ($n=3$) of this method, using an octree structure. It is chosen when the data is strictly 3D and standard octree implementations are preferred. [Link](8067-octree-based-splatting.md)

### Splatting-Based AMR Volume Rendering
Uses Adaptive Mesh Refinement (AMR) grids rather than a fixed $2^n$-tree. It is chosen when the data has irregular refinement patterns that do not align with a regular hyper-cube hierarchy. [Link](10507-splatting-based-amr-volume-rendering.md)

### Octree Splatting
A general term for octree-based splatting, often implying specific optimizations for 3D. [Link](13649-octree-splatting.md)

### TopoGS
A method for Gaussian Splatting that uses topological structures. It is chosen for 3D Gaussian Splatting applications where topological consistency is important. [Link](17246-topogs.md)

### LinearMask-GS
A method for Gaussian Splatting using linear masks. It is chosen for efficient 3D Gaussian Splatting rendering. [Link](17755-linearmask-gs.md)

## Operating characteristics
*   **Time Complexity:** The build phase is linear with respect to the number of voxels, $O(M)$, where $M=k^n$, as each voxel is processed a constant number of times during the bottom-up merge [S1]. The rendering phase depends on the error bound $\epsilon$; in the worst case (all leaves selected), it is $O(M)$, but typically much faster due to culling [S1].
*   **Memory Complexity:** The storage overhead is bounded by $\frac{1}{2^n - 1}$ relative to the raw data size [S1]. This overhead decreases exponentially as $n$ increases, making the method highly memory-efficient for high-dimensional data compared to storing the full grid or other hierarchical structures [S1].
*   **Measured Performance:** For a 6-dimensional grid with 64 samples per direction ($64^6 \approx 68$ billion samples), the method enables efficient visualization that would otherwise be infeasible [S1].
*   **Strengths:**
    *   Scalability to high dimensions ($n > 3$) [S1].
    *   Low storage overhead for high $n$ [S1].
    *   User-controlled trade-off between accuracy and speed via $\epsilon$ [S1].
*   **Limitations:**
    *   Requires regular grid data [S1].
    *   Large branching factor ($2^n$) for high $n$ may complicate traversal implementation [S1].
    *   Not suitable for irregular or unstructured data without preprocessing [S1].
*   **When to Use:** When visualizing high-dimensional datasets (e.g., 4D, 6D) on regular grids where memory efficiency and interactive performance are critical [S1].
*   **When Not to Use:** For irregular data, or when $n$ is very small (e.g., $n=2$) where the overhead of the hierarchy might not be justified compared to simpler methods [S1].

## Background
The method was introduced by Laur and Hanrahan, who accelerated 3D splatting by adopting an octree hierarchical representation [S1]. This work generalized that approach to arbitrary $n$ dimensions, showing how the relative storage overhead of the full $2^n$-tree hierarchy decreases as the dimension of the embedding space increases [S1]. It is used in hyper-volume visualization systems to handle large n-dimensional datasets, such as those arising in scientific computing and configuration space analysis [S1].

## Sources
[S1] Hypervolume visualization: a challenge in simplicity, https://doi.org/10.1145/288126.288173, fetched 2026-10-26
