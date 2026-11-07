# Stackless Traversal

Also called: Algorithme sans pile, Parcours sans pile, Parent-link traversal, Stack-less Tree Traversal

## Overview
Stackless Traversal is a ray-traversal algorithm for hierarchical volume structures (such as octrees or N3-trees) that eliminates the need for an explicit stack data structure by recalculating Digital Differential Analyzer (DDA) parameters at every descent and ascent step [S1, unofficial]. It distinguishes itself from sibling methods like [DDA Tree Traversal with Mini-Stack](5244-dda-tree-traversal-with-mini-stack.md) by trading increased computational cost for reduced memory usage and avoidance of stack-related overheads, relying instead on parent pointers stored within the hierarchical structure to navigate back up the tree [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a hierarchical volume structure (e.g., octree) where each node stores a pointer to its parent and its spatial bounds.
*   **Output:** A sequence of volume samples or a compositing result along the ray path.

**Data Structures:**
*   **Node Structure:** Each node contains:
    *   `parent`: A direct pointer to the parent node. This pointer references the exact child slot within the parent's grid, allowing extraction of the current cell's relative position in the parent without recalculation [S1, unofficial].
    *   `bounds`: Bounding box of the node.
    *   `data`: Volume data (if leaf) or metadata (if internal).
    *   `is_empty`: Boolean flag indicating if the subtree is empty [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** Compute initial DDA parameters for the ray entering the root node.
2.  **Descent:** Determine the child octant intersected by the ray using the standard child selection formula provided in [Octree and Tree Traversal](4134-octree-and-tree-traversal.md).
3.  **Empty Check:** If the selected child is marked as empty, skip the subtree. This is achieved by advancing the ray parameter $t$ to the exit of the current node or using neighbor pointers to jump to the next potentially occupied node [S1, unofficial][G].
4.  **Leaf Processing:** If the current node is a leaf:
    *   If data is available, perform regular ray-marching within the brick's data [S1, unofficial].
    *   If data is unavailable (e.g., dynamic loading failure), use the stored average color/value and accumulate it based on the distance traversed by the ray in that brick (pre-integration) [S1, unofficial].
5.  **Ascent:** If the ray exits a subtree or needs to backtrack, use the `parent` pointer to move to the parent node. The direct pointer allows extraction of the current cell's information in the parent's grid without recalculation [S1, unofficial].
6.  **Re-calculation:** Upon moving to a new node (either via descent or ascent), recalculate all DDA parameters for the ray relative to the new node's coordinate system and bounds [S1, unofficial].
7.  **Repeat:** Continue until the ray exits the root or reaches a maximum depth.

**Formulas:**
*   **Child Selection:** For a node with center $\mathbf{c}$ and half-extent $s$, the ray position $\mathbf{p}$ determines the child index $k \in \{0, \dots, 7\}$:
    $$
    k = 4 \cdot \mathbb{I}(p_x \ge c_x) + 2 \cdot \mathbb{I}(p_y \ge c_y) + 1 \cdot \mathbb{I}(p_z \ge c_z)
    $$
    where $\mathbb{I}(\cdot)$ is the indicator function [S1, unofficial][S6, S8, unofficial].
*   **DDA Recalculation:** At each step, the DDA step size $\Delta t$ and entry/exit parameters $t_{min}, t_{max}$ are recomputed based on the current node's bounds and the ray direction $\mathbf{d}$ [S1, unofficial].

**Parameters:**
*   **Conditional Block Size:** Limited to reduce divergent threads on GPU multiprocessors [S1, unofficial].
*   **Register Usage:** Limited to allow multiprocessors to handle more parallel threads, better covering texture access latency [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **GPU Memory Latency:** On older architectures like the G80, dynamically indexed arrays are placed in 'local' memory (device memory) by the shader compiler, which is uncached and up to 50x slower than registers [S1, unofficial]. This negatively impacts performance compared to register-based stack access. Local memory is uncached device memory, meaning every access incurs maximum latency and transfer cost from graphics memory [S1, unofficial].
*   **Precision:** Recalculating DDA parameters at every step may introduce floating-point precision errors over long traversal paths.
*   **Boundary Conditions:** Handling rays that exactly align with node boundaries requires consistent tie-breaking rules to avoid missing or double-sampling voxels.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stackless_traverse(ray_origin, ray_direction, root_node):
    # Initialize DDA parameters for the root node
    current_node = root_node
    dda_params = compute_dda(ray_origin, ray_direction, current_node.bounds)
    
    while current_node is not None:
        # Determine which child the ray enters
        child_index = get_child_index(ray_origin, ray_direction, current_node.center, current_node.half_extent)
        
        # Check if child exists and is not empty
        child_node = current_node.children[child_index]
        
        if child_node is None or child_node.is_empty:
            # Skip empty subtree
            # Advance ray to exit of current node
            t_exit = dda_params.t_max
            ray_origin = ray_origin + ray_direction * t_exit
            current_node = current_node.parent
            if current_node is not None:
                # Recalculate DDA for the parent context after ascent
                dda_params = compute_dda(ray_origin, ray_direction, current_node.bounds)
            continue
        
        # Move to child
        current_node = child_node
        # Recalculate DDA for the new node
        dda_params = compute_dda(ray_origin, ray_direction, current_node.bounds)
        
        # Process leaf nodes
        if current_node.is_leaf:
            if current_node.has_data:
                sample_volume(ray_origin, ray_direction, current_node.data, dda_params)
            else:
                # Use stored average for unloaded bricks
                accumulate_average(current_node.average_value, dda_params.t_max - dda_params.t_min)
        
        # Check if ray has exited the current node
        if ray_origin is outside(current_node.bounds):
            current_node = current_node.parent
            if current_node is not None:
                dda_params = compute_dda(ray_origin, ray_direction, current_node.bounds)
```

## Subtopics

### Mini-Stack Traversal
Uses a small, fixed-size stack stored in registers or shared memory to avoid global memory stack overhead. Choose when register pressure is low and stack depth is predictable. See [DDA Tree Traversal with Mini-Stack](5244-dda-tree-traversal-with-mini-stack.md).

### Hermite Data Computation via Bisection
Focuses on computing derivative data for smooth interpolation during traversal. Choose when high-quality gradients are needed. See [Hermite Data Computation via Bisection](822-hermite-data-computation-via-bisection.md).

### Octree Path Computation
Computes the path through the octree using binary location codes. Choose when memory layout allows efficient code-based navigation. See [Octree Path Computation](897-octree-path-computation.md).

## Operating characteristics
*   **Time Complexity:** Higher per-step computational cost due to repeated DDA parameter recalculation at every descent and ascent [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, as no stack is required. Memory is saved by storing parent pointers in the structure [S1, unofficial].
*   **Measured Performance:** On the NVIDIA G80 GPU, the stackless algorithm was approximately 50% faster than the mini-stack algorithm in tested scenarios [S1, unofficial]. However, this performance gain is heavily dependent on hardware architecture; on architectures where dynamic indexing incurs high local memory latency, the benefit may be reduced or reversed [S1, unofficial].
*   **Strengths:** Eliminates stack overflow risks; reduces memory bandwidth usage associated with stack operations; simplifies GPU shader implementation by avoiding complex stack management.
*   **Limitations:** Increased arithmetic intensity; potential precision loss from repeated recalculations; performance sensitive to GPU memory hierarchy (local vs. register access) [S1, unofficial].
*   **When to Use:** When memory is constrained, or when the overhead of managing a stack outweighs the cost of recalculation. Particularly effective on hardware with efficient parent-pointer access and low local memory latency.
*   **When Not to Use:** On hardware with high local memory latency for dynamic indexing (e.g., older GPUs like G80 if not optimized) or when traversal depth is very shallow and stack overhead is negligible.

## Background
Stackless Traversal was developed as an alternative to stack-based methods for interactive exploration of procedural volumes. It was implemented and tested by the authors of [S1, unofficial] to avoid the performance penalties associated with stack operations on early GPU architectures. The method was designed to limit conditional block sizes and register usage to improve thread divergence and occupancy on GPU multiprocessors [S1, unofficial]. It is part of a broader effort to optimize volume rendering algorithms for hardware constraints, specifically addressing the limitations of shader compilers in managing dynamic memory access [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
