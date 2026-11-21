# Prioritized-Layered Projection (PLP)


## Overview
PLP is an approximate, from-point, object-space visibility algorithm designed for large datasets that requires minimal preprocessing [S1, unofficial]. It solves the problem of efficiently selecting a subset of geometry to render within a strict per-frame budget by using heuristics to estimate cell visibility likelihood [S1, unofficial]. Unlike Teller et al.'s from-region algorithm, which requires extensive preprocessing and assumes axis-aligned cells, PLP makes no geometric assumptions and operates with very little setup overhead [S1, unofficial].

## How it works
**Inputs:** A dataset partitioned into cells; a user-defined budget $B$ representing the maximum number of cells (or amount of geometry) to be rendered per frame; the current viewpoint.
**Outputs:** An approximate visible set of cells.

**Data Structures:**
1.  **Spatial Partitioning:** A structure (e.g., grid or tree) organizing the dataset cells for efficient spatial queries. The specific layout is not detailed in the sources [G: typically a uniform grid or octree for spatial locality].
2.  **Cell Statistics:** For each cell, simple statistics are precomputed. The sources do not specify the exact fields or bit-packing of these statistics [G: typically bounding box volume, center position, or opacity bounds].

**Steps:**
1.  **Preprocessing:**
    1.  Build the spatial partitioning for the dataset.
    2.  Compute simple statistics for each cell.
2.  **Runtime Visibility Estimation:**
    1.  For each cell in the spatial partitioning, calculate a heuristic score estimating the likelihood that the cell is visible from the current viewpoint. The sources do not provide the specific formula for this heuristic [G: typically involves distance to camera, projected area, or opacity bounds].
3.  **Budgeted Selection:**
    1.  Sort or prioritize cells based on their visibility heuristic scores.
    2.  Select the top $B$ cells (or cells whose cumulative cost fits within budget $B$) to form the approximate visible set.
    3.  Render only the cells in this set.

**Parameters:**
*   $B$: User-defined budget of geometry to be rendered per frame. Typical values are not given in the sources [G: depends on hardware capabilities and frame rate requirements].

**Edge Cases and Pitfalls:**
*   **Approximation Error:** As an approximate algorithm, PLP does not guarantee finding all visible primitives. Cells that are actually visible may be excluded if their heuristic score is low [S1, unofficial].
*   **Heuristic Quality:** The effectiveness of the algorithm depends heavily on the chosen heuristic. Poor heuristics may result in rendering invisible cells or missing visible ones [S1, unofficial].

## Subtopics

### Conservative Prioritized-Layered Projection (cPLP)
cPLP is a conservative, image-space algorithm developed by Klosowski and Silva [S1, unofficial]. It uses PLP to obtain an initial guess of the visible set and then augments this approximate set to make it conservative, ensuring that no visible primitives are missed [S1, unofficial]. This addresses the limitation of PLP's approximation by guaranteeing completeness at the cost of potentially rendering more geometry than strictly necessary.

## Operating characteristics
**Time and Memory Complexity:**
*   Preprocessing requires building a spatial partitioning and computing cell statistics. The exact complexity is not given in the sources [G: likely $O(N)$ or $O(N \log N)$ depending on the partitioning structure, where $N$ is the number of cells].
*   Runtime complexity depends on the heuristic calculation and sorting/prioritization. The sources do not provide specific complexity bounds [G: likely $O(N)$ for heuristic evaluation and $O(N \log N)$ or $O(N)$ for selection if using a heap or selection algorithm].

**Measured Performance:**
*   Specific performance metrics, hardware, resolution, or scene details are not provided in the sources [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Requires very little preprocessing; makes no assumptions about the geometry of the model (unlike Teller et al. [138]); handles arbitrary cell shapes [S1, unofficial].
*   **Limitations:** Does not guarantee exact visibility (approximate); may miss visible cells; performance depends on heuristic quality [S1, unofficial].

**When to Use:**
*   Use PLP when rendering large datasets with limited preprocessing time and when an approximate visible set is sufficient to meet a strict per-frame rendering budget.
*   Use cPLP when a conservative (complete) visible set is required, and PLP can provide a good initial guess to reduce the search space.

**When Not to Use:**
*   Do not use PLP if exact visibility is required.
*   Do not use PLP if the dataset allows for extensive preprocessing and axis-aligned geometry assumptions, in which case Teller et al.'s algorithm might be more suitable [S1, unofficial].

## Background
PLP was developed by Klosowski and Silva [S1, unofficial]. It is presented in the context of out-of-core visualization techniques for large datasets [S1, unofficial]. The algorithm is contrasted with from-region visibility algorithms like Teller et al.'s, which require long preprocessing times and assume axis-aligned cells [S1, unofficial]. cPLP is also attributed to Klosowski and Silva [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
