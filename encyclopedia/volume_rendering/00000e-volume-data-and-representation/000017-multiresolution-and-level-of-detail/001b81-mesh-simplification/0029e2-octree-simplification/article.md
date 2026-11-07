# Octree Simplification
Also called: SimplifyOctree
## Overview
Octree Simplification is a post-processing step for Dual Contouring that reduces the complexity of the generated octree representation by collapsing leaf nodes into pseudo-leaf nodes. It solves the problem of excessive memory usage and rendering overhead in highly detailed isosurface meshes by removing geometric detail that falls below a specified error threshold. It sets itself apart from sibling mesh simplification methods by operating directly on the hierarchical octree structure and leveraging the Quadratic Error Function (QEF) residuals inherent to the Dual Contouring pipeline, rather than modifying the final triangle mesh topology.
## How it works
**Inputs and Outputs:**
- Input: An octree structure where leaf nodes contain `OctreeDrawInfo` instances holding a QEF and corner bitmasks. A scalar `threshold` value.
- Output: A modified octree where some internal nodes have been converted to "Pseudo Leaf" nodes (leaves with size > 1) or pruned entirely, reducing the total node count.

**Data Structures:**
- `OctreeNode`: Contains a `type` (Leaf, Internal, Pseudo_Leaf) and up to 8 `children`.
- `OctreeDrawInfo`: Contains `qef` (a symmetric 4x4 matrix or equivalent accumulator) and `corners` (an integer bitmask representing the 8 corners of the voxel).
- `QEF`: An accumulator for quadratic error terms. Supports `add(QEF)` and `solve() -> (position, error)`.

**Algorithm Steps:**
The method is implemented via a recursive function `SimplifyOctree(node, threshold)`:

1. **Recursion:** Call `SimplifyOctree` on all 8 children of the current `node`.
2. **Initialization:** Initialize `isCollapsible = true`, `qef` (empty), `signs[8] = {-1, -1, -1, -1, -1, -1, -1, -1}`, `midsign = -1`, `edgeCount = 0`.
3. **Child Processing:** Iterate `i` from 0 to 7:
   - If `node->children[i]` exists:
     - If `child->type == Node_Internal`: Set `isCollapsible = false`. [S1, unofficial]
     - Else (child is Leaf):
       - Accumulate QEF: `qef.add(child->drawInfo->qef)`. [S1, unofficial]
       - Extract corner bits: `midsign = (child->drawInfo->corners >> (7 - i)) & 1`; `signs[i] = (child->drawInfo->corners >> i) & 1`. [S1, unofficial]
       - Increment `edgeCount`. [S1, unofficial]
4. **Collapse Check:** If `!isCollapsible`, return `node` unchanged. This prevents collapsing if any child is an internal node. [S1, unofficial]
5. **Leaf Aggregation:** If all children are Leaf nodes:
   - Solve the accumulated `qef` to obtain a new vertex position and an `error` value. [S1, unofficial]
   - Compare `error` against `threshold`.
   - If `error <= threshold`: Create a new Pseudo Leaf Node. This node represents the collapsed children. Its `drawInfo` contains the new QEF and position. The original children are discarded. [S1, unofficial]
   - If `error > threshold`: Discard the new node (return `null` or keep children, depending on implementation; the source implies discarding the *proposed* new node if it doesn't meet criteria, effectively keeping the structure or pruning if the error is too high to justify a single vertex). [S1, unofficial]
6. **Return:** Return the resulting node (either the original internal node, the new pseudo-leaf, or null if pruned).

**Parameters:**
- `threshold`: A scalar value controlling the aggressiveness of simplification. Lower values result in more aggressive collapsing. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Internal Child Constraint:** An internal node cannot be collapsed if at least one of its children is also an internal node. This preserves the hierarchical integrity required for the contouring process. [S1, unofficial]
- **Precision:** The QEF solution must be robust. If the QEF is singular or ill-conditioned, the error calculation may be unreliable. [G]
- **Boundary Conditions:** The algorithm assumes a valid octree structure. Nodes with fewer than 8 children (sparse octrees) must handle missing children gracefully (ignoring them in the loop).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simplify_octree(node, threshold):
    # Step 1: Recursively simplify children
    for i in range(8):
        if node.children[i] is not None:
            node.children[i] = simplify_octree(node.children[i], threshold)

    # Step 2: Initialize variables
    is_collapsible = True
    qef = QEF()
    signs = [-1] * 8
    midsign = -1
    edge_count = 0

    # Step 3: Process children
    for i in range(8):
        child = node.children[i]
        if child is not None:
            if child.type == Node_Internal:
                is_collapsible = False
            else:
                # Accumulate QEF
                qef.add(child.draw_info.qef)
                
                # Extract corner bits
                # Note: Source uses specific bit shifts; ensure consistency with draw_info.corners layout
                midsign = (child.draw_info.corners >> (7 - i)) & 1
                signs[i] = (child.draw_info.corners >> i) & 1
                
                edge_count += 1

    # Step 4: Check collapsibility
    if not is_collapsible:
        return node

    # Step 5: Check if all children are leaves
    # In a sparse octree, we must ensure all existing children are leaves
    # and that we have processed all 8 slots (or handle sparse logic)
    # Assuming dense or fully processed sparse structure where 'child is not None'
    # implies a leaf if is_collapsible is true.
    
    if edge_count > 0:
        # Solve QEF
        position, error = qef.solve()
        
        # Step 6: Decide based on threshold
        if error <= threshold:
            # Create Pseudo Leaf Node
            new_node = OctreeNode()
            new_node.type = Node_Pseudo_Leaf
            new_node.size = node.size * 2 # Or appropriate scaling factor
            new_node.draw_info = OctreeDrawInfo()
            new_node.draw_info.qef = qef
            new_node.draw_info.position = position
            # Reconstruct corners if needed for downstream contouring
            # Source implies using the new QEF for position, but corner bits
            # might need to be aggregated or set to a default for the pseudo-leaf
            new_node.draw_info.corners = 0 # Placeholder; exact reconstruction not detailed in source
            return new_node
        else:
            # Error too high, do not collapse
            # Return original node with its simplified children
            return node
    else:
        # No children or all children were internal (handled by is_collapsible)
        return node
```
## Subtopics
## Operating characteristics
- **Time Complexity:** Linear with respect to the number of nodes in the octree, $O(N)$, as each node is visited once.
- **Memory Complexity:** Reduces memory usage by replacing multiple leaf nodes with a single pseudo-leaf node. The reduction factor depends on the `threshold` and the smoothness of the isosurface.
- **Strengths:** Integrates seamlessly with Dual Contouring; does not require mesh regeneration; preserves topological correctness by respecting the octree hierarchy.
- **Limitations:** Cannot collapse internal nodes; aggressive simplification may introduce visible artifacts if the threshold is too low.
- **When to use:** When the initial octree is too large for real-time rendering or storage, and the isosurface is relatively smooth.
- **When not to use:** When high-frequency geometric details must be preserved, or when the octree contains complex internal structures that cannot be represented by a single vertex.
## Background
The method originates from the implementation of Dual Contouring on distance fields, specifically described in a blog post by N. Gildea in 2014. It leverages the QEF framework inherent to Dual Contouring to guide the simplification process, ensuring that the resulting pseudo-leaf nodes provide the best possible fit to the original data within the error bounds.
## Sources
[S1] Implementing Dual Contouring, http://ngildea.blogspot.com.br/2014/11/implementing-dual-contouring.html, fetched 2026-10-30 (unofficial)
