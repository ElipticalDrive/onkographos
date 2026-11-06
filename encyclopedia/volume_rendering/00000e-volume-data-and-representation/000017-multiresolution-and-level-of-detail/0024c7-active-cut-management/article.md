# Active Cut Management

Also called: Active Cut Management with Merge/Split Queues, Adaptive voxelization via cut adaptation, View-dependent hierarchical LOD for volume rendering

## Overview
Active Cut Management is a view-dependent level-of-detail technique for volume rendering that dynamically maintains a set of octree nodes, called the **active cut**, which exactly covers the model space without overlap [S1]. It solves the problem of balancing visual fidelity against rendering time and memory bandwidth by performing frame-by-frame merges and splits of octree nodes based on screen-space error metrics [S1]. Unlike static multiresolution approaches, it adapts the granularity of the volume representation in real-time to the viewer's position and the available hardware budget, ensuring texel sizes remain close to one pixel [S1].

## How it works
**Inputs:**
*   Current camera position and orientation.
*   Target frame rate (or frame time budget).
*   Available texture memory size.
*   Current active cut (set of nodes from the previous frame).
*   Octree structure with associated texture data (residing in RAM, disk, or texture memory).

**Outputs:**
*   Updated active cut for the current frame.
*   List of nodes to be rendered.
*   Texture load/unload commands for RAM/disk/texture memory.

**Data Structures:**
1.  **Active Cut:** A set of octree nodes $\{N_i\}$ such that their spatial extents partition the model space exactly once. Each node $N_i$ has a status: Leaf (Red), Non-Leaf with projected texel size $< 1$ pixel (Green), or Non-Leaf with projected texel size $> 1$ pixel (Yellow) [S1].
2.  **Merge Queue:** A priority queue containing the **parents** of all nodes currently on the active cut. Priority is determined by the error metric (see Formulas).
3.  **Split Queue:** A priority queue containing all nodes currently on the active cut. Priority is determined by the error metric.
4.  **Texture Cache:** An LRU cache in main memory and texture memory tracking the residency status of texture nodes [S1].

**Formulas:**
*   **Projected Texel Size ($E$):** The maximum screen-space size of a node's texels in pixel units. This serves as the error metric [S1].
    $$ E(N) = \max(\text{screen space size of texels in } N) $$
    Ideally, $E(N) \approx 1$ pixel [S1].
*   **Merge/Split Validity Condition:** A merge followed by a split is permitted only if it reduces the overall error:
    $$ E(\text{merge\_node}) < E(\text{split\_node}) $$
    This ensures that forcing a merge to enable a split is beneficial [S1].

**Parameters:**
*   **Frame Time Budget:** The total time allowed for rendering and cut adaptation. Limits the number of nodes rendered (fill rate) and the number of merges/splits (texture loads) [S1].
*   **Texture Memory Limit:** Maximum size of texture memory, restricting the total number of leaf nodes that can be resident [S1].

**Algorithm Steps:**

1.  **Initialize Queues:** Construct the Merge Queue and Split Queue based on the active cut from the previous frame. Sort both queues by error metric $E(N)$ [S1].
2.  **Adapt/Load Phase:**
    a.  **Check Constraints:** Determine the maximum number of splits and merges allowed in this frame based on:
        *   Frame time budget.
        *   Disk-to-RAM and RAM-to-Texture memory bandwidths.
        *   Texture memory capacity [S1].
    b.  **Process Split Queue:**
        i.  Select the node $N_{split}$ with the highest error from the Split Queue.
        ii. If $N_{split}$ is already in the process of being split (partial load), check if another node has higher error. If so, stall progress on $N_{split}$ and select the new highest-error node [S1].
        iii. If $N_{split}$ is not currently splitting:
            *   Check if texture memory limit is reached. If yes, check if a merge can be forced. A merge is forced only if $E(\text{merge\_node}) < E(N_{split})$ [S1].
            *   If limits are not reached or a merge is forced, begin loading $N_{split}$'s children from RAM/disk into texture memory. This operation may span multiple frames [S1].
            *   Update texture cache (LRU) [S1].
    c.  **Process Merge Queue:**
        i.  Select the parent node $N_{parent}$ with the highest priority from the Merge Queue.
        ii. If texture memory is full and a split is pending, a merge may be forced to free space, subject to the error reduction condition [S1].
        iii. If allowed, merge $N_{parent}$'s children into $N_{parent}$ (unload children from texture memory, load parent if necessary) [S1].
    d.  **Iterate:** Repeat steps (b) and (c) until the frame time budget is exhausted or no further beneficial merges/splits can be performed [S1].
3.  **Cull/Render Phase:**
    a.  Traverse the octree hierarchically starting from the root.
    b.  For each node, cull if:
        *   It is entirely outside the viewing frustum.
        *   Its scalar field value range lies entirely in a transfer function range with zero opacity [S1].
    c.  Render the nodes remaining on the active cut that were not culled [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def active_cut_management(previous_cut, camera, frame_budget, texture_memory_limit, octree):
    """
    Manages the active cut for a single frame.
    """
    # 1. Initialize Queues
    merge_queue = PriorityQueue()
    split_queue = PriorityQueue()
    
    for node in previous_cut:
        # Calculate error metric: max screen space size of texels in pixels
        error = calculate_screen_space_error(node, camera)
        
        if node.is_leaf():
            split_queue.push(node, priority=error)
        else:
            # Non-leaf nodes on cut are candidates for split if error is high
            split_queue.push(node, priority=error)
            
        # Parents of cut nodes are candidates for merge
        if node.parent is not None:
            merge_queue.push(node.parent, priority=error) # Priority based on parent's potential error reduction

    current_cut = set(previous_cut)
    texture_memory_usage = calculate_texture_memory_usage(current_cut)
    time_elapsed = 0
    
    # 2. Adapt/Load Phase
    while time_elapsed < frame_budget:
        # Determine best candidate for split
        best_split_node = split_queue.peek()
        if best_split_node is None:
            break
            
        # Check if we can perform a split
        can_split = True
        
        # Check texture memory limit
        if texture_memory_usage >= texture_memory_limit:
            # Must force a merge to free space
            best_merge_node = merge_queue.peek()
            if best_merge_node is None:
                can_split = False
            else:
                # Condition: merge error < split error
                if best_merge_node.error < best_split_node.error:
                    perform_merge(best_merge_node, current_cut, texture_memory_usage)
                    merge_queue.pop()
                    # After merge, re-evaluate split queue priorities and memory
                    update_queues(current_cut, merge_queue, split_queue)
                    texture_memory_usage = calculate_texture_memory_usage(current_cut)
                else:
                    can_split = False
        
        if can_split:
            # Check if node is already being split (partial load)
            if best_split_node.is_being_split():
                # Check if another node has higher error
                next_best = split_queue.peek_after(best_split_node)
                if next_best and next_best.error > best_split_node.error:
                    # Stall current split, start new one
                    best_split_node = next_best
                else:
                    # Continue current split
                    pass
            else:
                # Start new split
                if not best_split_node.is_resident_in_texture_memory():
                    load_node_from_ram_or_disk(best_split_node)
                    texture_memory_usage += best_split_node.child_texture_size
                
            # Perform split operation (may be partial)
            split_node(best_split_node, current_cut)
            update_queues(current_cut, merge_queue, split_queue)
            texture_memory_usage = calculate_texture_memory_usage(current_cut)
            
        # Check if we can perform a merge (if not already done above for split enablement)
        if not can_split:
            best_merge_node = merge_queue.peek()
            if best_merge_node is not None:
                # Merge reduces memory and potentially error
                perform_merge(best_merge_node, current_cut, texture_memory_usage)
                merge_queue.pop()
                update_queues(current_cut, merge_queue, split_queue)
                texture_memory_usage = calculate_texture_memory_usage(current_cut)
                
        time_elapsed += estimate_operation_time()

    # 3. Cull/Render Phase
    render_nodes = []
    for node in current_cut:
        if not is_outside_frustum(node, camera) and not is_zero_opacity(node):
            render_nodes.append(node)
            
    return render_nodes

def calculate_screen_space_error(node, camera):
    """
    Returns the maximum screen space size of a node's texels in pixel units.
    """
    # Implementation depends on camera projection and node bounding box
    # Not given in the sources. [G]
    pass

def perform_merge(node, current_cut, texture_memory_usage):
    """
    Merges a node's children into the node itself.
    Removes children from current_cut, adds node to current_cut.
    Unloads children textures, loads parent if needed.
    """
    # Remove children from cut
    for child in node.children:
        if child in current_cut:
            current_cut.remove(child)
            unload_texture(child)
            texture_memory_usage -= child.texture_size
            
    # Add parent to cut
    current_cut.add(node)
    if not node.is_resident_in_texture_memory():
        load_texture(node)
        texture_memory_usage += node.texture_size

def split_node(node, current_cut):
    """
    Splits a node into its children.
    Removes node from current_cut, adds children to current_cut.
    Loads children textures.
    """
    # Remove node from cut
    if node in current_cut:
        current_cut.remove(node)
        unload_texture(node)
        
    # Add children to cut
    for child in node.children:
        current_cut.add(child)
        if not child.is_resident_in_texture_memory():
            load_texture(child)
            texture_memory_usage += child.texture_size
```

**Edge Cases and Pitfalls:**
*   **Texture Cache Misses:** Splits and merges that cause cache misses in main memory or texture memory are more expensive. The LRU policy affects the cost of operations [S1].
*   **Interrupted Splits:** If a node is being split but another node has a higher error, the current split is stalled, and the new node begins splitting immediately [S1].
*   **Multi-frame Splits:** A single split operation may span multiple frames if it involves loading several texture nodes from RAM or disk. The system must track the state of partially loaded nodes [S1].
*   **Memory Limits:** If texture memory is full, a split can only proceed if a merge is forced, and only if the merge reduces overall error [S1].

## Operating characteristics
*   **Time Complexity:** The adaptation phase is bounded by the frame time budget. Each frame, a limited number of merges and splits are performed. The cost of each operation depends on texture cache hit/miss status [S1].
*   **Memory Complexity:** The number of nodes on the active cut is limited by texture memory size. The system dynamically adjusts the resolution to fit within the available texture memory [S1].
*   **Performance:** The method maintains texel sizes close to one pixel, optimizing visual quality and fill rate. It adapts to the viewer's position, providing higher resolution closer to the viewpoint [S1].
*   **Strengths:** View-dependent refinement, efficient use of texture memory, handles large datasets by streaming textures from disk/RAM [S1].
*   **Limitations:** Requires complex queue management and state tracking for partial splits. Performance is sensitive to texture cache misses and bandwidth limitations [S1].
*   **When to use:** When rendering large volumetric datasets with limited texture memory and variable frame rates, where view-dependent detail is critical [S1].
*   **When not to use:** When hardware lacks sufficient texture memory or bandwidth to support dynamic loading, or when a fixed resolution is acceptable [S1].

## Background
Active Cut Management was introduced by Leven et al. in the context of interactive visualization of unstructured grids using hierarchical 3D representations [S1]. The concept of the "active cut" is similar to view-dependent continuous level of detail techniques used for polygonal models, such as those by Xia and Varshney, Duchandeau et al., Hoppe, and Luebke and Erikson [S1]. The method was demonstrated on datasets like the Oxygen Post model (513,375 tetrahedra) and Cylinder data set (615,000 tetrahedra) [S1].

## Sources
[S1] PDF Interactive Visualization of Unstructured Grids Using Hierarchical 3D ..., https://web.eecs.umich.edu/~jjcorso/pubs/leven02unstructgrid.pdf, fetched 2026-10-27
