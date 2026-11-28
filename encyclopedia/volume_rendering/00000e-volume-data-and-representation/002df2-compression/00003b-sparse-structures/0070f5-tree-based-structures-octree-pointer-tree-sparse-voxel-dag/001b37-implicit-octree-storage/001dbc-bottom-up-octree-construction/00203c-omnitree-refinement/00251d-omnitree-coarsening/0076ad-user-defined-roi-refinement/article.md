# User-Defined ROI Refinement

Also called: Local Refinement, Octree-based Multiresolution Volume Rendering, Octree-based approach, ROI-based Octree Refinement

## Overview
User-Defined ROI Refinement is an interactive technique for multiresolution volume rendering of large scattered datasets, allowing users to specify Regions of Interest (ROI) in a coarse preview to generate higher-fidelity views of specific areas [S1, unofficial]. It distinguishes itself from static multiresolution methods by enabling dynamic, user-driven subdivision of octree leaf nodes covering the ROI, balancing global performance with local detail [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of scattered 3D points with associated scalar function values without connectivity information [S1, unofficial].
*   **Output:** A rendered image where the ROI is displayed with higher resolution (finer octree depth) than the background [S1, unofficial].

**Data Structures:**
*   **Octree:** A hierarchical structure where each node represents a bounding box. Nodes are subdivided until a termination criterion is met [S1, unofficial].
*   **Node Attributes:** Each node stores the average alpha ($A$) and color ($C$) of all points contained within its bounding box [S1, unofficial].
*   **Global/Local Structure:** The method may use a single global octree for the entire dataset, or a global octree combined with one or more local octrees dedicated to specific ROIs [S1, unofficial].

**Algorithm Steps:**

1.  **Pre-processing (Global Octree Construction):**
    *   Compute the bounding box of all original scattered points [S1, unofficial].
    *   Subdivide the bounding box octree-fashion until each leaf node satisfies a termination criterion [S1, unofficial].
    *   **Termination Criteria:** Stop subdividing a node if:
        1.  The number of original points lying in the node is smaller than a threshold number, OR
        2.  The error between the function values of the original points inside the node and an average function value associated with the node is smaller than some error threshold [S1, unofficial].

2.  **Coarse Approximation Generation:**
    *   Select a maximum depth $K$ for the initial preview [S1, unofficial].
    *   Process only nodes at depth $\le K$ [S1, unofficial].
    *   Approximate all points inside a node by a single representative point at the node center, using the average alpha $A$ and color $C$ of the contained points [S1, unofficial].

3.  **Weight Function Optimization:**
    *   Pre-compute a threshold $l'$ for various opacity values [S1, unofficial].
    *   If the intersection length $l > l'$, set the weight function $F(l, A) = 1$ to avoid expensive power calculations, as the function increases quickly and saturates [S1, unofficial].

4.  **Ray Traversal and Blending:**
    *   Traverse the octree recursively [S1, unofficial].
    *   **Intersection Length ($l$):** Calculate the approximate number of points along the ray inside a cell:
        $$ l = \frac{n d}{\text{edge}} $$
        Where:
        *   $n = \sqrt[3]{N_{pts}}$, with $N_{pts}$ being the number of points in the cell (assuming uniform distribution).
        *   $d$ is the distance between the ray's entry and exit points in the cell.
        *   $\text{edge}$ is the length of the cell's bounding box edge [S1, unofficial].
    *   To compute $d$ efficiently when the ray direction is constant, the source provides cosine-based approximations for axis-aligned rays: $d = (b.y - a.y) \cos \beta$ or $d = (b.z - a.z) \cos \gamma$, avoiding the computationally expensive general Euclidean distance formula [S1, unofficial].
    *   **Weight Function:**
        $$ F(l, A) = 1 - (1 - A)^l $$
    *   **Blending Formula:**
        $$ B_i = F(l_{i+1}, A_{i+1})C_{i+1} + (1 - F(l_{i+1}, A_{i+1}))B_{i+1} $$
        Where $B_i$ is the resulting color at the $i$-th node, and $A_{i+1}$ and $C_{i+1}$ are average alpha and color values of points inside the $(i+1)$-th node [S1, unofficial].
    *   **Ray Termination:** Stop traversal when:
        $$ (1 - A_1)(1 - A_2)\dots(1 - A_i) < e $$
        Where $e$ is a threshold value [S1, unofficial].

5.  **Local Refinement (ROI Specification):**
    *   After generating the coarse image, the user specifies an ROI (e.g., via a bounding box) [S1, unofficial].
    *   Identify all octree leaf nodes that cover the ROI coordinates [S1, unofficial].
    *   **Subdivision:** Subdivide these identified leaf nodes to a deeper level (higher resolution) to improve image quality in the ROI [S1, unofficial].
    *   If using **Local Octrees**, build a new, shallower but denser octree specifically for the ROI points, using the global octree as a search tree to locate the relevant bounding box [S1, unofficial].

6.  **Coarsening (Inverse Operation):**
    *   To reduce storage or accelerate rendering, users can merge leaf nodes back into larger nodes [S1, unofficial].
    *   This involves fusing children along specified dimensions, effectively reversing the refinement process [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_intersects_box(ray, bbox):
    """Standard AABB ray-box intersection test [G]."""
    t_min, t_max = -float('inf'), float('inf')
    for i in range(3):
        if ray.dir[i] == 0:
            if ray.origin[i] < bbox.min[i] or ray.origin[i] > bbox.max[i]:
                return False
        else:
            t1 = (bbox.min[i] - ray.origin[i]) / ray.dir[i]
            t2 = (bbox.max[i] - ray.origin[i]) / ray.dir[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    return t_max >= t_min

def ray_box_intersection_distance(ray, bbox):
    """Returns distance between entry and exit points along ray [G]."""
    t_min, t_max = -float('inf'), float('inf')
    for i in range(3):
        if ray.dir[i] == 0:
            if ray.origin[i] < bbox.min[i] or ray.origin[i] > bbox.max[i]:
                return 0.0
        else:
            t1 = (bbox.min[i] - ray.origin[i]) / ray.dir[i]
            t2 = (bbox.max[i] - ray.origin[i]) / ray.dir[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    return t_max - t_min

def distance_from_view_plane(child_bbox, view_origin):
    """Simple heuristic for sorting children front-to-back [G]."""
    center = (child_bbox.min + child_bbox.max) / 2.0
    return (center - view_origin).dot(ray.dir)

def traverse(ray, node, depth, max_depth_K, threshold_e, state):
    if not ray_intersects_box(ray, node.bbox):
        return
    if node.is_leaf or depth >= max_depth_K:
        d = ray_box_intersection_distance(ray, node.bbox)
        n_pts = node.point_count
        edge = node.edge_length
        if edge == 0 or n_pts == 0:
            return
        n = n_pts ** (1/3)
        l = (n * d) / edge
        if l > state.l_prime_threshold:
            weight = 1.0
        else:
            weight = 1.0 - (1.0 - node.alpha) ** l
        state.accumulated_color = weight * node.color + (1.0 - weight) * state.accumulated_color
        state.accumulated_opacity = 1.0 - (1.0 - state.accumulated_opacity) * node.alpha
        if (1.0 - state.accumulated_opacity) < threshold_e:
            state.enough = True
    else:
        children = sorted(node.children, key=lambda c: distance_from_view_plane(c.bbox, ray.origin))
        for child in children:
            traverse(ray, child, depth + 1, max_depth_K, threshold_e, state)
            if state.enough:
                return

# Initialization for a single ray cast
state = {
    'accumulated_color': [0.0, 0.0, 0.0],
    'accumulated_opacity': 0.0,
    'enough': False,
    'l_prime_threshold': precomputed_threshold  # Precomputed per alpha value
}
traverse(ray, root_node, 0, K, threshold_e, state)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Calculating $l$ involves floating-point division and cube roots. Use the precomputed threshold $l'$ to avoid expensive power operations when $l$ is large [S1, unofficial].
*   **Boundaries:** Ray-box intersection distance $d$ must be calculated efficiently; use the cosine-based approximations for axis-aligned rays to avoid the computationally expensive general Euclidean distance formula [S1, unofficial].
*   **Degenerate Input:** If a node has 0 points or 0 edge length, handle division by zero in the $l$ calculation [S1, unofficial].

## Subtopics

### Global and Local Octrees
A variant where the dataset is represented by one global octree for the entire volume and one or more local octrees for specific ROIs [S1, unofficial]. The global octree is pre-constructed to save interactive time [S1, unofficial]. Local octrees are built in real-time by using the global octree to locate the bounding box containing the ROI, then constructing a new octree only for points within that box [S1, unofficial]. Local octrees typically have less depth than the global octree but approximate fewer points per node, yielding better approximation quality for the covered space (e.g., global depth 7 vs. local depth 3 for the same data density) [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Performance:** The coarse approximation allows for fast preview. Refinement adds overhead only for the ROI. Coarsening reduces memory and speeds up rendering by merging nodes [S1, unofficial].
*   **Strengths:** Interactive exploration of large datasets; user-driven focus; efficient storage via error-controlled compression [S1, unofficial].
*   **Limitations:** Requires pre-processing; scattered data assumption (no connectivity); weight function approximation may introduce artifacts if $l$ is estimated poorly [S1, unofficial].

## Background
The method was proposed by E. Brugger, B. Hamann, and colleagues at Lawrence Livermore National Laboratory and UC Davis [S1, unofficial]. It addresses the challenge of rendering large scattered volume datasets (e.g., from scientific simulations) where traditional ray casting is too slow [S1, unofficial]. The approach incorporates Levoy's ray termination condition to accelerate traversal [S1, unofficial].

## Sources
[S1] An octree-based multiresolution approach supporting interactive rendering of very large volume data sets, https://escholarship.org/uc/item/44p5z0p4, fetched 2026-10-26 (unofficial)
