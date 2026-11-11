# Parametric Octree Traversal

Also called: Parametric Ray-Octree Traversal, Parametric Algorithm for Octree Traversal, Revelles 2000, Revelles' algorithm, Revelles Octree Traversal, An efficient parametric algorithm for octree traversal

## Overview
Parametric Octree Traversal is a top-down recursive algorithm for finding the subset of voxels in a sparse voxel octree pierced by a ray [S1, S5, unofficial]. It solves the problem of efficiently traversing non-uniform octrees without maintaining neighbor pointers or sorting potentially visited sub-voxels [S1, S5, unofficial]. It sets itself apart from bottom-up methods like Samet [S1, S5, unofficial] by avoiding neighbor finding, and from HERO [S1, S5, unofficial] by computing the sequence of visited sub-voxels sequentially using simple parametric comparisons rather than sorting [S1, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray $r$ defined by origin $p=(p_x, p_y, p_z)$ and direction $d=(d_x, d_y, d_z)$, and an octree structure containing a root node and bounding box dimensions ($xmin, xmax, ymin, ymax, zmin, zmax$) [S1, S5, unofficial].
*   **Output:** The set of terminal (leaf) nodes intersected by the ray [S1, S5, unofficial].

**Data Structures:**
*   **Octree:** Contains a pointer to the root node, dimensions ($xmin, xmax, ymin, ymax, zmin, zmax$), and sizes ($sizeX, sizeY, sizeZ$) [S1, S5, unofficial].
*   **Node:**
    *   `terminal`: Boolean indicating if the node is a leaf [S2, S3, S4, unofficial].
    *   `children`: Array of 8 pointers to child nodes, indexed 0–7 [S2, S3, S4, unofficial].
    *   (Optional) `debug_ID`: Integer identifier for debugging [S2, S3, S4, unofficial].

**Preprocessing (Handling Negative Directions):**
The algorithm assumes non-negative ray direction components for the base logic. If any component is negative, the ray is reflected across the center plane of the octree, and a bitmask `a` is set to track the inversion [S1, S2, S3, S4, S5, unofficial].

1.  Initialize `a = 0` [S1, S2, S3, S4, unofficial].
2.  If $d_x < 0$:
    *   $p_x \leftarrow sizeX - p_x$ [S1, S2, unofficial] or $p_x \leftarrow sizeX - p_x$ [S3, S4, S5, unofficial] (Note: S1/S2 use size, S3/S4/S5 use center*2 which is equivalent to size for axis-aligned bounds starting at 0).
    *   $d_x \leftarrow -d_x$ [S1, S2, S3, S4, S5, unofficial].
    *   `a |= 4` [S1, S2, S3, S4, S5, unofficial].
3.  If $d_y < 0$:
    *   $p_y \leftarrow sizeY - p_y$ [S1, S2, unofficial] or $p_y \leftarrow sizeY - p_y$ [S3, S4, S5, unofficial].
    *   $d_y \leftarrow -d_y$ [S1, S2, S3, S4, S5, unofficial].
    *   `a |= 2` [S1, S2, S3, S4, S5, unofficial].
4.  If $d_z < 0$:
    *   $p_z \leftarrow sizeZ - p_z$ [S1, S2, unofficial] or $p_z \leftarrow sizeZ - p_z$ [S3, S4, S5, unofficial].
    *   $d_z \leftarrow -d_z$ [S1, S2, S3, S4, S5, unofficial].
    *   `a |= 1` [S1, S2, S3, S4, S5, unofficial].

**Parametric Interval Calculation:**
Compute inverse direction components for stability [S2, S3, S4, unofficial]:
$$div_x = 1/d_x, \quad div_y = 1/d_y, \quad div_z = 1/d_z$$
Handle $d_i=0$ by allowing infinite values: $x/0 = +\infty$ if $x>0$, $-\infty$ if $x<0$ [S1, S5, unofficial].

Compute initial entry/exit t-values for the octree bounding box [S1, S2, S3, S4, S5, unofficial]:
$$tx_0 = (xmin - p_x) \cdot div_x, \quad tx_1 = (xmax - p_x) \cdot div_x$$
$$ty_0 = (ymin - p_y) \cdot div_y, \quad ty_1 = (ymax - p_y) \cdot div_y$$
$$tz_0 = (zmin - p_z) \cdot div_z, \quad tz_1 = (zmax - p_z) \cdot div_z$$

**Intersection Test:**
Check if the ray intersects the octree:
$$t_{min} = \max(tx_0, ty_0, tz_0)$$
$$t_{max} = \min(tx_1, ty_1, tz_1)$$
If $t_{min} \ge t_{max}$, the ray misses the octree [S1, S2, S3, S4, S5, unofficial]. Otherwise, the algorithm calls `proc_subtree` with the root node and the six parametric t-values [S1, unofficial].

**Traversal Logic (`proc_subtree`):**
For a node with parametric bounds $(tx_0, ty_0, tz_0)$ to $(tx_1, ty_1, tz_1)$:

1.  **Early Exit:** If $tx_1 < 0$ or $ty_1 < 0$ or $tz_1 < 0$, return (ray is behind the viewer) [S1, S2, S3, S4, S5, unofficial].
2.  **Leaf Check:** If the node is terminal, process it and return [S1, unofficial].
3.  **Midpoints:** Compute mid-plane parameters [S1, S2, S3, S4, S5, unofficial]:
    $$tx_m = 0.5(tx_0 + tx_1)$$
    $$ty_m = 0.5(ty_0 + ty_1)$$
    $$tz_m = 0.5(tz_0 + tz_1)$$
4.  **First Node:** Determine the first child to visit using `first_node(tx_0, ty_0, tz_0, tx_m, ty_m, tz_m)` [S1, S2, S3, S4, S5, unofficial].
5.  **Loop:** Iterate through children using a `while` loop while `curr_node < 8` [S1, S2, S3, S4, S5, unofficial].

**`first_node` Function:**
Determines the entry plane by finding $\max(tx_0, ty_0, tz_0)$ [S1, S5, unofficial].
*   If $tx_0$ is max (Entry YZ plane):
    *   Bit 1 set if $ty_m < tx_0$.
    *   Bit 0 set if $tz_m < tx_0$.
    *   Return index formed by bits [S1, S2, S3, S5, unofficial].
*   Else if $ty_0$ is max (Entry XZ plane):
    *   Bit 2 set if $tx_m < ty_0$.
    *   Bit 0 set if $tz_m < ty_0$.
    *   Return index formed by bits [S1, S2, S3, S5, unofficial].
*   Else (Entry XY plane, $tz_0$ is max):
    *   Bit 2 set if $tx_m < tz_0$.
    *   Bit 1 set if $ty_m < tz_0$.
    *   Return index formed by bits [S1, S2, S3, S5, unofficial].

**`new_node` Function:**
Determines the next child to visit based on the exit plane (minimum of $tx_m, ty_m, tz_m$) [S1, S2, S3, S4, S5, unofficial].
*   If $tx_m < ty_m$ and $tx_m < tz_m$: Return X-index (4, 5, 6, or 7 depending on context) [S2, S3, S4, S5, unofficial].
*   Else if $ty_m < tz_m$: Return Y-index (2, 3, 6, or 7) [S2, S3, S4, S5, unofficial].
*   Else: Return Z-index (1, 3, 5, or 7) [S2, S3, S4, S5, unofficial].
*   *Note:* The specific integer return values depend on the current state and are hardcoded in the switch cases below [S1, S2, S3, S4, S5, unofficial].

**Switch Case Logic (Iterating Children):**
The traversal uses a switch statement on `curr_node`. For each case, it recursively calls `proc_subtree` with the appropriate sub-volume bounds and updates `curr_node` using `new_node` [S1, S2, S3, S4, S5, unofficial]. The child index is accessed as `node.children[curr_node ^ a]` to account for axis inversions [S1, S2, S3, S4, S5, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
a = 0  # Global bitmask for axis inversions

def proc_terminal(node):
    # Process leaf node (e.g., accumulate color, add to list)
    pass

def ray_parameter(octree, ray):
    global a
    a = 0
    if ray.dx < 0.0:
        ray.ox = octree.sizeX - ray.ox
        ray.dx = -ray.dx
        a |= 4
    if ray.dy < 0.0:
        ray.oy = octree.sizeY - ray.oy
        ray.dy = -ray.dy
        a |= 2
    if ray.dz < 0.0:
        ray.oz = octree.sizeZ - ray.oz
        ray.dz = -ray.dz
        a |= 1
    tx0 = (octree.xmin - ray.ox) / ray.dx
    tx1 = (octree.xmax - ray.ox) / ray.dx
    ty0 = (octree.ymin - ray.oy) / ray.dy
    ty1 = (octree.ymax - ray.oy) / ray.dy
    tz0 = (octree.zmin - ray.oz) / ray.dz
    tz1 = (octree.zmax - ray.oz) / ray.dz
    if max(tx0, ty0, tz0) < min(tx1, ty1, tz1):
        proc_subtree(tx0, ty0, tz0, tx1, ty1, tz1, octree.root)

def proc_subtree(tx0, ty0, tz0, tx1, ty1, tz1, node):
    if tx1 < 0.0 or ty1 < 0.0 or tz1 < 0.0:
        return
    if node.terminal:
        proc_terminal(node)
        return
    txm = 0.5 * (tx0 + tx1)
    tym = 0.5 * (ty0 + ty1)
    tzm = 0.5 * (tz0 + tz1)
    curr_node = first_node(tx0, ty0, tz0, txm, tym, tzm)
    while curr_node < 8:
        if curr_node == 0:
            proc_subtree(tx0, ty0, tz0, txm, tym, tzm, node.children[curr_node ^ a])
            curr_node = new_node(txm, 4, tym, 2, tzm, 1)
        elif curr_node == 1:
            proc_subtree(tx0, ty0, tzm, txm, tym, tz1, node.children[curr_node ^ a])
            curr_node = new_node(txm, 5, tym, 3, tz1, 8)
        elif curr_node == 2:
            proc_subtree(tx0, tym, tz0, txm, ty1, tzm, node.children[curr_node ^ a])
            curr_node = new_node(txm, 6, ty1, 8, tzm, 3)
        elif curr_node == 3:
            proc_subtree(tx0, tym, tzm, txm, ty1, tz1, node.children[curr_node ^ a])
            curr_node = new_node(txm, 7, ty1, 8, tz1, 8)
        elif curr_node == 4:
            proc_subtree(txm, ty0, tz0, tx1, tym, tzm, node.children[curr_node ^ a])
            curr_node = new_node(tx1, 8, tym, 6, tzm, 5)
        elif curr_node == 5:
            proc_subtree(txm, ty0, tzm, tx1, tym, tz1, node.children[curr_node ^ a])
            curr_node = new_node(tx1, 8, tym, 7, tz1, 8)
        elif curr_node == 6:
            proc_subtree(txm, tym, tz0, tx1, ty1, tzm, node.children[curr_node ^ a])
            curr_node = new_node(tx1, 8, ty1, 8, tzm, 7)
        elif curr_node == 7:
            proc_subtree(txm, tym, tzm, tx1, ty1, tz1, node.children[curr_node ^ a])
            curr_node = 8

def first_node(tx0, ty0, tz0, txm, tym, tzm):
    if tx0 > ty0 and tx0 > tz0:
        ans = 0
        if tym < tx0: ans |= 2
        if tzm < tx0: ans |= 1
        return ans
    elif ty0 > tz0:
        ans = 0
        if txm < ty0: ans |= 4
        if tzm < ty0: ans |= 1
        return ans
    else:
        ans = 0
        if txm < tz0: ans |= 4
        if tym < tz0: ans |= 2
        return ans

def new_node(txm, x, tym, y, tzm, z):
    if txm < tym:
        if txm < tzm: return x
    else:
        if tym < tzm: return y
    return z
```

**Edge Cases and Pitfalls:**
*   **Zero Direction Components:** Rays parallel to axes cause division by zero. Handled by IEEE floating point infinities ($+\infty, -\infty$) [S1, S4, S5, unofficial].
*   **Negative Directions:** Must be handled by reflection and bitmask `a` before traversal begins [S1, S2, S3, S4, S5, unofficial].
*   **No Intersection:** The initial bounding box check prevents unnecessary traversal [S2, S3, S4, S5, unofficial].

## Subtopics

### [GPU Parametric Octree Traversal](3504-gpu-parametric-octree-traversal.md)
Adapts the CPU-based parametric traversal for GPU execution using a mipmap hierarchy to privilege arithmetic over memory accesses, suitable for soft shadow computation.

### [Parametric Quadtree Traversal](5018-parametric-quadtree-traversal.md)
Applies the same parametric coordinate logic to 2D quadtrees, normalizing coordinates via flipping to avoid branching on direction signs.

## Operating characteristics
*   **Complexity:** Within a single parent node, the longest path involves 4 children (3 transitions), totaling 10 comparisons [S1, unofficial].
*   **Memory:** Requires less memory than bottom-up methods like Samet-Net because it does not store neighbor voxel pointers [S1, S5, unofficial].
*   **Performance:** On a Silicon Graphics Indigo 2 (MIPS R4000), it was 10.4%, 12.7%, and 6.8% faster than binary tree traversal for scenes with 512, 4096, and 32768 spheres respectively [S1, S5, unofficial]. It outperforms bottom-up methods (Samet, SametCorner, SametNet) and is comparable to or faster than Gargantini [S1, S5, unofficial].
*   **Strengths:** Simple implementation, no neighbor finding, efficient for full or nearly full octrees [S1, S5, unofficial].
*   **Limitations:** Recursive implementation may cause stack overflow for very deep trees (though iterative stack-based versions exist) [S1, unofficial].

## Background
The algorithm was introduced by J. Revelles, C. Ureña, and M. Lastra in 2000 [S1, S2, S5, unofficial]. It is based on the principles of parametric methods for BSP trees and octrees, differing in the selection of the first sub-voxel using simple comparisons on parameter values [S1, unofficial]. It avoids the neighbor finding process of Samet [S1, S5, unofficial] and the sorting of potentially visited sub-voxels of HERO [S1, S5, unofficial].

## Sources
[S1] An Efficient Parametric Algorithm for Octree Traversal, http://wscg.zcu.cz/wscg2000/Papers_2000/X31.pdf, fetched 2026-10-24 (unofficial)
[S2] Ray / Octree traversal: Parametric algorithm implementation - Jeroen Baert's Blog, https://forceflow.be/2012/05/10/ray-octree-traversal-parametric-algorithm-implementation/, fetched 2026-10-24 (unofficial)
[S3] Ray - Octree intersection algorithms, https://stackoverflow.com/questions/10228690/ray-octree-intersection-algorithms, fetched 2026-10-24 (unofficial)
[S4] Ray - Octree intersection algorithms, https://www.stackoverflow.com/questions/10228690, fetched 2026-10-25 (unofficial)
[S5] An Efficient Parametric Algorithm For Octree Traversal, https://www.scribd.com/document/74957433/An-Efficient-Parametric-Algorithm-for-Octree-Traversal, fetched 2026-10-25 (unofficial)
