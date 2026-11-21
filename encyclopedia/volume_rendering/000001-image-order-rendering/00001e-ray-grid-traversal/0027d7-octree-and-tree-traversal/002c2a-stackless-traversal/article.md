# Stackless Traversal

Also called: Algorithme sans pile, Parcours sans pile, Parent-link traversal, Stack-less Tree Traversal

## Overview
Stackless Traversal is a method for traversing hierarchical volume structures (such as N3-trees or octrees) that eliminates the need for an explicit stack data structure by recalculating Digital Differential Analyzer (DDA) parameters at every descent and ascent [S1, unofficial]. It sets itself apart from sibling methods like [DDA Tree Traversal with Mini-Stack](5244-dda-tree-traversal-with-mini-stack.md) by relying on parent pointers stored within the hierarchy to navigate, trading memory usage for computational cost [S1, unofficial]. This approach is particularly advantageous in memory-constrained environments, such as early GPU architectures where dynamic array indexing forced stack data into slow local memory [S1, unofficial].

## How it works
The algorithm traverses a hierarchical grid by maintaining the current ray state and navigating the tree structure using explicit parent links [S1, unofficial]. Unlike stack-based methods that push and pop nodes, this method recalculates the ray's entry point and DDA deltas whenever it moves between a parent and child node [S1, unofficial].

### Inputs and Outputs
*   **Input:** Ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; Hierarchical volume structure where each node stores a pointer to its parent and its spatial bounds; Grid resolution and voxel size [S1, unofficial].
*   **Output:** A sequence of visited nodes (bricks) and their corresponding traversal intervals $[t_{entry}, t_{exit}]$ for volume integration [S1, unofficial].

### Data Structures
The hierarchical structure must support:
1.  **Parent Pointer:** Each node contains a reference to its immediate parent node [S1, unofficial].
2.  **Child Pointer:** Each node contains references to its children (if not a leaf) [G].
3.  **Spatial Bounds:** Each node stores its minimum and maximum coordinates (or index ranges) to define its voxel extent [G].
4.  **Node Type/Status:** Flags indicating if the node is empty, a leaf (brick), or an internal node [S1, unofficial].

### DDA Parameters
For a uniform grid with voxel size $v$ and dimensions $N_x, N_y, N_z$, the traversal relies on three key precomputed values per axis $i \in \{x, y, z\}$ [G]:

1.  **Step Direction ($\text{step}_i$):**
    $$
    \text{step}_i = \begin{cases} 
    +1 & \text{if } d_i > 0 \\
    -1 & \text{if } d_i < 0 \\
    0 & \text{if } d_i = 0 
    \end{cases}
    $$
2.  **Time/Distance Delta ($\Delta t_i$):**
    $$
    \Delta t_i = \frac{v_i}{|d_i|} \cdot \text{step}_i
    $$
    If $d_i = 0$, the ray never crosses a boundary in that dimension, and the corresponding distance is treated as infinity [G].
3.  **Time/Distance to Next Boundary ($t_{max, i}$):**
    $$
    t_{max, i} = \frac{\text{boundary}_i - o_i}{d_i}
    $$
    The initial boundary is determined by the starting voxel index [G].

### Algorithm Steps
1.  **Initialization:** Compute initial DDA parameters ($\text{step}$, $\Delta t$, $t_{max}$) for the ray origin within the root node's bounds. Identify the starting leaf node containing the ray origin [G].
2.  **Process Current Node:**
    *   If the current node is **empty**, skip it and proceed to the next step (ascent) [S1, unofficial].
    *   If the current node is a **leaf (brick)**:
        *   Perform ray marching within the brick's volume [S1, unofficial].
        *   If data is unavailable (e.g., dynamic loading failure), use the stored average and accumulate it based on the distance traversed by the ray in that brick (pre-integration) [S1, unofficial].
    *   If the current node is an **internal node**, proceed to navigation [S1, unofficial].
3.  **Navigation (Descent/Ascent):**
    *   **Ascent:** If the ray exits the current node, use the **parent pointer** stored in the structure to move to the parent node [S1, unofficial]. This link points directly to the exact sub-node of the parent, allowing extraction of the current cell information in the parent's grid without recalculation [S1, unofficial].
    *   **Descent:** If the ray enters a child node, **recalculate all DDA parameters** ($\text{step}$, $\Delta t$, $t_{max}$) for the new node [S1, unofficial].
4.  **Loop:** Repeat steps 2-3 until the ray exits the entire volume hierarchy [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stackless_traversal(ray_origin, ray_direction, root_node, voxel_size):
    # DDA initialization [G]
    step = [1 if d > 0 else (-1 if d < 0 else 0) for d in ray_direction]
    t_delta = [voxel_size / abs(d) if d != 0 else float('inf') for d in ray_direction]
    
    # Find initial leaf node containing ray_origin [G]
    current_node = find_leaf(root_node, ray_origin)
    current_ray_t = 0.0
    
    while current_node is not None:
        # Compute distance to next boundary on each axis for current node [G]
        t_max = compute_t_max(ray_origin, ray_direction, current_node.bounds, step, t_delta)
        
        # Exit time from current node
        t_exit = min(t_max)
        
        # Process node based on type [S1, unofficial]
        if current_node.is_empty:
            # Skip empty node, advance ray to exit
            current_ray_t += t_exit
            ray_origin = ray_origin + ray_direction * t_exit
            # Ascent: move to parent via stored pointer [S1, unofficial]
            current_node = current_node.parent
            continue
            
        if current_node.is_leaf:
            # Ray march within brick bounds [S1, unofficial]
            process_brick(current_node, ray_origin, ray_direction, t_exit)
            current_ray_t += t_exit
            ray_origin = ray_origin + ray_direction * t_exit
            # Ascent: move to parent via stored pointer [S1, unofficial]
            current_node = current_node.parent
            continue
        
        # Internal node: descend to child containing ray
        # Child index from ray position relative to node center [G]
        center = current_node.center
        child_idx = 4 * (ray_origin.x >= center.x) + 2 * (ray_origin.y >= center.y) + (ray_origin.z >= center.z)
        
        # Descent: move to child
        current_node = current_node.children[child_idx]
        
        # Recalculate all DDA parameters for the new child node [S1, unofficial]
        t_max = compute_t_max(ray_origin, ray_direction, current_node.bounds, step, t_delta)
        
        # Advance ray to child entry point
        t_entry = min(t_max)
        current_ray_t += t_entry
        ray_origin = ray_origin + ray_direction * t_entry
        
    return

# Helper: Find leaf containing point [G]
def find_leaf(root, point):
    node = root
    while not node.is_leaf:
        center = node.center
        idx = 4 * (point.x >= center.x) + 2 * (point.y >= center.y) + (point.z >= center.z)
        node = node.children[idx]
    return node

# Helper: Compute t_max for DDA [G]
def compute_t_max(origin, direction, bounds, step, t_delta):
    t_max = []
    for i in range(3):
        if direction[i] == 0:
            t_max.append(float('inf'))
        else:
            boundary = bounds.max[i] if step[i] > 0 else bounds.min[i]
            t_max.append((boundary - origin[i]) / direction[i])
    return t_max

# Helper: Ray march within brick [G]
def process_brick(leaf, origin, direction, t_exit):
    # Standard fixed-step sampling within leaf bounds [t_entry, t_exit]
    pass
```

### Parameters and Optimization
*   **Conditional Block Size:** Limit the size of `if...then...else` bodies to reduce divergent threads on GPU multiprocessors [S1, unofficial].
*   **Register Usage:** Limit the number of registers used to allow multiprocessors to handle more parallel threads, better covering texture access latency [S1, unofficial].

### Edge Cases and Pitfalls
*   **GPU Memory Latency:** On architectures like the NVIDIA G80, dynamically indexed arrays are placed in "local" memory (device memory), which is uncached and up to 50x slower than registers [S1, unofficial]. This negatively impacts performance compared to register-based stack access [S1, unofficial]. Local memory accesses incur maximum latency and transfer cost from graphics memory [S1, unofficial].
*   **Zero Direction Components:** When $d_i = 0$, the ray never crosses a boundary in that dimension, and the corresponding distance is treated as infinity to prevent division by zero [G].

## Subtopics

### [Restart Trail](9459-restart-trail.md)
A variant for BVHs that uses a bit-array to track processed subtrees, allowing complete stack exhaustion without state loss. Choose this when memory is extremely constrained and restarts are frequent.

## Operating characteristics
*   **Time Complexity:** Higher per-node computational cost due to DDA recalculation at every ascent/descent [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary stack space (excluding the tree structure itself) [S1, unofficial].
*   **Performance:** On the NVIDIA G80 GPU, the stackless algorithm was approximately 50% faster than the mini-stack algorithm [S1, unofficial]. This result is attributed to the G80's shader compiler placing dynamically indexed arrays (used for the stack) in slow, uncached local memory, whereas the stackless method kept data in registers or faster memory spaces [S1, unofficial].
*   **Strengths:** No stack overflow risk; minimal memory footprint; efficient on hardware with slow local memory for dynamic arrays [S1, unofficial].
*   **Limitations:** Higher arithmetic intensity per node visit; requires parent pointers in the data structure [S1, unofficial].
*   **When to Use:** Memory-constrained environments (e.g., embedded systems, early GPUs); scenarios where stack depth is unpredictable or large [S1, unofficial].
*   **When Not to Use:** Hardware with fast, cached local memory and efficient stack operations; scenes with very shallow hierarchies where recalculation overhead outweighs stack costs [G].

## Background
The Stackless Traversal algorithm was developed and tested for traversing hierarchical volume structures (N3-tree/octree) [S1, unofficial]. It was designed to address the limitations of stack-based methods on early GPU architectures (specifically the G80) where dynamic memory indexing was penalized by the hardware compiler [S1, unofficial]. The method leverages parent pointers stored in the hierarchical structure to navigate without a stack, recalculating DDA parameters to maintain correctness [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
