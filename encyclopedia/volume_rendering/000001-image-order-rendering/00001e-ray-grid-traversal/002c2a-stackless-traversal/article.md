# Stackless Traversal

Also called: Algorithme sans pile, Parcours sans pile, Parent-link traversal, Stack-less Tree Traversal

## Overview
Stackless Traversal is a hierarchical volume rendering method that traverses tree structures (specifically N3-trees) without a stack data structure by recalculating Digital Differential Analyzer (DDA) parameters at each tree descent and ascent [S1, unofficial]. It differs from sibling methods like [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md) and [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) by relying on explicit parent pointers for upward navigation and full DDA recalculation rather than incremental updates or implicit stack management, optimizing for GPU register pressure and branch divergence [S1, unofficial].

## How it works
**Inputs:** A hierarchical volume structure (N3-tree) where each node stores a pointer to its parent and, if non-empty, an average color/value and occupancy status [S1, unofficial]. A ray defined by origin and direction [S1, unofficial].
**Outputs:** Integrated color and opacity for the ray [S1, unofficial].

**Data Structures:**
*   **Node:** Contains a pointer to the parent node, a pointer to child nodes (if internal), and metadata (average color, occupancy) [S1, unofficial].
*   **DDA State:** Current position, step size, and direction signs for the current voxel/brick [S1, unofficial].

**Steps:**
1.  **Initialization:** Compute initial DDA parameters for the root node based on the ray origin and direction [S1, unofficial].
2.  **Descent:** Traverse down the tree. At each step, recalculate all DDA parameters for the child node from scratch based on the current ray state and the child's spatial bounds [S1, unofficial].
3.  **Node Evaluation:** If the current node is empty, ignore it and proceed to the next candidate [S1, unofficial]. If the node is not empty, determine if it is a loaded brick or an unloaded brick [S1, unofficial].
4.  **Accumulation:** If the node is a loaded brick, perform standard ray-marching sampling. If the node is an unloaded brick (due to dynamic loading failure or out-of-core constraints), use the stored average color/value and accumulate it based on the distance traversed by the ray in that brick (pre-integration) [S1, unofficial].
5.  **Ascent:** Return to the parent node using the parent pointer stored in the hierarchical structure [S1, unofficial]. Extract the current cell information directly from the parent's grid using the direct pointer to the exact child node, avoiding DDA recalculation during ascent [S1, unofficial].
6.  **Iteration:** Repeat descent and ascent until the ray exits the volume bounds or is fully occluded [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stackless_traverse(ray_o, ray_d, root):
    # Initialize DDA for root brick
    step = [1 if ray_d[i] > 0 else (-1 if ray_d[i] < 0 else 0) for i in range(3)]
    delta_t = [1.0 / abs(ray_d[i]) if ray_d[i] != 0 else float('inf') for i in range(3)]
    t_max = [(root.bounds[i].start - ray_o[i]) / ray_d[i] if ray_d[i] != 0 else float('inf') for i in range(3)]
    current = root
    
    while current is not None:
        # Descent: find next child index based on minimum t_max
        child_idx = -1
        min_t = float('inf')
        for i in range(3):
            if t_max[i] < min_t:
                min_t = t_max[i]
                child_idx = i
        if child_idx == -1:
            # Ascent: use parent pointer
            current = current.parent
            if current is not None:
                # Extract cell info from parent grid using direct pointer to exact child
                # Avoids full DDA recalculation during ascent
                t_max = extract_dda_from_parent(current, child_idx)
            continue
        
        current = current.children[child_idx]
        
        # Recalculate DDA for the new child's bounds from scratch
        t_max = [(current.bounds[i].start - ray_o[i]) / ray_d[i] if ray_d[i] != 0 else float('inf') for i in range(3)]
        
        if current.is_empty:
            current = current.parent
            continue
            
        # Compute intersection distance with current brick
        t_entry = max((current.bounds[i].start - ray_o[i]) / ray_d[i] for i in range(3) if ray_d[i] != 0)
        t_exit = min((current.bounds[i].end - ray_o[i]) / ray_d[i] for i in range(3) if ray_d[i] != 0)
        dist = t_exit - t_entry
        
        # Accumulate contribution
        if current.is_loaded:
            color = sample_voxel_data(ray_o, ray_d, current.data, t_entry, t_exit)
        else:
            color = current.average_color
        accumulate(color, dist)
        
        # Advance ray origin to brick exit
        ray_o = ray_o + ray_d * t_exit
```

**Parameters:**
*   **Conditional Block Size:** Limited to reduce divergent threads on GPU multiprocessors [S1, unofficial].
*   **Register Usage:** Limited to allow multiprocessors to handle more parallel threads, better covering texture access latency [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **GPU Memory Latency:** On NVIDIA G80 architecture, dynamically indexed arrays are placed in 'local' memory (device memory) by the shader compiler, which is uncached and up to 50x slower than registers [S1, unofficial].
*   **Dynamic Loading Failure:** If a brick fails to load dynamically (e.g., due to sudden load overload), the algorithm falls back to using the stored average color in the node [S1, unofficial].

## Operating characteristics
*   **Performance:** Approximately 50% faster than the mini-stack algorithm on the tested hardware (G80) [S1, unofficial].
*   **Strengths:** Eliminates stack memory overhead; reduces register pressure by avoiding stack storage; limits branch divergence through constrained conditional blocks [S1, unofficial].
*   **Limitations:** Recalculating DDA parameters at every step increases computational cost per node compared to incremental updates; performance is sensitive to GPU memory hierarchy (local memory penalty) [S1, unofficial].
*   **When to use:** When stack memory is constrained or when GPU register pressure is a bottleneck; suitable for hierarchical structures with parent pointers [S1, unofficial].
*   **When not to use:** On architectures where local memory latency is prohibitive and register usage cannot be sufficiently optimized [G].

## Background
Stackless Traversal was developed as part of research into interactive exploration of procedural volume data, specifically targeting N3-tree structures [S1, unofficial]. It was designed to optimize GPU rendering by minimizing stack usage and branch divergence, addressing performance bottlenecks observed in stack-based traversals on early GPU architectures like the G80 [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
