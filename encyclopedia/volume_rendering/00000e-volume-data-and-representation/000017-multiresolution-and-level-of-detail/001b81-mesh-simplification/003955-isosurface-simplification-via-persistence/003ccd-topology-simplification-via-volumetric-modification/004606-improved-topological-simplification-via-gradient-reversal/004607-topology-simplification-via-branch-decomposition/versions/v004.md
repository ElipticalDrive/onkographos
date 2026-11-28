# Topology Simplification via Branch Decomposition

Also called: Branch Decomposition, Hierarchical Contour Tree Simplification, Improved Topological Simplification for Volume Rendering

## Overview
Topology Simplification via Branch Decomposition is a method for storing and traversing a hierarchy of contour tree simplifications to control the topology of volume-rendered isosurfaces. It solves the problem of "oversegmentation," where standard contour tree representations create too many distinct topological zones for regions that share value ranges, by concatenating sequences of arcs into single hierarchical units [S1][S2, unofficial]. Unlike its parent method, which modifies the scalar field via gradient reversal, this method operates on the structural hierarchy of the contour tree itself, allowing for efficient user interaction and transfer function propagation [S1]. It is distinct from methods that treat all zones of equivalent nesting depth as equivalent, as it explicitly respects the hierarchical importance of features [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A contour tree of a scalar field, where nodes represent critical points (minima, maxima, saddles) and edges represent monotone paths between them.
*   **Output:** A branch decomposition, which is a rooted tree of "branches." Each branch represents a topological simplification (cancellation) of a saddle-extremum pair.

**Data Structures**
The branch decomposition is a hierarchical decomposition of the contour tree.
*   **Branch:** Defined by a pair of critical points: a saddle and an extremum connected by a monotone path.
*   **Root Branch:** Exactly one branch connects two leaves (extrema) of the original contour tree.
*   **Interior Branch:** Every other branch connects one leaf (extremum) to an interior node (saddle) of another branch.
*   **Hierarchy:** The decomposition forms a tree where nodes are branches. The parent of a branch is the branch containing the saddle that connects to the extremum of the child branch.

**Steps**
1.  **Construct the Branch Decomposition:**
    1.  Identify all saddle-extremum pairs in the contour tree.
    2.  Assign a priority measure to each pair. The standard measure is **persistence**, defined as:
        $$ P = |f(s) - f(e)| $$
        where $f(s)$ is the scalar value at the saddle $s$ and $f(e)$ is the scalar value at the extremum $e$ [S1][S2, unofficial]. Alternative geometric measures include volume or hypervolume (integral of the scalar field over the enclosed volume) [S1][S2, unofficial].
    3.  Sort pairs by priority. Pairs with lower priority are processed first [S1][S2, unofficial].
    4.  Iterate through the sorted pairs:
        a.  Prune the saddle-extremum pair from the contour tree.
        b.  Collapse the remaining node (the saddle) into the parent structure, leaving an arc.
        c.  Create a branch node representing this pair.
        d.  Link this branch as a child to the arc (or branch) created by the collapse. This arc is the parent to which the pair is linked [S1][S2, unofficial].
    5.  Repeat until all pairs are processed. The result is a rooted tree of branches, with the most important branches (highest priority) at the top [S1][S2, unofficial].

2.  **Apply Simplification (Rendering):**
    1.  To simplify the topology, discard branches from the decomposition. Discarding a branch is equivalent to performing a cancellation of the corresponding critical points, which is equivalent to a vertex prune operation [S1].
    2.  For each discarded branch, modify the rendering to reflect the topological change. This is achieved by setting the transfer function for the canceled branch to a solid color.
    3.  The solid color is the color of the **parent branch’s** transfer function evaluated at the scalar value of the **saddle** that connects the child branch to the parent [S1].
    4.  This effectively "chops off" peaks and "fills in" valleys, displaying an image consistent with the pruned contour tree [S1].

**Parameters**
*   **Priority Measure:** Persistence (default) or geometric measures (volume, hypervolume) [S1][S2, unofficial].
*   **Threshold:** A cutoff value for the priority measure to determine which branches to discard [G].

**Edge Cases and Pitfalls**
*   **Genus Changes:** Saddles corresponding to genus changes (changes in the number of connected components or holes) are currently not considered in the simplification scheme. Contours of different genus are distinguished by function value alone and do not affect segmentation or contribute to contour tree complexity in this context [S1][S2, unofficial].
*   **Oversegmentation:** Without branch decomposition, the volume may be oversegmented into too many topological zones. Branch decomposition avoids this by concatenating sequences of arcs into a single unit, unifying their topological zones [S1][S2, unofficial].
*   **Artifact Avoidance:** Unlike methods using tetrahedral meshes and interpolated opacities, this method uses hexahedral cells and looks up opacities directly in the transfer function to avoid artifacts in cells spanning multiple topological zones [S2, unofficial].

## Subtopics

### [Pruning Hierarchy Construction](16712-pruning-hierarchy-construction.md)
A variant that expands the branch decomposition into a ternary-like structure to reduce the branching factor, making it more suitable for user interfaces while maintaining strict ordering of feature importance.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The construction involves sorting saddle-extremum pairs and tree manipulations, which is generally efficient for hierarchical storage.
*   **Memory Complexity:** Not given in the sources. The branch decomposition stores a hierarchy of branches, which is more compact than storing all individual topological zones separately.
*   **Strengths:** Avoids oversegmentation; allows user interaction on a coarse, simplified tree; permits propagation of transfer functions to full resolution; efficient storage of simplification hierarchy.
*   **Limitations:** Does not consider genus changes; relies on the accuracy of the contour tree construction; topological changes in the function may not be ideal from a user's perspective if the simplification is too aggressive (as noted in the parent method's critique of simple chopping, though branch decomposition provides a structured way to manage this) [S1].
*   **When to Use:** When efficient hierarchical storage and traversal of contour tree simplifications is required for volume rendering, particularly when avoiding oversegmentation is critical.
*   **When Not to Use:** When genus changes are important for the visualization or when geometric measures are preferred over persistence for prioritization (though geometric measures are supported, persistence is the standard).

## Background
Branch decomposition was described by Pascucci et al. [35] as an efficient way for storing a hierarchy of contour tree simplifications. It was adopted in the context of topology-controlled volume rendering by Pascucci and Cole-McLaughlin [19] and later extended by Carr et al. [34] and others [S1][S2, unofficial]. The method addresses the limitations of flat contour tree representations by providing a hierarchical structure that reflects the importance of topological features.

## Sources
[S1] Topology-Controlled Volume Rendering, https://doi.org/10.1109/tvcg.2007.47, fetched 2026-10-27
[S2] Topology-Controlled Volume Rendering, https://www.computer.org/csdl/journal/tg/2007/02/04069241/13rRUytF41s, fetched 2026-10-31 (unofficial)
