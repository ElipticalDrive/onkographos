# Homogeneity-Based KD-Tree Construction

Also called: AMR Mesh Construction via Homogeneity-Based KD-Tree

## Overview
This method constructs a hierarchical KD-tree data structure for Adaptive Mesh Refinement (AMR) datasets to enable efficient ray traversal and rendering without replicating cells into a voxel grid. It solves the problem of memory waste and performance penalties associated with homogeneous page tables or flat hierarchical structures by grouping same-level cells into axis-aligned patches. It sets itself apart from siblings like [Adaptive Octree Construction with 2:1 Balance and Surface Propagation](830-adaptive-octree-construction-with-2-1-balance-and-surface-pr.md) by explicitly leveraging cell level homogeneity to define leaf nodes, and from [Adaptive Mesh Refinement (AMR) Data Structures](5344-adaptive-mesh-refinement-amr-data-structures.md) by providing a specific construction algorithm for CPU-GPU hybrid traversal.

## How it works
**Inputs:** An AMR dataset consisting of cells with associated refinement levels and spatial positions.
**Outputs:** A KD-tree where leaf nodes contain homogeneous patches (axis-aligned boxes of same-level cells) and internal nodes contain split information.

**Data Structures:**
*   **Box:** An axis-aligned bounding box defined by minimum and maximum coordinates.
*   **Node:** Represents a region in the KD-tree.
    *   `is_leaf`: Boolean flag.
    *   `data`: If leaf, contains the list of cells in the patch.
    *   `children`: If internal, a tuple of two child nodes.
    *   `split_axis`: (Implicit in traversal) The axis along which the box was split.

**Parameters:**
*   `homogeneity(box)`: A function returning the ratio of homogeneity of cell levels inside the box. Returns $1.0$ if all cells share the same level, $0.0$ if all have different levels. [S1]

**Algorithm:**
The construction is recursive. The root box contains the entire dataset.

1.  **Check Homogeneity:** Calculate $H = \text{homogeneity}(b)$ for the current box $b$.
2.  **Base Case (Leaf):** If $H == 1.0$, the box is homogeneous.
    *   Mark node $n$ as a leaf.
    *   Copy all cells from box $b$ to node $n$.
    *   Return.
3.  **Recursive Step (Split):** If $H < 1.0$:
    *   Initialize `best_score` to $0$.
    *   Initialize `best_cut` to `None`.
    *   For each axis direction $d \in \{x, y, z\}$:
        *   For each possible cut plane $p$ in direction $d$ (typically at cell boundaries):
            *   Split box $b$ by $p$ into $b_{left}$ and $b_{right}$.
            *   Calculate `current_score` = $\text{homogeneity}(b_{left}) + \text{homogeneity}(b_{right})$? *Note: The source text says "homogeneity(b cut by p)" which implies the score of the resulting configuration. Given the goal is to maximize homogeneity, and the base case is 1, the score likely represents the sum or average homogeneity of the children, or simply the homogeneity of the resulting split state. The pseudocode below follows the literal "current score = homogeneity(b cut by p)" but interprets it as the metric to maximize. In practice, this usually means maximizing the homogeneity of the resulting children.* [S1]
            *   If `current_score` > `best_score`:
                *   `best_score` = `current_score`
                *   `best_cut` = $p$
    *   Split box $b$ using `best_cut` to create `child_box_1` and `child_box_2`.
    *   Create two child nodes `child_node_1` and `child_node_2`.
    *   Set $n.children = (\text{child\_node\_1}, \text{child\_node\_2})$.
    *   Recursively call `build_mesh(child_box_1, child_node_1)`.
    *   Recursively call `build_mesh(child_box_2, child_node_2)`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def homogeneity(box):
    """
    Returns the ratio of homogeneity of cell levels inside the box.
    Returns 1.0 if all cells share the same level.
    Returns 0.0 if all cells have different levels.
    """
    levels = [cell.level for cell in box.cells]
    if not levels:
        return 1.0 # Empty box is trivially homogeneous
    unique_levels = set(levels)
    if len(unique_levels) == 1:
        return 1.0
    # If not all same, calculate ratio. 
    # Source defines 0 if all different, 1 if all same. 
    # Intermediate values are "ratio of homogeneity". 
    # A common interpretation for ratio is 1 / len(unique_levels) or similar.
    # However, for the split decision, we just need a scalar metric.
    # We will return 1.0 if homogeneous, 0.0 otherwise for simplicity 
    # as the exact intermediate formula is not specified, 
    # but the comparison logic remains valid.
    return 0.0 

def split_box(box):
    best_score = 0.0
    best_cut = None
    
    # Iterate over axes
    for axis in ['x', 'y', 'z']:
        # Iterate over possible cut planes
        # In a discrete grid, cuts are at cell boundaries
        cuts = get_possible_cuts(box, axis)
        
        for p in cuts:
            b_left, b_right = box.split(p, axis)
            
            # Calculate score of the split
            # The source says "homogeneity(b cut by p)". 
            # This likely implies the combined homogeneity of the children.
            score = homogeneity(b_left) + homogeneity(b_right)
            
            if score > best_score:
                best_score = score
                best_cut = (p, axis)
                
    if best_cut is None:
        return box, box # Fallback, though unlikely if cuts exist
        
    p, axis = best_cut
    b_left, b_right = box.split(p, axis)
    return b_left, b_right

def build_mesh(box, node):
    if homogeneity(box) == 1.0:
        node.is_leaf = True
        node.data = box.cells
        return
    
    node.is_leaf = False
    child_node_1 = Node()
    child_node_2 = Node()
    
    b_left, b_right = split_box(box)
    
    node.children = (child_node_1, child_node_2)
    
    build_mesh(b_left, child_node_1)
    build_mesh(b_right, child_node_2)
```

**Edge Cases and Pitfalls:**
*   **Empty Boxes:** If a box contains no cells, `homogeneity` should return 1.0 to prevent infinite recursion or unnecessary splits. [G]
*   **Cut Plane Selection:** The number of possible cut planes can be large. Optimization is required to avoid $O(N^2)$ complexity in splitting. [G]
*   **GPU Traversal:** The resulting tree is traversed on the CPU, but patches are sent to the GPU. The CPU traversal must be back-to-front to support correct alpha blending. [S1]

## Subtopics

### [AMR KD-Tree Construction](6768-amr-kd-tree-construction.md)
A general method for representing and traversing block-structured (Berger-Colella) Adaptive Mesh Refinement (AMR) data using a kd-tree structure to enable efficient query, sampling, and direct rendering. It sets itself apart by grouping same-level cells into patches and building a KD-tree based on level homogeneity, avoiding the memory waste and performance penalty associated with flat hierarchical structures.

## Operating characteristics
*   **Time Complexity:** The construction time depends on the number of cells and the cost of evaluating homogeneity for each potential split. In the worst case, it is $O(N \log N)$ or higher depending on the cut plane search strategy. [G]
*   **Memory Complexity:** The structure stores cells only once in leaf nodes, avoiding the memory waste of voxel replication. The overhead is the tree structure itself. [S1]
*   **Performance:** Rendering is accelerated by sending homogeneous patches to the GPU, where traversal is trivial. CPU traversal handles the hierarchy. This hybrid approach balances CPU flexibility and GPU parallelism. [S1]
*   **Strengths:** No additional cells are added; memory efficient; leverages GPU regularity. [S1]
*   **Limitations:** CPU-GPU synchronization overhead; complex traversal logic on CPU. [S1]

## Background
This method was presented in the context of high-quality, semi-analytical volume rendering for AMR datasets. It addresses the limitations of previous multi-octree approaches that required data reformatting or voxel replication. The technique was developed to handle hierarchical structures efficiently on the GPU by using a hybrid CPU-GPU sorting approach. [S1]

## Sources
[S1] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
