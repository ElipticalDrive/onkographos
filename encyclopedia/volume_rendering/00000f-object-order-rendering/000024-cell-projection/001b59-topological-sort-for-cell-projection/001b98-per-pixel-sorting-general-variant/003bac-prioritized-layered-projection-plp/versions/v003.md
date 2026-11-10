# Prioritized-Layered Projection (PLP)
## Overview
PLP is an approximate, from-point, object-space visibility algorithm designed for large datasets that requires minimal preprocessing [S1, unofficial]. It solves the problem of efficiently selecting a subset of geometry to render within a strict per-frame budget by using heuristics to estimate cell visibility likelihood [S1, unofficial]. Unlike Teller et al.'s from-region algorithm, which requires extensive preprocessing and assumes axis-aligned cells, PLP makes no geometric assumptions and operates with very little setup overhead [S1, unofficial].

## How it works
**Inputs and Outputs:**
* **Inputs:** A dataset partitioned into cells; a user-defined budget $B$ representing the maximum amount of geometry to be rendered per frame; the current viewpoint [S1, unofficial].
* **Outputs:** An approximate visible set of cells [S1, unofficial].

**Data Structures:**
1. **Spatial Partitioning:** A structure organizing the dataset cells for efficient spatial queries [S1, unofficial]. Not given in the sources.
2. **Cell Statistics:** For each cell, simple statistics are precomputed [S1, unofficial]. Not given in the sources.

**Steps:**
1. **Preprocessing:** Build a spatial partitioning for the dataset and compute simple statistics for each cell [S1, unofficial].
2. **Runtime Visibility Estimation:** Use heuristics to estimate how likely it is for each cell to be visible [S1, unofficial].
3. **Budgeted Selection:** Add cells to an approximate visible set up to the user-defined budget [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
* $B$: User-defined budget of geometry to be rendered per frame. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
* **Approximation Error:** As an approximate algorithm, PLP does not guarantee finding all visible primitives [S1, unofficial].
* **Heuristic Quality:** The effectiveness of the algorithm depends heavily on the chosen heuristic [S1, unofficial].

**Pseudocode:**
Not provided in the sources.

## Subtopics

### Conservative Prioritized-Layered Projection (cPLP)
cPLP is a conservative, image-space algorithm developed by Klosowski and Silva [S1, unofficial]. It uses PLP to obtain an initial guess of the visible set and then augments this approximate set to make it conservative [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
* Preprocessing requires building a spatial partitioning and computing cell statistics. The exact complexity is not given in the sources.
* Runtime complexity depends on the heuristic calculation and prioritization. The sources do not provide specific complexity bounds.

**Measured Performance:**
* Specific performance metrics, hardware, resolution, or scene details are not given in the sources.

**Strengths and Limitations:**
* **Strengths:** Requires very little preprocessing and makes no assumptions about the geometry of the model, unlike Teller et al.'s algorithm which requires long preprocessing and assumes axis-aligned cells [S1, unofficial].
* **Limitations:** Does not guarantee exact visibility and may miss visible cells [S1, unofficial].

**When to Use:**
* Use PLP when rendering large datasets with limited preprocessing time and when an approximate visible set is sufficient to meet a strict per-frame rendering budget [S1, unofficial].

**When Not to Use:**
* Do not use PLP if exact visibility is required or if the dataset allows for extensive preprocessing and axis-aligned geometry assumptions, in which case Teller et al.'s algorithm might be more suitable [S1, unofficial].

## Background
PLP was developed by Klosowski and Silva [S1, unofficial]. The algorithm is contrasted with from-region visibility algorithms like Teller et al.'s, which require long preprocessing times and assume axis-aligned cells [S1, unofficial]. cPLP is also attributed to Klosowski and Silva [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
