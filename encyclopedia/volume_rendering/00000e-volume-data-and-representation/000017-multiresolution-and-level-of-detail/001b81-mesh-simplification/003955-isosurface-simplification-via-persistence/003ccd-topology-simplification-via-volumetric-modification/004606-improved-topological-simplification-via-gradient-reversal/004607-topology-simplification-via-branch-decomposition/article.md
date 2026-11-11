# Topology Simplification via Branch Decomposition

Also called: Branch Decomposition, Hierarchical Contour Tree Simplification, Improved Topological Simplification for Volume Rendering

## Overview
Topology Simplification via Branch Decomposition is a method for storing and traversing a hierarchy of contour tree simplifications to enable topology-controlled volume rendering. It solves the problem of "oversegmentation" by concatenating sequences of contour tree arcs into single hierarchical units, allowing users to interact with a coarse simplified tree and propagate transfer functions to the full-resolution data. Unlike methods that treat all zones with equivalent nesting depth as identical, this approach unifies topological zones based on their hierarchical relationship, avoiding unnecessary segmentation boundaries where regions do not overlap in value range.

## How it works

### Inputs and Outputs
*   **Input:** A contour tree of a scalar field, where nodes represent critical points (minima, maxima, saddles) and edges represent monotone paths between them.
*   **Output:** A branch decomposition tree, where each node represents a "branch" defined by a saddle-extremum pair, and parent-child relationships indicate the hierarchical order of simplification.

### Data Structures
The branch decomposition is a rooted tree structure. Each node in this tree represents a **branch**.

**Branch Node Layout:**
*   `saddle`: The index or identifier of the saddle critical point.
*   `extremum`: The index or identifier of the extremum (minimum or maximum) critical point.
*   `priority`: A numeric value determining the simplification order.
*   `children`: A list of child branch nodes.
*   `parent`: A reference to the parent branch node (or null for the root).

**Root Branch Exception:**
Exactly one branch connects two leaves (the root branch). All other branches connect one leaf (extremum) to an interior node (saddle) of another branch.

### Algorithm Steps

1.  **Construct Initial Contour Tree:** Compute the contour tree of the scalar field. Identify all critical points and their connectivity. Exclude saddles corresponding to genus changes from the simplification scheme, as contours of different genus are distinguished by function value alone and do not affect segmentation complexity [S1][S2, unofficial].

2.  **Initialize Branch Tree:** Create an empty tree for the branch decomposition.

3.  **Iterative Pruning:**
    While the contour tree contains prunable saddle-extremum pairs:
    a.  **Select Pair:** Identify a saddle-extremum pair $(s, e)$ from the contour tree. The selection is determined by a **priority measure**.
    b.  **Calculate Priority:** Compute the priority for the pair.
        *   If using **Persistence**: $P(s, e) = |f(s) - f(e)|$, where $f$ is the scalar function value [S1][S2, unofficial].
        *   If using **Geometric Measures**: Compute volume or hypervolume (integral of the scalar field over the enclosed volume) [S1][S2, unofficial].
    c.  **Prune and Collapse:** Remove the pair $(s, e)$ from the contour tree. Collapse the remaining node (the saddle $s$) into an arc.
    d.  **Create Branch Node:** Create a new branch node $B$ with `saddle` = $s$, `extremum` = $e$, and `priority` = $P(s, e)$.
    e.  **Link Hierarchy:** Identify the arc created by the collapse as the parent. Link branch node $B$ as a child of this parent arc. If the parent arc is itself a branch from a previous step, link $B$ to that branch node. If the parent is a new arc not yet a branch, it will be linked in a subsequent step or treated as the root if it connects two leaves.
    f.  **Ordering:** Pairs with lower priority are pruned first. Consequently, they become children of branches with higher priority. The most important branches (highest priority) end up at the top of the rooted tree [S1][S2, unofficial].

4.  **Finalize:** The result is a rooted tree of branches. The root branch connects two leaves. All other branches connect an extremum to a saddle that is an interior node of a higher-priority branch.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_branch_decomposition(contour_tree, priority_function):
    """
    Computes the branch decomposition of a contour tree.
    
    Args:
        contour_tree: A graph structure representing the contour tree.
                      Nodes are critical points (min, max, saddle).
                      Edges connect critical points monotonically.
        priority_function: A function that takes (saddle, extremum) and returns a numeric priority.
                           Lower priority values are pruned first.
    
    Returns:
        root_branch: The root node of the branch decomposition tree.
    """
    # Copy contour tree to avoid modifying original
    current_tree = copy(contour_tree)
    
    # List to store created branch nodes
    branches = []
    
    # Map to track which contour tree nodes have been collapsed into arcs
    # An arc is represented by the saddle node that remains after pruning
    collapsed_arcs = {} 
    
    # While there are prunable pairs (saddle-extremum)
    while has_prunable_pairs(current_tree):
        # 1. Select pair with lowest priority
        saddle, extremum = select_lowest_priority_pair(current_tree, priority_function)
        priority = priority_function(saddle, extremum)
        
        # 2. Prune the pair from the contour tree
        # Removing the extremum and the edge between saddle and extremum
        remove_node_and_edges(current_tree, extremum)
        remove_edge(current_tree, saddle, extremum)
        
        # 3. Collapse the saddle into an arc
        # The saddle node remains in the tree, acting as the connection point
        # This arc is the "parent" for the current branch
        parent_arc_node = saddle
        
        # 4. Create the branch node
        branch_node = {
            'saddle': saddle,
            'extremum': extremum,
            'priority': priority,
            'children': [],
            'parent': None
        }
        branches.append(branch_node)
        
        # 5. Link the branch to its parent
        # The parent is the arc created by the collapse.
        # If the parent arc node was already part of a previous branch,
        # that branch is the parent. Otherwise, it might be a new arc or the root.
        
        # Check if the saddle node is already a child of an existing branch
        # (i.e., it was an extremum of a previous, higher-priority branch)
        parent_branch = find_branch_containing_node(branches, saddle)
        
        if parent_branch is not None:
            # The saddle is an interior node of an existing branch
            parent_branch['children'].append(branch_node)
            branch_node['parent'] = parent_branch
        else:
            # The saddle is not yet part of any branch.
            # It might be the root of the decomposition or an intermediate arc.
            # In a standard branch decomposition construction, the first pruned
            # pair's saddle becomes part of the root branch or a new arc.
            # For simplicity, if no parent branch exists, this branch is a candidate
            # for the root or will be linked when its saddle becomes a child of another branch.
            # [G] Implementation detail: Typically, the last remaining structure forms the root.
            # We mark this branch as potentially root.
            branch_node['is_potential_root'] = True
            
    # 6. Identify the root branch
    # The root branch is the one that connects two leaves and has no parent branch.
    root_branch = find_root_branch(branches)
    
    return root_branch

def select_lowest_priority_pair(tree, priority_func):
    """
    Finds the saddle-extremum pair with the minimum priority value.
    """
    min_priority = infinity
    best_pair = None
    
    for saddle in tree.saddles:
        for extremum in tree.extrema_connected_to(saddle):
            # Ensure monotone path exists
            if is_monotone_path(tree, saddle, extremum):
                p = priority_func(saddle, extremum)
                if p < min_priority:
                    min_priority = p
                    best_pair = (saddle, extremum)
                    
    return best_pair

def find_branch_containing_node(branches, node_id):
    """
    Checks if a contour tree node is the extremum of any existing branch.
    """
    for branch in branches:
        if branch['extremum'] == node_id:
            return branch
    return None

def find_root_branch(branches):
    """
    Identifies the root branch, which connects two leaves and has no parent.
    """
    for branch in branches:
        if branch['parent'] is None:
            return branch
    return None
```

### Parameters
*   **Priority Measure:**
    *   **Persistence:** $|f(s) - f(e)|$. Typical default [S1][S2, unofficial].
    *   **Volume:** Geometric measure of the enclosed volume [S1][S2, unofficial].
    *   **Hypervolume:** Integral of the scalar field over the enclosed volume [S1][S2, unofficial].
*   **Simplification Order:** Pairs with lower priority are pruned first. Higher priority branches are parents of lower priority branches [S1][S2, unofficial].

### Edge Cases and Pitfalls
*   **Oversegmentation:** Without branch decomposition, the volume may be oversegmented into too many topological zones. Branch decomposition avoids this by concatenating sequences of arcs into a single unit, unifying their topological zones [S1][S2, unofficial].
*   **Genus Changes:** Saddles corresponding to genus changes are not considered in the simplification scheme. Contours of different genus are distinguished by function value alone and do not contribute to contour tree complexity or segmentation [S1][S2, unofficial].
*   **Rendering Artifacts:** When canceling a branch, the transfer function for the canceled branch is set to a solid color, specifically the color of the parent branch’s transfer function at the value of the saddle connecting the branch. This avoids artifacts associated with interpolating opacities over tetrahedral meshes [S1][S2, unofficial]. The method uses hexahedral cells and looks up opacities directly in the transfer function [S2, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Construction:** Requires computing the contour tree ($O(N \log N)$ or $O(N)$ depending on algorithm) and then iteratively pruning pairs. The pruning step is proportional to the number of critical points.
*   **Storage:** The branch decomposition stores one node per prunable saddle-extremum pair. Memory usage is linear with respect to the number of critical points.

### Measured Performance
*   Not given in the sources.

### Scaling
*   The method permits user interactions on a coarse, simplified tree and propagating transfer functions down to the full resolution tree. This allows for efficient interaction regardless of the underlying data resolution [S1][S2, unofficial].

### Strengths and Limitations
*   **Strengths:**
    *   Avoids oversegmentation by unifying topological zones [S1][S2, unofficial].
    *   Provides a hierarchical representation of the contour tree, enabling multi-level interaction [S1][S2, unofficial].
    *   Reduces rendering artifacts by using hexahedral cells and direct transfer function lookups instead of tetrahedral interpolation [S2, unofficial].
*   **Limitations:**
    *   Does not consider genus changes, which may limit its ability to simplify topological features that change the genus of the isosurface [S1][S2, unofficial].
    *   The quality of simplification depends on the chosen priority measure (persistence vs. geometric) [S1][S2, unofficial].

### When to Use
*   Use when topology-controlled volume rendering is required, and the user needs to simplify the isosurface topology while maintaining visual consistency.
*   Use when avoiding oversegmentation is critical for performance or visual clarity.

### When Not to Use
*   Do not use if genus changes are important for the visualization, as they are ignored by this method [S1][S2, unofficial].
*   Do not use if the underlying data is not structured (e.g., unstructured tetrahedral meshes) without modification, as the method is designed for hexahedral cells and trilinear interpolants [S2, unofficial].

## Background

### Origin
The branch decomposition was originally described by Pascucci et al. [S1][S2, unofficial]. It was introduced as an efficient way for storing a hierarchy of contour tree simplifications.

### History and Development
The method was applied to volume rendering in the work "Topology-Controlled Volume Rendering" [S1][S2, unofficial]. It builds upon earlier work on topological simplification, such as that by Carr et al. [S1], which used vertex prune operations. The method was extended to improve volume rendering by using hexahedral cells and direct transfer function lookups, avoiding artifacts from tetrahedral interpolation [S2, unofficial].

### Where it is Used
*   Topology-controlled volume rendering.
*   Interactive visualization of scalar fields where simplification of topological features is desired.

## Sources

[S1] Topology-Controlled Volume Rendering, https://doi.org/10.1109/tvcg.2007.47, fetched 2026-10-27
[S2] Topology-Controlled Volume Rendering, https://www.computer.org/csdl/journal/tg/2007/02/04069241/13rRUytF41s, fetched 2026-10-31 (unofficial)
