# Redundant coarse data bridging


## Overview
Redundant coarse data bridging is a method for eliminating geometric cracks and holes that appear between different refinement levels in Adaptive Mesh Refinement (AMR) data structures [S1]. It solves the problem of discontinuous topology by utilizing data points stored at coarser levels that are typically unused during post-analysis to bridge the gap between adjacent levels [S1]. Unlike the stitching cells algorithm, which constructs unstructured geometry across levels, this method capitalizes on the existing overlap inherent in patch-based AMR structures [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An AMR dataset with multiple refinement levels, specifically patch-based AMR where coarse-level data overlaps with fine-level domains [S1].
*   **Output:** A continuous, gap-free representation where geometric cracks between refinement levels are eliminated [S1].

**Data Structures:**
The method relies on the standard AMR hierarchy where each level contains a set of patches or cells [G]. The key structural feature is the existence of "redundant" data points at coarser levels, defined as data points stored at coarser levels that are not utilized during post-analysis [S1]. These points (e.g., 0D data points) lie within the spatial extent of finer-level patches [S1]. Exact field layouts and bit packing are not given in the sources.

**Steps:**
1. **Identify Overlap:** Detect regions where coarser-level patches overlap with finer-level patches [S1].
2. **Locate Redundant Data:** Identify data points at the coarser level that fall within the boundaries of the finer-level patches [S1].
3. **Bridge the Gap:** Use these redundant coarse data points to fill the geometric or topological gaps between the coarse and fine levels [S1].
4. **Ensure Continuity:** Incorporate these redundant points into the visualization pipeline to ensure iso-surfaces or scalar fields are continuous across level boundaries, fixing cracks seen in naive re-sampling [S1].

**Formulas:** Not given in the sources.

**Parameters:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Patch-based AMR Specificity:** The method is particularly effective in patch-based AMR where overlap is a structural characteristic [S1]. It may not apply directly to non-patch-based AMR structures without modification [G].
*   **Data Redundancy Management:** The method assumes that the redundant data is available and accessible [S1]. Care must be taken to ensure that the redundant coarse data does not interfere with the higher-resolution fine data during query operations [G].

## Subtopics

### [AMR Grid Completion](15674-amr-grid-completion.md)
Converts an adaptive wavelet grid into a traditional node-centered AMR grid by adding supplementary nodes, enabling use with established AMR visualization software like ParaView or VisIt. Chosen when leveraging existing AMR rendering pipelines is preferred over implementing custom wavelet-based renderers.

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources.

**Measured Performance:** Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Eliminates cracks and holes between AMR levels without constructing new unstructured geometry [S1]. Fixes visualization artifacts that appear with re-sampling [S1].
*   **Limitations:** Relies on the presence of redundant coarse data, which may not be available in all AMR configurations.

**When to Use:** Use for visualizing patch-based AMR data when encountering cracks between refinement levels and a simpler alternative to stitching cells is desired [S1].

## Background
**Origin:** The method is presented as an alternative to the stitching cells algorithm for filling gaps between AMR levels [S1].

**History and Development:** Illustrated in 2D examples demonstrating the elimination of cracks between coarse and fine levels [S1].

**Where it is Used:** AMR visualization pipelines [S1].

## Sources
[S1] Analyzing Impact of Data Reduction Techniques on Visualization for AMR Applications Using AMReX Framework, https://doi.org/10.1145/3624062.3625123, fetched 2026-10-27
