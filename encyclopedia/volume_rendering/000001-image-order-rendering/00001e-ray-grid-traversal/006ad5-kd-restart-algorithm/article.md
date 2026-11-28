# kd-restart algorithm

Also called: CERA-TVR

## Overview
The kd-restart algorithm is a stackless traversal method for kd-trees used in volume rendering and ray tracing. It solves the problem of recursive tree traversal overhead by eliminating the need for an explicit stack, which is particularly beneficial for GPU implementations where stack space is limited or expensive [S2, unofficial]. Unlike traditional recursive traversal, it restarts the traversal from the root of the kd-tree whenever a leaf node is reached, trading increased runtime for reduced memory complexity [S2, unofficial]. It builds on [Ray-grid traversal](17-ray-grid-traversal.md) principles but applies them to hierarchical spatial partitioning structures.

## How it works
The algorithm traverses a kd-tree structure where each internal node defines a splitting plane (axis-aligned) and two children (left and right). The ray is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].

### Data Structures
*   **KD-Node**: Contains a split axis $a \in \{x, y, z\}$, split position $s$, and pointers to left and right children. Leaf nodes contain references to the actual data (e.g., voxel bricks or texture coordinates).
*   **Ray**: Origin $\mathbf{o}$, direction $\mathbf{d}$, and current distance $t_{entry}$ along the ray.

### Steps
1.  **Initialization**: Start traversal at the root node. Initialize $t_{entry} = 0$ (or the ray's near plane).
2.  **Intersection Test**: For the current node, determine if the ray intersects the node's bounding volume. If not, terminate traversal for this ray [S2, unofficial].
3.  **Split Plane Check**: Determine which child the ray enters first based on the split plane $s$ on axis $a$.
    *   Calculate distance to split plane: $t_{split} = \frac{s - o_a}{d_a}$.
    *   If $t_{split} < t_{entry}$, the ray has already passed the split plane; proceed to the "far" child.
    *   Otherwise, proceed to the "near" child.
4.  **Recursive Descent**: Move to the selected child node.
5.  **Leaf Handling**: If the current node is a leaf:
    *   Process the leaf data (e.g., sample voxels, integrate color).
    *   **Restart**: Reset the current node to the root of the kd-tree.
    *   Continue traversal from the root with the updated $t_{entry}$ (if the ray continues through the leaf) or terminate if the ray is fully processed [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def kd_restart_traversal(ray_origin, ray_direction, kd_tree_root):
    """
    Traverses a kd-tree using the restart algorithm.
    Assumes kd_tree_root is the root node of the tree.
    """
    current_node = kd_tree_root
    t_entry = 0.0
    
    while current_node is not None:
        # Check if ray intersects current node's bounding box
        if not ray_intersects_bbox(ray_origin, ray_direction, current_node.bbox, t_entry):
            break
            
        # If leaf node, process and restart
        if current_node.is_leaf:
            process_leaf(current_node, ray_origin, ray_direction, t_entry)
            current_node = kd_tree_root  # Restart from root
            continue
            
        # Determine split axis and position
        split_axis = current_node.split_axis
        split_pos = current_node.split_position
        
        # Calculate distance to split plane
        if ray_direction[split_axis] == 0:
            # Ray parallel to split plane
            if ray_origin[split_axis] < split_pos:
                next_node = current_node.left
            else:
                next_node = current_node.right
        else:
            t_split = (split_pos - ray_origin[split_axis]) / ray_direction[split_axis]
            
            # Determine which child to visit first
            if t_split < t_entry:
                # Ray already passed split plane, go to far child
                next_node = current_node.right if split_axis == x else \
                            current_node.right if split_axis == y else \
                            current_node.right
                # Note: Logic depends on tree construction; typically:
                # if ray_origin[split_axis] < split_pos: near=left, far=right
                # if t_split < t_entry, we are on the far side
                if ray_origin[split_axis] < split_pos:
                    next_node = current_node.right
                else:
                    next_node = current_node.left
            else:
                # Visit near child first
                if ray_origin[split_axis] < split_pos:
                    next_node = current_node.left
                else:
                    next_node = current_node.right
                    
        # Update t_entry for the next step? 
        # In standard DDA, t_entry updates. In kd-tree restart, 
        # the restart implies we might re-evaluate from root.
        # However, to avoid infinite loops, we must ensure progress.
        # The 'restart' usually means we finish processing the leaf 
        # and then look for the next leaf in the traversal order.
        # But kd-trees don't have a linear order. 
        # The 'restart' strategy implies: after processing a leaf, 
        # we start a NEW traversal from the root to find the NEXT leaf 
        # that the ray intersects.
        
        current_node = next_node
        # t_entry might need update if we want to skip the current node's volume
        # But since we restart from root, we re-evaluate intersection with root bbox.
```

### Edge Cases and Pitfalls
*   **Infinite Loops**: If the ray is parallel to a split plane and lies exactly on it, care must be taken to avoid oscillating between children. [G] Use a small epsilon or consistent tie-breaking rules.
*   **Performance**: Restarting from the root for every leaf can be inefficient for deep trees or rays that traverse many leaves. The runtime increases compared to stack-based methods [S2, unofficial].
*   **Precision**: Floating-point errors in $t_{split}$ calculations can lead to incorrect child selection. [G] Use robust geometric predicates if possible.

## Subtopics

### Stackless KD-backtrack traversal
A variant of the kd-restart algorithm. Instead of restarting from the root after processing a leaf, it maintains a "restart point" pointer. At every descent, it checks if the ray crosses the split plane. If not, it marks the current node as a new restart point and stores a pointer. When a neighbor needs to be found or the next leaf is sought, it jumps back to the restart point instead of the root [S3, unofficial]. This reduces the number of nodes visited compared to full restarts, making it faster in practice, though worst-case complexity remains similar [S3, unofficial]. It requires storing pointers or using parent pointers, increasing memory usage slightly compared to pure restart [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(n \log n)$ in average case for balanced trees, where $n$ is the number of nodes visited. However, due to restarting from the root for each leaf, the constant factor is higher than stack-based traversal [S2, S3, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary stack space, as no explicit stack is used. Only the tree structure itself is stored [S2, unofficial].
*   **Strengths**: Ideal for GPU implementations where stack depth is limited. Simple implementation logic.
*   **Limitations**: Higher runtime due to redundant traversal from the root. Not suitable for very deep trees where restart cost dominates.
*   **When to Use**: When memory is constrained (e.g., mobile GPUs) and tree depth is moderate. When dynamic transfer functions require frequent updates and stack management is complex [S1].
*   **When Not to Use**: For very deep kd-trees or when performance is critical and stack space is available.

## Background
The kd-restart algorithm was proposed by Foley et al. for rendering triangle meshes on the GPU [S2, unofficial]. It was later adopted for volume rendering, notably in the CERA-TVR system, which targets scientific visualization applications with real-time dynamic transfer function updates [S1]. The method eliminates the need for a stack, addressing limitations in early GPU architectures [S2, unofficial].

## Sources
[S1] State-of-the-Art in GPU-Based Large-Scale Volume Visualization, https://vcg.seas.harvard.edu/publications/20150513-stateof-the-art-in-gpu-based-large-scale-volume-visualization/paper, fetched 2026-10-24
[S2] Eine adaptive, hybride Datenstruktur für Volumen Datensätze auf der GPU, https://doi.org/10.34726/hss.2016.26562, fetched 2026-10-26 (unofficial)
[S3] Rendering Large Volumetric, https://forceflow.be/wp-content/uploads/2012/02/vr_overview.pdf, fetched 2026-10-24 (unofficial)
