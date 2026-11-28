# SVO Ray Marching with Stack-Based Navigation

Also called: SVO Ray March, SVO Ray Marcher, Sparse Voxel Octree Ray Tracing

## Overview
SVO Ray Marching with Stack-Based Navigation is a ray traversal algorithm for Sparse Voxel Octrees (SVO) that uses an explicit stack to manage traversal state, enabling efficient navigation through the hierarchical structure to find voxel intersections [S1, unofficial]. It differs from parent methods by decoupling the traversal logic from the ray's parametric distance, using a fixed-point-like scale representation to step through the tree levels, and explicitly managing node validity and corner positions via a stack of `node_info_t` elements [S1, unofficial]. This approach allows for robust handling of empty space and early termination based on pixel error thresholds [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- `raypos`: The origin of the ray ($\mathbf{p}$) [S1, unofficial].
- `raydir`: The direction of the ray ($\mathbf{d}$), normalized [S1, unofficial].
- `svo_tree`: The Sparse Voxel Octree data structure [S1, unofficial].
- `minraylength2`: A minimum ray length constant added to the calculated distance [S1, unofficial].
- `rayScale2`: A parameter related to ray step size or resolution, used for pixel error calculation [S1, unofficial].

**Outputs:**
- `*out_t`: The distance to the hit voxel along the ray [S1, unofficial].
- `*out_normal`: The normal of the hit voxel face [S1, unofficial].
- `bool`: True if a hit was found, false otherwise [S1, unofficial].

### Data Structures
The algorithm relies on a stack to store traversal state [S1, unofficial].

**`node_info_t`**
- `parent`: Index or pointer to the parent node in the SVO [S1, unofficial].
- `corner`: A 3-bit integer (or equivalent) representing the corner of the current node within its parent [S1, unofficial].
- `lower`: A 3D vector ($\mathbb{R}^3$) representing the absolute lower bound of the current node in world space [S1, unofficial].

**Stack**
- A dynamic array or linked list of `node_info_t` elements [S1, unofficial].
- Operations: `push`, `pop`, `back` (access top element), `size` (get current depth) [S1, unofficial].

### Parameters
- `rayScale2`: Used in `svo_voxelpixelerror` to determine the threshold for pixel error [S1, unofficial].
- `minraylength2`: A minimum ray length constant added to the calculated distance [S1, unofficial].

### Algorithm Steps
1. **Initialization**:
   - Initialize an empty stack `stack` [S1, unofficial].
   - Determine the root node of the SVO [S1, unofficial].
   - Calculate the initial `lower` bound for the root node (typically the origin of the SVO space) [S1, unofficial].
   - Push the root node onto the stack [S1, unofficial].
   - Set `rayisoutside = false` [S1, unofficial].

2. **Traversal Loop**:
   - While `stack.size > 0` [S1, unofficial]:
     - Let `level = stack.size` [S1, unofficial].
     - Calculate `scale = 1.0 / (1 << level)` [S1, unofficial].
     - Let `current = stack.back()` [S1, unofficial].
     - Determine node validity:
       - Check if the node is null using `svo_tree_is_null` [S1, unofficial].
       - Check if the node is a leaf using `svo_tree_isleaf_f3` [S1, unofficial].
       - Check if the node has children using `svo_tree_has_children` [S1, unofficial].
       - Check if the node contains a voxel using `svo_tree_voxelexists` [S1, unofficial].
     - If `rayisoutside` is true:
       - Pop the stack: `svo_stack_pop(&stack)` [S1, unofficial].
       - If the stack is empty, break and return false [S1, unofficial].
       - Retrieve the new `current` from `stack.back()` [S1, unofficial].
       - Adjust `lower`: `lower -= corner_increment0 * scale`, where `corner_increment0` is the unit vector corresponding to the popped node's corner [S1, unofficial].
       - Attempt to move to the next corner in the `outface` direction: `next_corner1 = move_corner(current.corner, outface)` [S1, unofficial].
       - If `next_corner1` is valid:
         - Set `rayisoutside = false` [S1, unofficial].
         - Update `current.parent` and `current.corner` to the popped node's parent and `next_corner1` [S1, unofficial].
         - Adjust `lower`: `lower -= corner_increment0 * scale; lower += corner_increment1 * scale`, where `corner_increment1` corresponds to `next_corner1` [S1, unofficial].
         - Continue to the next iteration [S1, unofficial].
       - If `next_corner1` is invalid (boundary of parent):
         - Set `rayisoutside = true` [S1, unofficial].
         - Continue to the next iteration (loop will pop again) [S1, unofficial].
     - If the current node has a voxel (`svo_tree_voxelexists`) and the ray intersects the voxel data (`svo_intersects_voxel_data_f3`):
       - Calculate direction bounds: `dir_bounds = svo_calculate_dir_bounds_f3(lower, lower + scale, raydir)` [S1, unofficial].
       - Check for pixel error: `error = svo_voxelpixelerror(sdistancef3(raypos, dir_lower), rayScale2, level)` [S1, unofficial].
       - If `error < threshold`:
         - Record hit distance: `*out_t = minraylength2 + sdistancef3(raypos, dir_lower)` [S1, unofficial].
         - Compute normal from the opposite face of `outface` [S1, unofficial].
         - Return true [S1, unofficial].
     - If the node is a leaf (`svo_tree_isleaf_f3`):
       - Record hit distance and normal [S1, unofficial].
       - Return true [S1, unofficial].
     - If the node has children (`svo_tree_has_children`):
       - Push the current node onto the stack: `svo_stack_push(&stack, &current)` [S1, unofficial].
       - Calculate the next corner for the child node: `next_corner = svo_push_calculate_next_corner_f3(cube_normalized_dir, dir_lower, scale, t1)` [S1, unofficial].
       - Update `lower` for the child node: `next_lower = lower + (corner_increment * child_scale)`, where `child_scale = 1.0 / (1 << (level + 1))` [S1, unofficial].
       - Get the child node descriptor: `current = svo_tree_get_child(address_space, current, next_corner)` [S1, unofficial].
       - Update `lower = next_lower` [S1, unofficial].
       - Continue to the next iteration [S1, unofficial].

3. **Termination**:
   - If the stack is empty and no hit was found, return false [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def svo_ray_march_stack_based(raypos, raydir, svo_tree, minraylength2, rayScale2):
    stack = []
    root_node = svo_tree.get_root()
    root_lower = svo_tree.get_lower_bound()
    current = NodeInfo(parent=None, corner=0, lower=root_lower)
    stack.append(current)
    rayisoutside = False
    
    while len(stack) > 0:
        level = len(stack)
        scale = 1.0 / (1 << level)
        current = stack[-1]
        
        is_null = svo_tree_is_null(svo_tree, current)
        is_leaf = svo_tree_isleaf_f3(svo_tree, current)
        has_children = svo_tree_has_children(svo_tree, current)
        has_voxel = svo_tree_voxelexists(svo_tree, current)
        
        if rayisoutside:
            stack.pop()
            if len(stack) == 0:
                return False, None, None
            
            current = stack[-1]
            corner_increment0 = get_corner_unit_vector(current.corner)
            current.lower -= corner_increment0 * scale
            
            outface = get_outface(raydir, current.corner)
            next_corner1 = move_corner(current.corner, outface)
            
            if next_corner1 is not None:
                rayisoutside = False
                current.corner = next_corner1
                corner_increment1 = get_corner_unit_vector(next_corner1)
                current.lower -= corner_increment0 * scale
                current.lower += corner_increment1 * scale
                continue
            else:
                rayisoutside = True
                continue
                
        if has_voxel and svo_intersects_voxel_data_f3(svo_tree, current):
            dir_bounds = svo_calculate_dir_bounds_f3(current.lower, current.lower + scale, raydir)
            error = svo_voxelpixelerror(sdistancef3(raypos, dir_bounds.dir_lower), rayScale2, level)
            if error < threshold:
                out_t = minraylength2 + sdistancef3(raypos, dir_bounds.dir_lower)
                out_normal = compute_normal_from_face(opposite_face(outface))
                return True, out_t, out_normal
                
        if is_leaf:
            out_t = minraylength2 + sdistancef3(raypos, dir_bounds.dir_lower)
            out_normal = compute_normal_from_face(opposite_face(outface))
            return True, out_t, out_normal
            
        if has_children:
            stack.append(current)
            next_corner = svo_push_calculate_next_corner_f3(cube_normalized_dir, dir_bounds.dir_lower, scale, t1)
            child_scale = 1.0 / (1 << (level + 1))
            next_lower = current.lower + (corner_increment * child_scale)
            current = svo_tree_get_child(address_space, current, next_corner)
            current.lower = next_lower
            continue
            
    return False, None, None
```

## Operating characteristics

## Background

## Sources
[S1] realazthat/landscapes, https://github.com/realazthat/landscapes, fetched 2026-10-26 (unofficial)
