# Adaptive Gaussian Merging and Retrieval

Also called: Adaptive Merging, Screen-size based Merging

## Overview
Adaptive Gaussian Merging and Retrieval is a dynamic memory management and Level of Detail (LoD) strategy for 3D Gaussian Splatting that maintains a hierarchical structure based on primitive screen size to handle large scenes with limited GPU memory [S1]. It solves the problem of VRAM exhaustion during online optimization by merging sub-pixel Gaussians into coarser parents and offloading them to CPU RAM, while retrieving finer details when the camera zooms in [S1]. Unlike static hierarchies, this method is view-adaptive, triggering merges and retrievals based on the current viewpoint's projection of the Gaussians [S1].

## How it works

### Inputs and Outputs
*   **Inputs:** A set of 3D Gaussian primitives $G_i$ organized in a hierarchy; the current camera viewpoint (projection matrix and intrinsics); a merging threshold $\tau_l$; a retrieval threshold $\tau_h$.
*   **Outputs:** An updated active set of Gaussian primitives (leaves and intermediate nodes) optimized for rendering; updated parent-child links in the hierarchy.

### Data Structures
The hierarchy consists of nodes. Each node stores:
1.  **Gaussian Parameters:** Position $\mathbf{p}$, opacity $\alpha$, rotation $\mathbf{q}$ (quaternion), scale $\mathbf{s}$, and Spherical Harmonics (SH) coefficients [S1].
2.  **Topology:** IDs of parent node(s) and child node(s) [S1].
3.  **Status:** Flags indicating if the node is currently in the active set (on GPU) or offloaded (on CPU) [S1].

### Parameters
*   $\tau_l = 0.5$ pixels: The screen size threshold below which a Gaussian is considered sub-pixel and eligible for merging [S1].
*   $\tau_h = 1$ pixel: The screen size threshold above which a merged node triggers retrieval of its finer children [S1].
*   Merge Trigger Threshold: Merging is only triggered if more than 20% of the Gaussians satisfy $S_i < \tau_l$ [S1].

### Algorithm Steps

1.  **Compute Screen Sizes:** For every Gaussian $G_i$ in the current active set, compute its screen size $S_i$ from the current viewpoint. This involves projecting the Gaussian's covariance ellipse onto the image plane and measuring its extent in pixels [S1].
2.  **Check Merge Condition:** Count the number of Gaussians where $S_i < \tau_l$. If this count exceeds 20% of the total active Gaussians, proceed to merge; otherwise, skip merging for this frame [S1].
3.  **Select Candidates:** Identify the set of Gaussians $C_{merge} = \{G_i \mid S_i < \tau_l\}$ [S1].
4.  **Grouping via KNN:** For each candidate $G_i \in C_{merge}$, find the 4 closest Gaussians using a fast K-Nearest Neighbors (KNN) search [S1].
5.  **Deduplication:** Remove duplicate entries in the grouping list to ensure a Gaussian primitive does not have multiple parents in the same merge operation [S1].
6.  **Merge Operation:** Perform merging on groups of 4 Gaussians using KL-divergence based merging [S1]. This creates a new parent Gaussian $G_{parent}$ that approximates the children [S1].
7.  **Update Hierarchy:**
    *   Remove the sub-pixel children from the active set.
    *   Add $G_{parent}$ to the active set.
    *   Establish child-parent links between $G_{parent}$ and its children [S1].
    *   **Optimization:** If a Gaussian already has a parent, check if *all* its children are being merged. If so, "pull" the existing parent from the hierarchy (promote it to active) instead of creating a new node [S1].
8.  **Offloading:** Move small footprint leaf Gaussian primitives that are not in the active set to CPU RAM to free VRAM [S1].
9.  **Retrieval (Navigation):** When navigating, for any merged node $G_{parent}$ in the active set, compute its screen size $S_{parent}$. If $S_{parent} > \tau_h$:
    *   Reload the finer child Gaussians from CPU RAM to GPU.
    *   Update the coarse parent nodes with the new children's parameters.
    *   Remove $G_{parent}$ from the active set.
    *   Add the children to the active set [S1].
10. **Loop Closure Handling:** When loop closure is detected, move all Immediate 3D Gaussian Splat Reconstruction of Unordered Input with Global Consistency nodes (hierarchy nodes following the transform of the keyframe most used to spawn its children) asynchronously to avoid latency [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_merge_and_retrieve(active_gaussians, camera_view, tau_l=0.5, tau_h=1.0, merge_threshold=0.2):
    """
    active_gaussians: List of Gaussian objects with fields: id, params, parent_id, children_ids, is_offloaded
    camera_view: Object with project() method to compute screen size
    """
    
    # Step 1: Compute screen sizes
    for g in active_gaussians:
        g.screen_size = compute_screen_size(g.params, camera_view)
        
    # Step 2: Check merge condition
    subpixel_gaussians = [g for g in active_gaussians if g.screen_size < tau_l]
    if len(subpixel_gaussians) <= len(active_gaussians) * merge_threshold:
        # Not enough sub-pixel Gaussians to trigger merge
        return active_gaussians

    # Step 3 & 4: Grouping via KNN
    # Note: KNN search is performed on the spatial coordinates of subpixel_gaussians
    merge_groups = []
    for g in subpixel_gaussians:
        neighbors = fast_knn_search(g, subpixel_gaussians, k=4)
        merge_groups.append([g] + neighbors)
        
    # Step 5: Deduplication
    # Ensure no Gaussian has multiple parents in the same batch
    unique_groups = remove_duplicates(merge_groups)
    
    # Step 6 & 7: Merge and Update Hierarchy
    new_active_set = []
    nodes_to_remove = set()
    nodes_to_add = []
    
    for group in unique_groups:
        # Step 6: Merge using KL-divergence
        parent_gaussian = kl_divergence_merge(group)
        
        # Step 7: Update links
        for child in group:
            nodes_to_remove.add(child.id)
            child.parent_id = parent_gaussian.id
            if parent_gaussian.id not in [n.id for n in nodes_to_add]:
                nodes_to_add.append(parent_gaussian)
                
        # Optimization: Check if existing parent should be pulled
        # If a Gaussian already has a parent and all its children are in nodes_to_remove,
        # promote the existing parent instead of adding a new one.
        # (Implementation detail: Check existing parents of children in group)
        
    # Update active set
    for g in active_gaussians:
        if g.id not in nodes_to_remove:
            new_active_set.append(g)
            
    new_active_set.extend(nodes_to_add)
    
    # Step 8: Offloading
    # Gaussians not in new_active_set and marked as leaf/small are offloaded to CPU
    for g in active_gaussians:
        if g.id not in [ng.id for ng in new_active_set]:
            g.is_offloaded = True
            
    return new_active_set

def retrieve_if_zoomed(active_gaussians, camera_view, tau_h=1.0):
    """
    Called during navigation to retrieve finer details if zoomed in.
    """
    new_active_set = []
    nodes_to_reload = []
    
    for g in active_gaussians:
        if g.screen_size > tau_h and g.has_children():
            # Step 9: Retrieve
            children = load_children_from_cpu(g.children_ids)
            nodes_to_reload.extend(children)
            # Remove parent from active set
            pass 
        else:
            new_active_set.append(g)
            
    new_active_set.extend(nodes_to_reload)
    return new_active_set
```

## Operating characteristics

*   **Time Complexity:** The merge operation involves a KNN search, which is typically $O(N \log N)$ or $O(N)$ with approximate methods, where $N$ is the number of sub-pixel Gaussians [S1]. The merge itself is constant time per group of 4 [S1]. Retrieval is proportional to the number of nodes exceeding $\tau_h$ [S1].
*   **Memory Complexity:** Reduces GPU VRAM usage by offloading sub-pixel Gaussians to CPU RAM [S1]. The hierarchy structure adds minimal overhead (parent/child IDs) [S1].
*   **Performance:** Enables handling of large scenes with limited GPU memory [S1]. The 20% trigger threshold prevents frequent merging overhead [S1]. Double buffering is used to avoid stutter during retrieval [S1].
*   **Strengths:** Dynamic adaptation to view; efficient memory usage; fast hierarchy updates compared to static methods [S1].
*   **Limitations:** Requires careful tuning of $\tau_l$ and $\tau_h$ [S1]. KNN search can be costly if not optimized [S1]. Asynchronous updates for loop closure are necessary to avoid latency [S1].

## Background

*   **Origin:** Proposed in "Immediate 3D Gaussian Splat Reconstruction of Unordered Input with Global Consistency" (2024) [S1].
*   **Context:** Part of the broader effort to scale 3D Gaussian Splatting to large-scale scenes (e.g., cities) where memory constraints are a bottleneck [S1]. It builds on the concept of hierarchical 3D Gaussian representations [S1].
*   **Usage:** Used in online/progressive reconstruction systems where keyframes are added incrementally [S1].

## Sources

[S1] Immediate 3D Gaussian Splat Reconstruction of Unordered Input with Global Consistency, https://doi.org/10.1145/3799902.3811167, fetched 2026-10-26
