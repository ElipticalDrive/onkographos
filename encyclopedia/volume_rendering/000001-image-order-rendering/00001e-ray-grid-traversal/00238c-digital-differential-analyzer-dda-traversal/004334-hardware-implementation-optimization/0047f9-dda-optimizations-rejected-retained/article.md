# DDA Optimizations (Rejected/Retained)

Also called: Boundary event counts, DDA region traversal, Explicit zero-event guards, Per-ray subtree caching, Redundant floor removal

## Overview
This article documents the empirical evaluation of various Digital Differential Analyzer (DDA) traversal optimizations within the Bend voxel renderer, distinguishing between techniques that were retained in the final implementation and those that were rejected due to performance regressions or insufficient gains [S1, unofficial]. The primary problem addressed is the reduction of CPU and GPU overhead associated with ray traversal through compressed voxel data structures [S1, unofficial]. What sets this analysis apart is its rigorous comparative profiling of specific algorithmic variants (such as subtree caching and zero-event guards) against a baseline DDA, revealing that simpler, unoptimized traversal often outperforms complex optimizations due to overhead costs [S1, unofficial].

## Shared principles
The evaluation framework for these optimizations relies on the following shared principles, which govern how performance is measured and how optimizations are integrated into the DDA traversal loop:

1.  **Performance Metrics:** Optimizations are evaluated based on cumulative CPU costs, actual work counters (e.g., number of tree visits or voxel accesses), and complete-frame rendering times [S1, unofficial]. Measurements are taken on both CPU (various architectures like CPU1, CPU6, CPU12) and GPU (fork-seven measurements) [S1, unofficial].
2.  **Correctness Validation:** All retained optimizations must preserve exact surface rendering and image byte accuracy compared to the baseline [S1, unofficial]. Rejected optimizations are also checked for correctness to ensure that performance issues, not bugs, are the cause of rejection [S1, unofficial].
3.  **Baseline Comparison:** The "retained DDA" serves as the baseline against which all new optimizations are compared. The baseline typically involves crossing compressed uniform regions using a standard DDA approach [S1, unofficial].
4.  **Implementation Context:** Optimizations are implemented within the context of a face renderer with shadows omitted to match minimal DDA shading requirements, ensuring that shading costs do not skew traversal performance measurements [S1, unofficial].

## Subtopics

### DDA Region Traversal
*   **What it is:** An optimization that compares upstream world representations to optimize the traversal of compressed uniform regions.
*   **What sets it apart:** It focuses on the structural representation of the voxel data to reduce traversal overhead, as opposed to modifying the DDA step logic itself.
*   **When to choose it:** When the voxel data structure allows for efficient region-based skipping or optimization.
*   **Link:** [DDA Region Traversal](10944-dda-region-traversal.md)

### Corrected Event Counting
*   **What it is:** A technique that replaces the baseline's binary search for event detection with a bounded estimate followed by an exact plane correction.
*   **What sets it apart:** It aims to reduce the computational cost of finding the next voxel boundary by avoiding expensive binary searches, using a cheaper estimate instead.
*   **When to choose it:** When binary search overhead is significant and a bounded estimate can be computed efficiently.
*   **Link:** [Corrected Event Counting](10945-corrected-event-counting.md)

### Per-Ray Subtree Caching
*   **What it is:** An optimization that caches subtree information on a per-ray basis to reduce redundant tree visits.
*   **What sets it apart:** It attempts to leverage spatial coherence between rays by caching traversal state, but this introduces ownership and synchronization overhead.
*   **When to choose it:** Not recommended. This optimization was rejected because the fewer tree visits did not offset the added ownership work.
*   **Link:** [Per-Ray Subtree Caching](10946-per-ray-subtree-caching.md)

### Explicit Zero-Event Guards
*   **What it is:** A set of short-circuiting prototypes that check for zero-event conditions to avoid unnecessary advancement plane calculations.
*   **What sets it apart:** It attempts to skip computation for rays that do not intersect any voxels in a region, but broader guards caused regressions on the GPU.
*   **When to choose it:** Not recommended. The retained DDA is unchanged because broader guards regress on GPU, and the smallest guard adds repeated CPU cost.
*   **Link:** [Explicit Zero-Event Guards](10947-explicit-zero-event-guards.md)

### Redundant Floor Removal
*   **What it is:** An optimization that simplifies the retained event estimate by removing redundant floor operations.
*   **What sets it apart:** It reduces computational complexity in the event estimation step while preserving exact surfaces and image bytes.
*   **When to choose it:** When floor operations are a bottleneck and can be safely removed without affecting correctness.
*   **Link:** [Redundant Floor Removal](10948-redundant-floor-removal.md)

### Boundary Event Counts
*   **What it is:** An optimization that reuses the known integer limit for axes leaving a region to determine boundary events.
*   **What sets it apart:** It leverages known geometric limits to avoid dynamic calculation of boundary events for exiting axes.
*   **When to choose it:** When exiting axes can be identified by their integer limits, leading to CPU improvements.
*   **Link:** [Boundary Event Counts](10949-boundary-event-counts.md)

### Indexed Compressed-Tree Storage
*   **What it is:** A prototype using paged-index storage for compressed trees.
*   **What sets it apart:** It explores alternative memory layouts for tree data, but was rejected based on complete-frame measurements.
*   **When to choose it:** Not recommended. The accepted DDA remains installed.
*   **Link:** [Indexed Compressed-Tree Storage](10950-indexed-compressed-tree-storage.md)

### GPU Decomposition and Pixel Assignment
*   **What it is:** An optimization that adds a typed Bend sweep and an opt-in interleaved layout for GPU processing.
*   **What sets it apart:** It focuses on GPU-specific decomposition and memory layout improvements.
*   **When to choose it:** When GPU workload is the primary bottleneck and interleaved layouts can improve memory access patterns.
*   **Link:** [GPU Decomposition and Pixel Assignment](10951-gpu-decomposition-and-pixel-assignment.md)

## Background
The evaluation of these optimizations is documented in the Bend voxel renderer project, specifically within the `docs/` directory of the repository [S1, unofficial]. The work was conducted by the Bend voxel renderer community, with the primary source being an unofficial repository hosted on GitHub [S1, unofficial]. The optimizations were tested on various CPU architectures (CPU1, CPU6, CPU12) and GPU configurations (fork-seven) to ensure broad applicability [S1, unofficial]. The project emphasizes empirical validation, with detailed profiling and correctness checks for each optimization attempt [S1, unofficial].

## Sources
[S1] aivv73/bend-voxel, https://github.com/aivv73/bend-voxel, fetched 2026-10-28 (unofficial)
