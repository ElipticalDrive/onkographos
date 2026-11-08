# Hero-like Octree Traversal

Also called: Depth-First Octree Traversal, HERO, HERO Algorithm, Hardware Enhancer for Ray-tracing Octrees

## Overview
Hero-like Octree Traversal is a depth-first search (DFS) algorithm for ray-tracing octrees that determines the order in which child voxels are penetrated by a ray using a 2x2x2 voxel test [S1, S2, unofficial]. It generates child addresses via arithmetic and logical operations, allowing the traversal to skip hidden objects by yielding the first terminal voxel hit [S2, unofficial]. It differs from its siblings by explicitly computing the traversal order of children at each node using bitmask logic (VMASK, CHILDMASK) and maintaining state on a stack to resume traversal, rather than relying on neighbor-finding or bottom-up approaches [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Ray defined by origin $\underline{\mathbf{E}}$ and direction $\underline{\mathbf{v}} = (x_v, y_v, z_v)$.
*   Octree structure where each non-terminal node contains pointers to 8 children and mid-plane coordinates $(x_{mid}, y_{mid}, z_{mid})$. Terminal nodes contain object lists.

**Outputs:**
*   The first terminal voxel (or object within it) intersected by the ray.
*   Optionally, the face struck (SLMASK) for secondary ray generation.

### Data Structures
**Octree Node:**
*   `is_terminal`: Boolean flag.
*   If `is_terminal`: Pointer to list of objects.
*   If not `is_terminal`: Array of 8 pointers to child nodes (index $n$ corresponds to child $n$).
*   `x_mid`, `y_mid`, `z_mid`: Coordinates of the mid-planes defining the children.

**Stack Node (per active parent node):**
*   `node_ptr`: Pointer to the current octree node.
*   `s_lmax`: Maximum lower limit (entry distance into current node context).
*   `s_umin`: Minimum upper limit (exit distance from current node context).
*   `s_xmid`, `s_ymid`, `s_zmid`: Precomputed distances to mid-planes.
*   `CHILDMASK`: 3-bit mask indicating which children are potentially intersected.
*   `LASTMASK`: 3-bit mask indicating the final set of children.
*   `MASKLIST[1..3]`: Array of 3-bit masks sorted by distance.
*   `MASKLIST_index`: Current index into `MASKLIST`.
*   `prev_s_umin_ptr`: Pointer to the previous child's `s_umin` (used as current child's `s_lmax`).
*   `SLMASK`: 3-bit mask indicating the face struck (for terminal nodes).

### Algorithm Steps

1.  **Precompute Ray Reciprocals:**
    Compute $r_x = 1/x_v$, $r_y = 1/y_v$, $r_z = 1/z_v$. Handle parallel rays by treating division by zero as infinity with correct sign [S2, unofficial].

2.  **Generate VMASK:**
    Create a 3-bit mask from the ray direction signs. Bit 0 corresponds to $z_v$, Bit 1 to $y_v$, Bit 2 to $x_v$.
    $$ \text{VMASK} = (\text{sign}(z_v) \ll 2) | (\text{sign}(y_v) \ll 1) | (\text{sign}(x_v) \ll 0) $$
    Where $\text{sign}(v) = 1$ if $v < 0$, else $0$ [S2, unofficial].

3.  **Initialize Root:**
    Compute intersection distances $s_i$ for the 6 bounding planes of the root node ($x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max}$):
    $$ s_i = -(c_i \cdot \underline{\mathbf{E}} + d_i) / (c_i \cdot \underline{\mathbf{v}}) $$
    Categorize each $s_i$ as lower or upper limit based on VMASK. If VMASK bit is 0, $s_{min}$ is lower, $s_{max}$ is upper. If 1, opposite.
    Compute $s_{lmax} = \max(\text{all lower limits})$ and $s_{umin} = \min(\text{all upper limits})$.
    If $s_{lmax} < s_{umin}$ and $s_{lmax} > 0$, the ray hits the root. Push root state to stack [S2, unofficial].

4.  **Traversal Loop (Depth-First):**
    While stack is not empty:
    a. Pop `stack_node`. Let `node` be `stack_node.node_ptr`.
    b. If `node` is terminal:
       i. Compute intersection with objects in `node`.
       ii. If hit, return hit.
       iii. If no hit, continue loop (pop next).
    c. If `node` is non-terminal:
       i. Compute mid-plane distances:
          $$ s_{xmid} = (x_{mid} - x_E) \cdot r_x $$
          $$ s_{ymid} = (y_{mid} - y_E) \cdot r_y $$
          $$ s_{zmid} = (z_{mid} - z_E) \cdot r_z $$
       ii. Sort $s_{xmid}, s_{ymid}, s_{zmid}$ in ascending order.
       iii. Assign masks: $001$ to $x_{mid}$, $010$ to $y_{mid}$, $100$ to $z_{mid}$. Store in `MASKLIST[1..3]` corresponding to sorted order [S2, unofficial].
       iv. Compute `CHILDMASK` and `LASTMASK`:
           $$ \text{CHILDMASK} = (s_{zmid} < s_{lmax}) \cdot 1 + (s_{ymid} < s_{lmax}) \cdot 2 + (s_{xmid} < s_{lmax}) \cdot 4 $$
           $$ \text{LASTMASK} = (s_{zmid} < s_{umin}) \cdot 1 + (s_{ymid} < s_{umin}) \cdot 2 + (s_{xmid} < s_{umin}) \cdot 4 $$
       v. Initialize `MASKLIST_index = 1`.
       vi. Compute first child ID: `child_id = CHILDMASK XOR VMASK`.
       vii. Check if `child_id` is valid (not NIL). If valid, push current node state back to stack with updated `CHILDMASK` and `MASKLIST_index`, and push the child node to stack with:
           - `s_lmax` = `stack_node.s_lmax` (for first child) or `prev_s_umin` (for subsequent).
           - `s_umin` = `stack_node.s_umin` (for last child) or relevant mid-plane distance (for intermediate).
           - Recursively process child.

5.  **Child Iteration Logic:**
    To find subsequent children of the same parent:
    i. While `(MASKLIST[MASKLIST_index] AND CHILDMASK) != 0`, increment `MASKLIST_index`.
    ii. If `MASKLIST_index > 3`, all children processed.
    iii. Else, `CHILDMASK = CHILDMASK OR MASKLIST[MASKLIST_index]`.
    iv. New `child_id = CHILDMASK XOR VMASK`.
    v. Update `s_lmax` for new child to be previous child's `s_umin`.
    vi. Update `s_umin` for new child based on the mask used (e.g., if mask `010` used, `s_umin = s_ymid`) [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hero_traverse(ray_origin, ray_dir, root_node):
    # Precompute reciprocals
    rx = 1.0 / ray_dir.x
    ry = 1.0 / ray_dir.y
    rz = 1.0 / ray_dir.z
    
    # Generate VMASK
    vmask = 0
    if ray_dir.z < 0: vmask |= 1
    if ray_dir.y < 0: vmask |= 2
    if ray_dir.x < 0: vmask |= 4
    
    # Stack element: (node, s_lmax, s_umin, s_xmid, s_ymid, s_zmid, child_mask, last_mask, masklist, masklist_idx, prev_s_umin)
    stack = []
    
    # Initialize Root
    s_min_x = (root_node.x_min - ray_origin.x) * rx
    s_max_x = (root_node.x_max - ray_origin.x) * rx
    s_min_y = (root_node.y_min - ray_origin.y) * ry
    s_max_y = (root_node.y_max - ray_origin.y) * ry
    s_min_z = (root_node.z_min - ray_origin.z) * rz
    s_max_z = (root_node.z_max - ray_origin.z) * rz
    
    # Categorize limits based on VMASK
    # If bit is 0: min is lower, max is upper. If 1: min is upper, max is lower.
    # Simplified logic for limits:
    lower_limits = []
    upper_limits = []
    
    # x planes
    if (vmask & 4) == 0: # x_v > 0
        lower_limits.append(s_min_x)
        upper_limits.append(s_max_x)
    else: # x_v < 0
        lower_limits.append(s_max_x)
        upper_limits.append(s_min_x)
        
    # y planes
    if (vmask & 2) == 0: # y_v > 0
        lower_limits.append(s_min_y)
        upper_limits.append(s_max_y)
    else: # y_v < 0
        lower_limits.append(s_max_y)
        upper_limits.append(s_min_y)
        
    # z planes
    if (vmask & 1) == 0: # z_v > 0
        lower_limits.append(s_min_z)
        upper_limits.append(s_max_z)
    else: # z_v < 0
        lower_limits.append(s_max_z)
        upper_limits.append(s_min_z)
        
    s_lmax = max(lower_limits)
    s_umin = min(upper_limits)
    
    if s_lmax < s_umin and s_lmax > 0:
        # Push root state
        # Mid-planes not needed for root, but structure requires them
        stack.append({
            'node': root_node,
            's_lmax': s_lmax,
            's_umin': s_umin,
            's_xmid': 0, 's_ymid': 0, 's_zmid': 0,
            'child_mask': 0, 'last_mask': 0,
            'masklist': [0, 0, 0],
            'masklist_idx': 1,
            'prev_s_umin': 0
        })
        
    while stack:
        state = stack.pop()
        node = state['node']
        
        if node.is_terminal:
            # Intersect objects in terminal node
            hit = intersect_objects(node.objects, ray_origin, ray_dir, state['s_lmax'], state['s_umin'])
            if hit:
                return hit
            continue
            
        # Non-terminal node
        # Compute mid-plane distances
        s_xmid = (node.x_mid - ray_origin.x) * rx
        s_ymid = (node.y_mid - ray_origin.y) * ry
        s_zmid = (node.z_mid - ray_origin.z) * rz
        
        # Sort mid-plane distances and assign masks
        # Masks: x=001, y=010, z=100
        mid_planes = [
            (s_xmid, 1),
            (s_ymid, 2),
            (s_zmid, 4)
        ]
        mid_planes.sort(key=lambda x: x[0])
        
        masklist = [0, 0, 0]
        for i, (dist, mask) in enumerate(mid_planes):
            masklist[i] = mask
            
        # Compute CHILDMASK and LASTMASK
        child_mask = 0
        if s_zmid < state['s_lmax']: child_mask |= 1
        if s_ymid < state['s_lmax']: child_mask |= 2
        if s_xmid < state['s_lmax']: child_mask |= 4
        
        last_mask = 0
        if s_zmid < state['s_umin']: last_mask |= 1
        if s_ymid < state['s_umin']: last_mask |= 2
        if s_xmid < state['s_umin']: last_mask |= 4
        
        # Find first child
        idx = 1
        current_child_mask = child_mask
        
        while idx <= 3:
            # Check if current masklist entry is already in child_mask
            if (masklist[idx-1] & current_child_mask) == 0:
                break
            idx += 1
            
        if idx > 3:
            # No more children
            continue
            
        # Update child_mask
        current_child_mask |= masklist[idx-1]
        
        # Calculate child ID
        child_id = current_child_mask ^ vmask
        
        # Determine s_lmax and s_umin for the child
        # First child uses parent's s_lmax
        # Subsequent children use previous child's s_umin
        # s_umin for child is determined by the mask used
        
        # For simplicity, assume we push the parent back and push the child
        # We need to track if this is the first child or subsequent
        # The algorithm description implies iterative generation
        
        # Push parent back with updated state for next iteration
        stack.append({
            'node': node,
            's_lmax': state['s_lmax'],
            's_umin': state['s_umin'],
            's_xmid': s_xmid, 's_ymid': s_ymid, 's_zmid': s_zmid,
            'child_mask': current_child_mask,
            'last_mask': last_mask,
            'masklist': masklist,
            'masklist_idx': idx + 1,
            'prev_s_umin': state['s_umin'] # Placeholder, actual logic requires tracking
        })
        
        # Determine child's s_umin based on mask
        mask_used = masklist[idx-1]
        child_s_umin = state['s_umin']
        if mask_used == 1: child_s_umin = s_zmid
        elif mask_used == 2: child_s_umin = s_ymid
        elif mask_used == 4: child_s_umin = s_xmid
        
        # Child's s_lmax is parent's s_lmax for first child, else prev_s_umin
        # Since we are pushing parent back, we need to know if this is the first child
        # The first child has s_lmax = state['s_lmax']
        # Subsequent children have s_lmax = previous child's s_umin
        
        # For the first child pushed:
        child_s_lmax = state['s_lmax']
        
        child_node = node.children[child_id]
        if child_node is not None:
            stack.append({
                'node': child_node,
                's_lmax': child_s_lmax,
                's_umin': child_s_umin,
                's_xmid': 0, 's_ymid': 0, 's_zmid': 0,
                'child_mask': 0, 'last_mask': 0,
                'masklist': [0, 0, 0],
                'masklist_idx': 1,
                'prev_s_umin': 0
            })
            
    return None
```

### Parameters
*   **VMASK**: 3-bit mask derived from ray direction signs [S2, unofficial].
*   **SLMASK**: 3-bit mask carried on stack for terminal voxels, indicating struck face (x, y, or z plane) [S2, unofficial].

### Edge Cases and Pitfalls
*   **Parallel Rays**: If ray is parallel to a bounding plane, the denominator in distance calculation is zero. Represent infinity by maximum numeric magnitude, ensuring correct sign evaluation [S2, unofficial].
*   **Floating Point Errors**: Mid-point interpolation ($s_{mid} = (s_{min} + s_{max})/2$) is unreliable. Use reciprocal multiplication [S2, unofficial].
*   **Numeric Overflow**: Handle carefully when ray is nearly parallel to planes [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Depends on tree depth and occupancy. Reduces steps compared to non-hierarchical traversal [S1, unofficial].
*   **Memory Complexity**: Stack depth proportional to tree depth.
*   **Strengths**: Rapid traversal, skips hidden objects, efficient bitwise operations [S2, unofficial].
*   **Limitations**: Complex state management on stack, sensitive to floating point precision [S2, unofficial].

## Background
Originated in "The HERO Algorithm for Ray-Tracing Octrees" by Mark Agate, Richard L. Grimsdale, and Paul F. Lister (1989) [S2, unofficial]. Designed for hardware implementation (HERO: Hardware Enhancer for Ray-tracing Octrees) to minimize floating point processing [S2, unofficial].

## Sources
[S1] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S2] PDF The HERO Algorithm for Ray-Tracing Octrees - EG, https://diglib.eg.org/bitstream/handle/10.2312/EGGH.EGGH89.061-073/061-073.pdf?sequence=1, fetched 2026-10-26 (unofficial)
