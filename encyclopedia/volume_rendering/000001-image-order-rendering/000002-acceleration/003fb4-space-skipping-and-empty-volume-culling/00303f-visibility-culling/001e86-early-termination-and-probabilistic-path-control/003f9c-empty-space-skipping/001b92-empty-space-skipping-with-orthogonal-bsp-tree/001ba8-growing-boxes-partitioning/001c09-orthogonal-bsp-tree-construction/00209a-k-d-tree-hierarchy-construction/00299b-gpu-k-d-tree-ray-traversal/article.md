# GPU K-D Tree Ray Traversal

Also called: K-D Tree GPU Traversal, Kd-Tree Ray Tracing on GPU

## Overview
GPU K-D Tree Ray Traversal accelerates ray tracing on graphics hardware by intersecting rays with a hierarchical spatial partitioning structure (K-D tree) [S1, unofficial]. It efficiently skips empty space and identifies intersected primitives, avoiding the computational cost of testing every primitive in the scene [S1, unofficial]. It is distinguished from CPU implementations by its adaptation to GPU architectures: early Shader Model 2.0 hardware used stackless multi-pass streaming kernels [S1, unofficial], while Shader Model 3.0 introduced single-pass traversal via dynamic branching [S1, unofficial]. It builds on [K-D Tree Hierarchy Construction](2300-k-d-tree-hierarchy-construction.md).

## How it works
The K-D tree is a hierarchical spatial partitioning structure where each node is divided into two sub-nodes by a plane orthogonal to an axis, positioned arbitrarily along that axis within the node [S1, unofficial]. It is a special case of a BSP-Tree [S1, unofficial].

### Inputs and Outputs
- **Input**: A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a K-D tree structure where each node contains a split axis, split position, and references to left/right children or leaf primitives [S1, unofficial].
- **Output**: The closest intersection point or hit primitive along the ray [G].

### Data Structures
The tree nodes are typically stored in a flat array [G]. Each node contains:
- `axis`: The dimension (0, 1, or 2) along which the split occurs [G].
- `split`: The position of the splitting plane along the specified axis [G].
- `left`: Index of the left child node [G].
- `right`: Index of the right child node [G].
- `prim_count`: Number of primitives in the leaf [G].
- `prim_offset`: Offset into a global primitive array for leaf nodes [G].

### Traversal Steps
1. Initialize the ray slab with near and far clip distances [G].
2. Start traversal at the root node [G].
3. Calculate the intersection parameter $t$ between the ray and the current node's split plane [G].
4. Determine traversal order based on the ray origin's position relative to the split plane [G].
5. Update the slab bounds ($t_{min}$, $t_{max}$) if the split plane is within the current ray segment [G].
6. Advance to the child node index [G].
7. If the current node is a leaf, test contained primitives against the ray slab [G].
8. Terminate when a hit is found, the slab is exhausted, or the maximum depth is reached [G].

### Formulas
The intersection parameter $t$ for the ray to reach the split plane is computed as:
$$t = \frac{s - o_a}{d_a}$$
where $s$ is the split position, $o_a$ is the ray origin component along the split axis $a$, and $d_a$ is the ray direction component along axis $a$ [G]. The ray-slab bounds $t_{min}$ and $t_{max}$ are updated by clamping $t$ to the current interval [G].

### Parameters
- `max_depth`: Maximum traversal depth to prevent infinite loops on degenerate trees; typically set to the tree height or a fixed constant like 32 [G].
- `t_min`: Near clip distance; typically $0.0$ or the camera's near plane [G].
- `t_max`: Far clip distance; typically $1000.0$ or the camera's far plane [G].

### Edge Cases and Pitfalls
- **Zero direction component**: If $d_a = 0$, the ray is parallel to the split plane; $t$ is undefined and the ray stays on one side of the plane [G].
- **Precision errors**: Floating-point inaccuracies near split planes can cause rays to miss nodes or traverse incorrectly; epsilon offsets are often applied [G].
- **Degenerate input**: Empty nodes or zero-volume cells can cause infinite loops; handled by `max_depth` or node validation [G].

### Pseudocode (Single-Pass Dynamic Branching)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_kd_tree(ray_origin, ray_direction, tree_nodes, max_depth):
    """
    Traverses a K-D tree for a single ray using dynamic branching.
    Assumes tree_nodes is a flat array of node structures.
    Returns the closest hit primitive index or -1 if no hit.
    """
    current_node = 0  # Start at root [G]
    t_min = 0.0       # Near clip plane [G]
    t_max = 1e9       # Far clip plane [G]
    
    for step in range(max_depth):
        node = tree_nodes[current_node]
        
        # Check if leaf
        if node.prim_count > 0:
            closest_prim = -1
            for i in range(node.prim_count):
                prim_idx = node.prim_offset + i
                t_hit = intersect_primitive(ray_origin, ray_direction, prim_idx)
                if t_hit is not None and t_min <= t_hit < t_max:
                    t_max = t_hit
                    closest_prim = prim_idx
            return closest_prim
        
        # Determine which child to visit first
        split_axis = node.axis
        split_pos = node.split
        
        # Ray parameter to intersection with split plane
        if ray_direction[split_axis] == 0.0:
            t_split = float('inf') if split_pos > ray_origin[split_axis] else float('-inf')
        else:
            t_split = (split_pos - ray_origin[split_axis]) / ray_direction[split_axis]
        
        # Determine traversal order based on ray origin side
        if ray_origin[split_axis] < split_pos:
            if t_split < t_min:
                current_node = node.right
            else:
                current_node = node.left
                if t_split < t_max:
                    t_max = t_split
        else:
            if t_split < t_min:
                current_node = node.left
            else:
                current_node = node.right
                if t_split < t_max:
                    t_max = t_split
        
        # If t_max <= t_min, we have passed the region of interest
        if t_max <= t_min:
            return -1
            
    return -1

def intersect_primitive(origin, direction, prim_idx):
    """Standard primitive intersection routine [G]."""
    # Implementation depends on primitive type (triangle, sphere, etc.)
    return None
```

## Operating characteristics
- **Time Complexity**: $O(\log N)$ on average for balanced trees, where $N$ is the number of primitives; worst case $O(N)$ for degenerate trees [G].
- **Memory Complexity**: $O(N)$ to store the tree nodes [G].
- **Performance**: Early GPU implementations (Foley and Sugerman) were slower than CPU counterparts due to streaming limitations [S1, unofficial]. Later implementations (Horn et al.) became competitive with CPU but were limited by SIMD divergence penalties on Shader Model 3.0 hardware [S1, unofficial].
- **Strengths**: Efficient empty-space skipping; flexible for arbitrary scenes [S1, unofficial].
- **Limitations**: Dynamic branching causes divergence penalties on SIMD GPUs; stackless methods require multiple passes or fixed-depth loops [S1, unofficial].

## Background
- **Origin**: The K-D tree was shown by Havran [Hav00] to be the most flexible and provide average best results for arbitrary scenes in CPU ray tracing [S1, unofficial].
- **History**: The first GPU ray tracing use of K-D trees was proposed by Ernst et al. [EVG04], using a stack and a heavy multi-pass algorithm, which did not achieve high performance [S1, unofficial]. Foley and Sugerman [FS05] improved this with stackless multi-pass methods for Shader Model 2.0 [S1, unofficial]. Horn et al. [HSHH07] proposed the first efficient GPU method competitive with CPU implementations, using Shader Model 3.0 dynamic branching for a single-pass algorithm [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
