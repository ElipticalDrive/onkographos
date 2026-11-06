# Quadtree Raycasting

Also called: raycast2

## Overview
Quadtree Raycasting is a recursive algorithm that traverses a 2D quadtree data structure to find the first solid node intersected by a ray. It solves the problem of efficiently locating geometry in sparse or hierarchical 2D volumes by skipping empty subtrees. It differs from standard grid-based raycasting (like [Ray-box intersection](776-ray-box-intersection.md)) by adapting the traversal step size and order to the hierarchical node boundaries rather than a uniform grid, and it differs from [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md) by operating on axis-aligned square quadrants.

## How it works
**Inputs and Outputs**
*   **Input:** Ray origin $\mathbf{s}$ (Vec2), ray direction $\mathbf{d}$ (Vec2), and a root Quadtree node.
*   **Output:** The first solid leaf node intersected by the ray, or `None` if no intersection is found within the tree bounds.

**Data Structures**
The quadtree node layout is defined as:
*   `is_full`: Boolean flag indicating if the node is entirely solid.
*   `children`: Optional array of 4 child nodes (`Option<[Box<QuadtreeNode>; 4]>`). `None` indicates a leaf.
*   `size`: Integer width/height of the node's spatial extent.
*   `position`: 2D integer coordinates (`IVec2`) of the node's bottom-left corner.

**Algorithm Steps**
1.  **Initialization:** Calculate the initial intersection interval $[t_{min}, t_{max}]$ for the root node.
    *   Compute boundary distances:
        $$tx_0 = \frac{pos.x - s.x}{|d.x|}, \quad ty_0 = \frac{pos.y - s.y}{|d.y|}$$
        $$tx_1 = \frac{pos.x + size - s.x}{|d.x|}, \quad ty_1 = \frac{pos.y + size - s.y}{|d.y|}$$
    *   Determine interval: $t_{min} = \max(tx_0, ty_0)$, $t_{max} = \min(tx_1, ty_1)$.
    *   If $t_{min} \ge t_{max}$, return `None`.
2.  **Traversal Setup:** Initialize a bitmask `mask = 0`.
    *   If $d.x < 0$, flip the start x-coordinate logic and set bit 0 of `mask`.
    *   If $d.y < 0$, flip the start y-coordinate logic and set bit 1 of `mask`.
    *   Call the recursive helper `proc_subtree` with the root node and initial bounds $(tx_0, ty_0, tx_1, ty_1)$.
3.  **Recursive Traversal (`proc_subtree`):**
    *   **Pruning:** If $tx_1 < 0$ or $ty_1 < 0$, the ray has passed the current node's back face; return `None`.
    *   **Leaf Check:** If the node is a leaf:
        *   If `is_full` is true, return the node and hit distance $t = \max(tx_0, ty_0)$ (assuming $tx_0 < 0$ and $ty_0 < 0$ implies entry before current step, otherwise use appropriate entry point).
        *   If not full, return `None`.
    *   **Internal Node:**
        *   Calculate midpoints: $tx_m = (tx_0 + tx_1) / 2$, $ty_m = (ty_0 + ty_1) / 2$.
        *   Determine the first child index to visit based on ray direction relative to the sub-regions.
        *   Iterate through children $i = 0$ to $3$:
            *   Apply mask to index: $child\_idx = i \oplus mask$.
            *   Recurse into the corresponding child with updated bounds:
                *   Child 0 (Bottom-Left): $(tx_0, ty_0, tx_m, ty_m)$
                *   Child 1 (Bottom-Right): $(tx_m, ty_0, tx_1, ty_m)$
                *   Child 2 (Top-Left): $(tx_0, ty_m, tx_m, ty_1)$
                *   Child 3 (Top-Right): $(tx_m, ty_m, tx_1, ty_1)$
            *   If recursion returns a hit, propagate it up.
            *   If no hit, continue to next child.
        *   If loop completes without hit, return `None`.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quadtree_raycast(start: Vec2, dir: Vec2, root: QuadtreeNode) -> Optional[Hit]:
    # Calculate initial intersection interval for root
    # Note: pos and size are attributes of root
    tx0 = (root.position.x - start.x) / abs(dir.x)
    ty0 = (root.position.y - start.y) / abs(dir.y)
    tx1 = (root.position.x + root.size - start.x) / abs(dir.x)
    ty1 = (root.position.y + root.size - start.y) / abs(dir.y)
    
    t_min = max(tx0, ty0)
    t_max = min(tx1, ty1)
    
    if t_min >= t_max:
        return None

    # Initialize mask for direction flipping
    mask = 0
    if dir.x < 0:
        # Flip start.x logic: effectively swapping tx0/tx1 roles for traversal order
        # In the source, this modifies the start variable passed to proc_subtree
        # Here we pass tx0, tx1 directly but use mask in child selection
        mask |= 1
    if dir.y < 0:
        mask |= 2
        
    return proc_subtree(root, tx0, ty0, tx1, ty1, mask)

def proc_subtree(node: QuadtreeNode, tx0: float, ty0: float, tx1: float, ty1: float, mask: int) -> Optional[Hit]:
    # Pruning: if the back face is behind the ray origin (in t-space)
    if tx1 < 0 or ty1 < 0:
        return None

    if node.is_leaf():
        if node.is_full:
            # Hit found. Distance is the entry point into this solid block.
            # If tx0 < 0 and ty0 < 0, the ray entered before this node's local t=0,
            # so we return the max of the entry bounds.
            t_hit = max(tx0, ty0)
            return Hit(node, t_hit)
        else:
            return None

    # Internal node: calculate midpoints in t-space
    txm = (tx0 + tx1) / 2.0
    tym = (ty0 + ty1) / 2.0

    # Determine traversal order. 
    # The source uses a loop 0..3 and applies mask to index.
    # We must check children in an order that respects ray direction.
    # The mask XOR operation maps logical child index to spatial quadrant.
    
    # Child 0: Bottom-Left
    child_idx = 0 ^ mask
    hit = proc_subtree(node.children[child_idx], tx0, ty0, txm, tym, mask)
    if hit: return hit

    # Child 1: Bottom-Right
    child_idx = 1 ^ mask
    hit = proc_subtree(node.children[child_idx], txm, ty0, tx1, tym, mask)
    if hit: return hit

    # Child 2: Top-Left
    child_idx = 2 ^ mask
    hit = proc_subtree(node.children[child_idx], tx0, tym, txm, ty1, mask)
    if hit: return hit

    # Child 3: Top-Right
    child_idx = 3 ^ mask
    hit = proc_subtree(node.children[child_idx], txm, tym, tx1, ty1, mask)
    if hit: return hit

    return None
```

**Parameters**
*   `start`: Ray origin in world space (Vec2).
*   `dir`: Ray direction vector (Vec2).
*   `mask`: Bitmask (u8) to invert child indices. Bit 0 set if $d.x < 0$, Bit 1 set if $d.y < 0$.

**Edge Cases and Pitfalls**
*   **Direction Flipping:** The algorithm relies on the `mask` to ensure consistent traversal order regardless of ray direction. Failure to correctly set the mask or apply it via XOR will result in visiting empty children before solid ones, or missing hits entirely [S1, unofficial].
*   **Precision:** Division by zero occurs if $d.x$ or $d.y$ is exactly 0. The source assumes non-zero components or handles them via the `signum` logic in the DDA variant, but the recursive `proc_subtree` method requires absolute values in denominators.
*   **Degenerate Input:** If $t_{min} \ge t_{max}$, the ray misses the bounding box. This check is performed at the root and implicitly in children via the $tx_1 < 0$ pruning.

## Operating characteristics
*   **Time Complexity:** $O(\log N)$ on average for sparse data, where $N$ is the number of leaf nodes, as it skips empty subtrees. In the worst case (fully dense or ray traverses many nodes), it approaches $O(N)$.
*   **Memory Complexity:** $O(N)$ to store the quadtree structure.
*   **Strengths:** Efficient for sparse 2D volumes; adaptive resolution allows high detail in dense regions and low detail in empty regions.
*   **Limitations:** Recursive nature may cause stack overflow for very deep trees; not directly parallelizable per-ray without careful synchronization; 2D only.
*   **When to use:** Sparse 2D volume rendering, collision detection in hierarchical 2D maps.
*   **When not to use:** Dense 3D volumes (use [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md) or voxel grids), uniform grids (use [Ray-box intersection](776-ray-box-intersection.md)).

## Background
The method originates from implementations of 2D raycasting against hierarchical spatial structures, specifically documented in the `quadtree-raycast` repository by Musta-Kissa [S1, unofficial]. It is often referred to as `proc_subtree` or `raycast2` in the context of these specific implementations. It builds on the principles of ray-primitive intersection, adapting the traversal logic to the quadtree's recursive structure.

## Sources
[S1] Musta-Kissa/quadtree-raycast, https://github.com/Musta-Kissa/quadtree-raycast, fetched 2026-10-24 (unofficial)
