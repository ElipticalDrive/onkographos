# Quadtree Raycasting
Also called: raycast2

## Overview
Quadtree Raycasting is a recursive ray-quadtree intersection algorithm that traverses a 2D quadtree to find the first hit node [S1, unofficial]. It solves the problem of efficiently locating the first occupied voxel in a hierarchical 2D grid by handling ray direction flipping via a mask to ensure consistent traversal order regardless of ray direction [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `start` (Vec2, ray origin in world space), `dir` (Vec2, ray direction vector), `node` (QuadtreeNode, the root or current subtree to traverse), `tx0`, `ty0`, `tx1`, `ty1` (f32, boundary distances in ray parameter space) [S1, unofficial].
*   **Output:** `Optional[Tuple[QuadtreeNode, float]]` representing the first hit node and the hit distance $t$, or `None` if no intersection is found within the bounds [S1, unofficial].

**Data Structures:**
The `QuadtreeNode` structure [S1, unofficial]:
*   `is_full`: bool, indicates if the node represents a fully occupied volume [S1, unofficial].
*   `children`: `Optional[List[QuadtreeNode]]`, a fixed-size array of 4 child nodes (bottom-left, bottom-right, top-left, top-right), or `None` if leaf [S1, unofficial].
*   `size`: f32, the spatial extent (width/height) of the node in world units [S1, unofficial].
*   `position`: IVec2, the integer coordinates of the node's bottom-left corner [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization and Masking:**
    Initialize a bitmask `mask` to 0 [S1, unofficial].
    *   If `dir.x < 0`: Flip `start.x` using `start.x = 2.0 * node.position.x + node.size - start.x` and set `mask` bit 0 [S1, unofficial].
    *   If `dir.y < 0`: Flip `start.y` using `start.y = 2.0 * node.position.y + node.size - start.y` and set `mask` bit 1 [S1, unofficial].

2.  **Intersection Check:**
    Calculate intersection distances to the node boundaries [S1, unofficial]:
    $$tx0 = (node.position.x - start.x) / |dir.x|$$
    $$ty0 = (node.position.y - start.y) / |dir.y|$$
    $$tx1 = (node.position.x + node.size - start.x) / |dir.x|$$
    $$ty1 = (node.position.y + node.size - start.y) / |dir.y|$$
    [S1, unofficial]
    
    Determine intersection interval [S1, unofficial]:
    $$t_{min} = \max(tx0, ty0)$$
    $$t_{max} = \min(tx1, ty1)$$
    [S1, unofficial]
    
    If $t_{min} \ge t_{max}$, the ray does not intersect the node bounding box; return `None` [S1, unofficial].

3.  **Recursive Traversal (`proc_subtree`):**
    Call the recursive function with the current node and the calculated boundary distances $(tx0, ty0, tx1, ty1)$ [S1, unofficial].
    
    Inside `proc_subtree(node, tx0, ty0, tx1, ty1, mask)`:
    1.  **Early Exit:** If $tx1 < 0$ or $ty1 < 0$, the ray has passed the node's far boundary; return `None` [S1, unofficial].
    2.  **Leaf Check:** If `node.children` is `None`:
        *   If `node.is_full` is true, the ray hits this node [S1, unofficial].
        *   Calculate hit distance $t = \max(tx0, ty0)$ [S1, unofficial].
        *   Return `(node, t)` [S1, unofficial].
    3.  **Internal Node:** If the node has children:
        *   Calculate midpoints [S1, unofficial]:
            $$txm = (tx0 + tx1) / 2$$
            $$tym = (ty0 + ty1) / 2$$
            [S1, unofficial]
        *   Determine the first child index to visit using `first_node(tx0, ty0, txm, tym)` which accounts for the ray direction relative to the sub-regions [S1, unofficial].
        *   Iterate through children using a loop variable `curr_node` starting from `first_node(...)` [S1, unofficial].
        *   For each child, apply the mask to the index: `child_idx = curr_node ^ mask` [S1, unofficial]. This ensures the logical child index corresponds to the correct spatial quadrant regardless of ray direction [S1, unofficial].
        *   **Child 0 (Bottom-Left):** Recurse with bounds $(tx0, ty0, txm, tym)$ [S1, unofficial]. If a hit is returned, propagate it up. Else, continue [S1, unofficial].
        *   **Child 1 (Bottom-Right):** Recurse with bounds $(txm, ty0, tx1, tym)$ [S1, unofficial]. If a hit is returned, propagate it up. Else, continue [S1, unofficial].
        *   **Child 2 (Top-Left):** Recurse with bounds $(tx0, tym, txm, ty1)$ [S1, unofficial]. If a hit is returned, propagate it up. Else, continue [S1, unofficial].
        *   **Child 3 (Top-Right):** Recurse with bounds $(txm, tym, tx1, ty1)$ [S1, unofficial]. If a hit is returned, propagate it up. Else, terminate the loop [S1, unofficial].
        *   If no child returns a hit, return `None` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from typing import Optional, Tuple, List

class QuadtreeNode:
    def __init__(self, is_full: bool, children: Optional[List['QuadtreeNode']], size: float, position: Tuple[int, int]):
        self.is_full = is_full
        self.children = children
        self.size = size
        self.position = position

def quadtree_raycast(start: Tuple[float, float], dir: Tuple[float, float], node: QuadtreeNode) -> Optional[Tuple[QuadtreeNode, float]]:
    # Step 1: Initialization and Masking
    mask = 0
    current_start = list(start)
    
    if dir[0] < 0:
        current_start[0] = 2.0 * node.position[0] + node.size - start[0]
        mask |= 1
    if dir[1] < 0:
        current_start[1] = 2.0 * node.position[1] + node.size - start[1]
        mask |= 2
        
    # Step 2: Intersection Check
    abs_dx = abs(dir[0]) if dir[0] != 0 else 1e-9
    abs_dy = abs(dir[1]) if dir[1] != 0 else 1e-9
    
    tx0 = (node.position[0] - current_start[0]) / abs_dx
    ty0 = (node.position[1] - current_start[1]) / abs_dy
    tx1 = (node.position[0] + node.size - current_start[0]) / abs_dx
    ty1 = (node.position[1] + node.size - current_start[1]) / abs_dy
    
    t_min = max(tx0, ty0)
    t_max = min(tx1, ty1)
    
    if t_min >= t_max:
        return None
        
    # Step 3: Recursive Traversal
    return proc_subtree(node, tx0, ty0, tx1, ty1, mask)

def proc_subtree(node: QuadtreeNode, tx0: float, ty0: float, tx1: float, ty1: float, mask: int) -> Optional[Tuple[QuadtreeNode, float]]:
    # Early Exit
    if tx1 < 0 or ty1 < 0:
        return None
        
    # Leaf Check
    if node.children is None:
        if node.is_full:
            t = max(tx0, ty0)
            return (node, t)
        else:
            return None
            
    # Internal Node
    txm = (tx0 + tx1) / 2.0
    tym = (ty0 + ty1) / 2.0
    
    # Determine traversal order based on ray direction relative to sub-regions [G]
    first_idx = first_node(tx0, ty0, txm, tym)
    
    curr_node = first_idx
    while curr_node < 4:
        # Apply mask to invert child index if necessary [S1, unofficial]
        child_idx = curr_node ^ mask
        
        child = node.children[child_idx]
        
        # Define bounds for the specific child quadrant [S1, unofficial]
        if child_idx == 0: # Bottom-Left
            hit = proc_subtree(child, tx0, ty0, txm, tym, mask)
        elif child_idx == 1: # Bottom-Right
            hit = proc_subtree(child, txm, ty0, tx1, tym, mask)
        elif child_idx == 2: # Top-Left
            hit = proc_subtree(child, tx0, tym, txm, ty1, mask)
        elif child_idx == 3: # Top-Right
            hit = proc_subtree(child, txm, tym, tx1, ty1, mask)
        else:
            hit = None
            
        if hit is not None:
            return hit
            
        # Move to next node in traversal order [G]
        curr_node += 1
        
    return None

def first_node(tx0: float, ty0: float, txm: float, tym: float) -> int:
    """Determines the first child index to visit based on ray entry quadrant [G]."""
    if tx0 < txm and ty0 < tym:
        return 0
    elif tx0 >= txm and ty0 < tym:
        return 1
    elif tx0 < txm and ty0 >= tym:
        return 2
    else:
        return 3
```

**Parameters:**
*   `start`: Vec2, ray origin [S1, unofficial].
*   `dir`: Vec2, ray direction [S1, unofficial].
*   `mask`: u8, bitmask for child index inversion. Bit 0 set if `dir.x < 0`, Bit 1 set if `dir.y < 0` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Division by zero if `dir.x` or `dir.y` is 0. Guard against this by using a small epsilon or handling axis-aligned rays separately [G].
*   **Boundaries:** The algorithm assumes `first_node` and `next_node` correctly determine traversal order based on ray direction relative to the current bounding box sub-regions [S1, unofficial]. Incorrect implementation of these helpers leads to missing hits or infinite loops [S1, unofficial].
*   **Degenerate Input:** Nodes with size 0 should be handled or avoided in the quadtree construction [G].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] Musta-Kissa/quadtree-raycast, https://github.com/Musta-Kissa/quadtree-raycast, fetched 2026-10-24 (unofficial)
