# DDA Tree Traversal with Mini-Stack

Also called: Mini-stack traversal, Stack-based hierarchical traversal

## Overview
This method extends the Digital Differential Analyzer (DDA) grid traversal algorithm to N-ary hierarchical trees for GPU shader implementation [S1, unofficial]. It solves the problem of efficiently traversing sparse voxel structures by maintaining a small, fixed-size stack of DDA states, allowing the ray to descend into child nodes and ascend back to parent nodes without dynamic memory allocation [S1, unofficial]. It is distinguished from standard DDA by its hierarchical boundary handling and from other tree traversals by its reliance on a statically allocated mini-stack indexed via Shader Model 4.0 features [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, hierarchical tree structure with N-ary nodes, maximum tree depth $D_{max}$, and node accessors [G].
*   **Output:** Sequence of leaf voxels visited by the ray, along with their entry and exit distances [G].

**Data Structures:**
*   **Mini-Stack:** A fixed-size array of length $D_{max}$ (compile-time constant) [S1, unofficial]. Each entry at index `level` stores:
    *   `t_max[3]`: Distance to the next boundary for each axis ($x, y, z$) within the current node's local grid.
    *   `t_delta[3]`: Distance increment per voxel step for each axis.
    *   `step[3]`: Direction of index increment ($+1$ or $-1$) for each axis.
    *   `current_index[3]`: Current voxel coordinates $(i, j, k)$ within the current node's local grid.
    *   `node_addr`: Memory address or index of the node in the indirection pool.
*   **Stack Pointer:** An integer variable tracking the current depth in the stack, initially 0 [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Compute DDA parameters for the root node's bounding box. Push the root's DDA state onto the stack at index 0. Set `stack_ptr = 0` [G].
2.  **Descent Phase:** While the current node is not a leaf, identify the child node intersected by the ray using the current DDA state. Compute DDA parameters for the child node's local grid (the entry distance into the child equals the exit distance from the parent's current cell). Update the stack entry at `stack_ptr` with the new DDA parameters and the child's node address. Increment `stack_ptr`. Repeat until a leaf node is reached [S1, unofficial].
3.  **Horizontal Advance:** Within the current node (leaf or internal), advance the DDA: find the axis $k$ with the minimum `t_max[k]`. Record the voxel at `current_index` and the interval $[t_{entry}, t_{max}[k]]$. Advance `current_index[k]` by `step[k]`. Update `t_max[k] += t_delta[k]`. Update `t_entry = t_max[k]`. If the new `current_index` is within the node's bounds, continue processing the next voxel. If outside, proceed to Vertical Ascent [S1, unofficial].
4.  **Vertical Ascent:** Decrement `stack_ptr`. If `stack_ptr < 0`, the ray has exited the entire tree; terminate. Retrieve the DDA state from the stack at the new `stack_ptr`. Resume the Horizontal Advance loop from the point where the descent occurred, using the stored DDA parameters [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dda_tree_traversal_mini_stack(ray_origin, ray_direction, root_node, max_depth, get_node, is_leaf, get_bounds, voxel_size=1.0):
    # Initialize DDA parameters for a bounding box [G]
    def init_dda(origin, direction, bounds):
        step = [0, 0, 0]
        t_max = [0.0, 0.0, 0.0]
        t_delta = [0.0, 0.0, 0.0]
        current_index = [0, 0, 0]
        for i in range(3):
            if direction[i] > 0:
                step[i] = 1
                current_index[i] = int((origin[i] - bounds.min[i]) / voxel_size)
                next_boundary = (current_index[i] + 1) * voxel_size + bounds.min[i]
                t_max[i] = (next_boundary - origin[i]) / direction[i]
                t_delta[i] = voxel_size / direction[i]
            elif direction[i] < 0:
                step[i] = -1
                current_index[i] = int((origin[i] - bounds.min[i]) / voxel_size)
                next_boundary = current_index[i] * voxel_size + bounds.min[i]
                t_max[i] = (next_boundary - origin[i]) / direction[i]
                t_delta[i] = -voxel_size / direction[i]
            else:
                step[i] = 0
                t_max[i] = float('inf')
                t_delta[i] = 0.0
        return step, t_max, t_delta, current_index

    stack = [None] * max_depth
    stack_ptr = 0

    # Initialization: root node
    bounds = get_bounds(root_node)
    step, t_max, t_delta, current_index = init_dda(ray_origin, ray_direction, bounds)
    stack[0] = {'step': step, 't_max': t_max, 't_delta': t_delta, 'current_index': current_index, 'node_addr': root_node.address}
    t_entry = 0.0

    while True:
        state = stack[stack_ptr]
        node = get_node(state['node_addr'])
        bounds = get_bounds(node)

        if is_leaf(node):
            # Horizontal Advance for leaf
            while True:
                min_idx = 0
                min_val = state['t_max'][0]
                for i in range(1, 3):
                    if state['t_max'][i] < min_val:
                        min_val = state['t_max'][i]
                        min_idx = i

                t_exit = min_val
                yield state['current_index'], t_entry, t_exit

                state['current_index'][min_idx] += state['step'][min_idx]
                state['t_max'][min_idx] += state['t_delta'][min_idx]
                t_entry = t_exit

                # Check bounds
                if not (bounds.min[0] <= state['current_index'][0] <= bounds.max[0] and
                        bounds.min[1] <= state['current_index'][1] <= bounds.max[1] and
                        bounds.min[2] <= state['current_index'][2] <= bounds.max[2]):
                    break
        else:
            # Internal node: find child to descend
            min_idx = 0
            min_val = state['t_max'][0]
            for i in range(1, 3):
                if state['t_max'][i] < min_val:
                    min_val = state['t_max'][i]
                    min_idx = i

            t_exit = min_val
            # Child indexing depends on tree layout (e.g., octree Morton order) [G]
            child_node = get_child(node, state['current_index'])

            if child_node is None:
                pass # No child in this direction, proceed to ascent
            else:
                # Compute DDA for child node
                child_bounds = get_bounds(child_node)
                child_origin = [ray_origin[i] + t_exit * ray_direction[i] for i in range(3)]
                child_step, child_t_max, child_t_delta, child_current_index = init_dda(child_origin, ray_direction, child_bounds)

                stack_ptr += 1
                stack[stack_ptr] = {'step': child_step, 't_max': child_t_max, 't_delta': child_t_delta, 'current_index': child_current_index, 'node_addr': child_node.address}
                t_entry = t_exit
                continue

        # Vertical Ascent
        stack_ptr -= 1
        if stack_ptr < 0:
            break
```

**Parameters:**
*   **Stack Size:** Fixed at compile time, equal to the maximum depth of the hierarchical structure [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Direction Components:** If a ray direction component is zero, the corresponding `t_delta` is set to $0.0$ and `t_max` to $\infty$, ensuring the ray never advances in that axis [G].
*   **Precision Errors:** Floating-point precision issues may cause the ray to miss small voxels or re-enter previously visited cells. Using epsilon offsets or robust geometric predicates is recommended [G].
*   **Empty Nodes:** If a node has no children in the ray's path, the algorithm must correctly ascend without attempting to access non-existent child nodes [S1, unofficial].

## Subtopics

### [Optimized Octree DDA Traversal for SSVDAGs](5259-optimized-octree-dda-traversal-for-ssvdags.md)
This method extends the DDA Tree Traversal with Mini-Stack to support Symmetry-aware Sparse Voxel DAGs (SSVDAGs), a compression format that exploits geometric symmetries to reduce memory footprint. It solves the problem of correctly traversing a DAG where child nodes and leaf voxels may be mirrored relative to their parent's coordinate system. The key distinction from standard octree DDA is the maintenance of a 3-bit `mirror_mask` that tracks reflection transformations, allowing the algorithm to map logical voxel indices to physical storage locations without decompressing the geometry.

## Operating characteristics
*   **Time Complexity:** Constant per traversal step, with total time proportional to the number of visited cells plus the number of hierarchical transitions [G].
*   **Memory Complexity:** $O(D)$, where $D$ is the maximum tree depth, due to the fixed-size mini-stack [S1, unofficial].
*   **Performance:** Highly efficient on GPUs due to the use of statically allocated arrays and minimal branching [S1, unofficial]. The fixed stack size avoids dynamic memory allocation overhead [S1, unofficial].
*   **Strengths:** Fast traversal, low memory overhead, suitable for real-time rendering on GPUs [S1, unofficial].
*   **Limitations:** Requires compile-time knowledge of maximum tree depth; less flexible for dynamically changing tree structures [S1, unofficial].

## Background
The method was proposed for the interactive exploration of procedural volumes [S1, unofficial]. It extends the classic DDA algorithm to N-ary trees, leveraging Shader Model 4.0 features for dynamic indexing of local arrays [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
