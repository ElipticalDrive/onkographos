# Interval Tree with Cell Grouping

Also called: Grouped Interval Tree

## Overview
The Interval Tree with Cell Grouping is a memory-optimized variant of the Standard Interval Tree for Isosurface Extraction [S1, unofficial]. It reduces the memory overhead of the auxiliary lists by having each pointer reference a group of eight adjacent cells rather than a single cell, resulting in a tree size approximately 1/8th that of a regular interval tree [S1, unofficial]. This reduction comes at the cost of increased query time, as intersecting groups require checking all eight constituent cells individually [S1, unofficial].

## How it works
**Inputs:** A 3D scalar volume grid; an isovalue $q$.
**Outputs:** A list of active cells (or groups of cells) intersecting the isovalue $q$.

**Data Structures:**
The tree structure mirrors the Standard Interval Tree but with grouped pointers [S1, unofficial].
*   **Node:** Contains a splitting value $q_x$ and two auxiliary lists: `I-Left` and `I-Right`.
*   **Pointer:** Each entry in `I-Left` and `I-Right` is a pointer to a **group of eight adjacent cells** [S1, unofficial].
*   **Group Definition:**
    *   The volume is partitioned into disjoint groups of eight adjacent cells [S1, unofficial].
    *   The group with the lowest coordinates $(x, y, z)$ serves as the representative for the set [S1, unofficial].
    *   The group's bounding box is defined by the global minimum and maximum values of all eight constituent cells [S1, unofficial].

**Algorithm Steps:**

1.  **Construction:**
    1.  Partition the volume into disjoint groups of eight adjacent cells.
    2.  For each group, compute the global minimum value $v_{min}$ and global maximum value $v_{max}$ across all eight cells.
    3.  Insert the group into the interval tree using $v_{min}$ and $v_{max}$ as the interval bounds, treating the group as a single entity.
    4.  Store the pointer to the group in the appropriate auxiliary lists (`I-Left` for cells with $v_{max} < q_x$, `I-Right` for cells with $v_{min} > q_x$) [S1, unofficial].

2.  **Query:**
    1.  Traverse the tree to find nodes where the interval $[v_{min}, v_{max}]$ intersects the query isovalue $q$.
    2.  For each intersecting node, iterate through the `I-Left` and `I-Right` lists.
    3.  **Stop Condition:** Terminate iteration through a specific list as soon as a group is encountered whose interval does not intersect $q$ [S1, unofficial].
    4.  **Verification:** For every group identified as intersecting $q$, iterate through all eight constituent cells.
    5.  Check each of the eight cells individually to determine if the isovalue $q$ lies within their specific scalar range.
    6.  Add all cells that satisfy the condition to the output list [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_grouped_interval_tree(root, q):
    active_cells = []
    traverse(root, q, active_cells)
    return active_cells

def traverse(node, q, active_cells):
    if node is None:
        return

    # Check if the node's interval intersects q
    if node.max_val < q or node.min_val > q:
        return

    # Check auxiliary lists
    # I-Left: sorted by max_val ascending. Stop when max_val < q.
    for group in node.i_left:
        if group.max_val < q:
            break
        verify_group_cells(group, q, active_cells)

    # I-Right: sorted by min_val descending. Stop when min_val > q.
    for group in node.i_right:
        if group.min_val > q:
            break
        verify_group_cells(group, q, active_cells)

    # Recurse
    if node.left_child:
        traverse(node.left_child, q, active_cells)
    if node.right_child:
        traverse(node.right_child, q, active_cells)

def verify_group_cells(group, q, active_cells):
    # Iterate through the 8 constituent cells of the group
    for cell in group.cells:
        if cell.min_val <= q <= cell.max_val:
            active_cells.append(cell)
```

**Parameters:**
*   **Group Size:** 8 cells per pointer [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Worst-Case Query Performance:** The complexity degrades to $O(\log(n) + 8k)$ if the isosurface intersects exactly one cell from each group of eight [S1, unofficial]. This scenario is rare in practical data volumes but represents the theoretical upper bound [S1, unofficial].
*   **Precision:** The method relies on exact comparisons of scalar values. Floating-point precision issues may affect boundary conditions where $q$ equals a cell's min or max value [G].

## Subtopics

### Ordered Grouped Interval Tree
An extension that combines cell grouping with sorting information to accelerate queries [S1, unofficial].
*   **Definition:** Stores pointers to groups of eight cells in two lists, sorted by the group's minimum value (ascending) and maximum value (descending) [S1, unofficial].
*   **Layout:** Each group includes an order vector of eight numbers (1–8), indicating the permutation of cells within the group. This vector requires 3 bits per number, totaling 24 bits (3 bytes) per group [S1, unofficial].
*   **Complexity:** Space overhead is 1.75 bytes per cell (calculated as $2 \times (4 \text{ bytes pointer} + 3 \text{ bytes order}) / 8 \text{ cells}$) [S1, unofficial]. Query times are intermediate between the standard interval tree and the unsorted grouped tree [S1, unofficial].
*   **Usage:** Choose when memory is constrained but query speed of the basic grouped tree is insufficient [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Query complexity is $O(\log(n) + 8k)$, where $k$ is the number of active cells in the original (un-grouped) tree [S1, unofficial]. The factor of 8 arises from the worst-case need to check all sub-cells in intersecting groups [S1, unofficial].
*   **Memory Complexity:** The total size of the interval tree is approximately 1/8th of a regular interval tree [S1, unofficial]. The height of the tree is also reduced because it effectively manages 1/8th the number of nodes [S1, unofficial].
*   **Performance:** Tests on an 8-bit volume show significant size reduction compared to the standard tree [S1, unofficial]. Query times are slower than the standard interval tree but faster than the unsorted grouped variant when ordering is applied [S1, unofficial].
*   **Strengths:** Drastic reduction in memory footprint, enabling larger volumes to fit in cache or main memory.
*   **Limitations:** Increased query time due to per-cell verification within groups; worst-case performance is significantly worse than the standard tree [S1, unofficial].

## Background
The Interval Tree with Cell Grouping was introduced as a method to address the prohibitive memory requirements of standard interval trees for isosurface extraction, which can require up to nine times the size of the volume data [S1, unofficial]. It was proposed in the context of the Matryoshka Algorithm for speeding up isosurfacing [S1, unofficial].

## Sources
[S1] Speeding Up Isosurfacing: The Matryoshka Algorithm, https://doi.org/10.2312/localchapterevents/tpcg/tpcg07/097-105, fetched 2026-10-26 (unofficial)
