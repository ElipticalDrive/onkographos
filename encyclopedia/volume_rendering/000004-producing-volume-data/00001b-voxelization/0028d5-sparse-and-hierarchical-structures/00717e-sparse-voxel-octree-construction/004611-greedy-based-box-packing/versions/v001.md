# Greedy-based Box Packing

Also called: Greedy-based Approach

## Overview
A volume rendering pre-processing method that partitions a volume into variably-sized bounding boxes based on alpha values and packs these boxes into 2D texture slices to bypass 3D texture padding waste [S1, unofficial]. It sets itself apart from its siblings by explicitly selecting between greedy partitioning and regular 2D packing based on a dataset sparsity check, whereas [Factoring Tree Construction](9647-factoring-tree-construction.md) builds a hierarchical tree structure directly without this specific 2D slice-packing hybrid decision logic.

## How it works
**Inputs:** 3D volume dataset (alpha values), user-defined density threshold, user-defined sparsity threshold.
**Outputs:** Packed 2D texture slices, lookup table mapping packed rectangles to original coordinates/orientation.

**Data Structures:**
1.  **Seed Map:** Binary 3D array. `1` if voxel alpha > density threshold (important/seed), `0` otherwise (unimportant).
2.  **Box List:** Collection of bounding boxes defined by min/max coordinates $(x_{min}, y_{min}, z_{min})$ to $(x_{max}, y_{max}, z_{max})$.
3.  **Rectangle List:** For each 2D slice, a list of 2D rectangles derived from box intersections.
4.  **Lookup Table:** Stores coordinates and orientation for each packed rectangle.

**Steps:**

1.  **Create Seed Map:** Iterate through all voxels. If $\alpha_{voxel} > \text{density\_threshold}$, mark as seed (1). Else, mark as unimportant (0) [S1, unofficial].
2.  **Sparse Check:** Calculate dataset sparsity $S$ (ratio of unimportant voxels to total voxels).
    *   If $S < \text{sparsity\_threshold}$, select **Greedy-based Approach**.
    *   Else, select **Regular 2D Packing** [S1, unofficial].
3.  **Greedy Partitioning (if Greedy-based selected):**
    *   Initialize an empty list of boxes.
    *   Identify all unassigned seed voxels in the Seed Map.
    *   For each unassigned seed voxel:
        *   Initialize a new box $B$ with dimensions $1 \times 1 \times 1$ at the seed's position.
        *   Mark the seed voxel as "taken".
        *   **Expansion Loop:**
            *   Evaluate profit for extending $B$ by one unit in six directions: $x$ (left/right), $y$ (up/down), $z$ (forward/backward).
            *   Profit formula:
                $$ \text{Profit} = N_{\text{new\_seeds}} - N_{\text{empty}} $$
                Where $N_{\text{new\_seeds}}$ is the count of seed voxels encompassed by the extension, and $N_{\text{empty}}$ is the count of empty (unimportant) voxels encompassed [S1, unofficial].
            *   Find the direction with the maximum positive profit.
            *   If max profit $> 0$:
                *   Extend $B$ in that direction.
                *   Mark all newly encompassed seed voxels as "taken".
                *   Repeat expansion loop.
            *   Else:
                *   Stop expansion. Add $B$ to Box List.
4.  **2D Slice Generation:**
    *   For each 2D slice (e.g., along the Z-axis):
        *   Intersect the Box List with the slice plane.
        *   Generate a list of 2D rectangles representing the intersection of each 3D box with the slice.
5.  **2D Greedy Packing:**
    *   Sort the rectangle list in descending order of area [S1, unofficial].
    *   Initialize the packed area (initially empty).
    *   For each rectangle $R$ in the sorted list:
        *   Identify empty regions adjacent to the currently packed area.
        *   Evaluate four placement scenarios for each gap:
            (a) Place right.
            (b) Place top.
            (c) Place right-transposed.
            (d) Place top-transposed.
        *   Calculate the growth in the slice's total bounding area for each scenario.
        *   Select the placement that minimizes this growth.
        *   Place $R$ in the selected orientation and location.
        *   Store $R$'s coordinates and orientation in the Lookup Table.
6.  **Stacking:** Stack all packed 2D slices to form the final 3D texture object [S1, unofficial].

**Parameters:**
*   **Density Threshold:** User-defined. Typical values shown in sources: 0.05 and 0.5 [S1, unofficial].
*   **Sparsity Threshold:** User-defined. Example value in source: 75% [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The profit calculation relies on integer counts of voxels, avoiding floating-point precision issues during box expansion [S1, unofficial].
*   **Boundaries:** Expansion checks must ensure the box does not exceed volume boundaries [G].
*   **Degenerate Input:** If no voxels exceed the density threshold, the box list is empty, resulting in an empty texture [G].

## Subtopics

### [Hybrid Greedy/Regular 2D Packing](11866-hybrid-greedy-regular-2d-packing.md)
A hybrid CPU-GPU pipeline that partitions a 3D volume into non-empty bounding boxes, packs them into 2D texture slices, and optionally applies post-packing rotation to minimize memory overhead for volume rendering. It differs from pure greedy approaches by integrating a sparsity check to switch between greedy partitioning and regular 2D packing, and includes post-packing rotation [11866-hybrid-greedy-regular-2d-packing.md].

### Maximal Rectangles Best Short Side Fit (MAXRECTS-BSSF)
A 2D packing algorithm identified as a top-tier variant for empty-space management, used to resolve spatial inefficiency in 2D rectangle packing by managing maximal rectangles [S2]. It translates theoretical 2D packing strategies into a practical 3D volume rendering pipeline by applying fast 2D packing directly to non-empty voxels [S2]. Unlike exact solvers which have massive computational costs, or simple Shelf algorithms which waste surface area when rectangle heights do not match shelf ceilings, MAXRECTS-BSSF offers a balance of efficiency and compactness [S2].

## Operating characteristics
*   **Time Complexity:** The greedy partitioning step is linear with respect to the number of seed voxels, but box expansion requires neighbor checks. The 2D packing step is NP-Hard in general, but the greedy heuristic runs in polynomial time relative to the number of rectangles per slice [S2].
*   **Memory Complexity:** Significantly reduced compared to full 3D textures due to the bypassing of 3D texture padding waste inherent in AMR structures [S1, unofficial].
*   **Measured Performance:** In a test case, an original slice of size $128 \times 128$ was condensed into a packed dimension of $54 \times 196$ [S1, unofficial].
*   **Strengths:** Explicitly bypasses severe 3D texture padding waste and complex decoding overheads of out-of-core streaming [S1, unofficial]. Adapts to dataset sparsity.
*   **Limitations:** The 2D packing is a heuristic and does not guarantee optimal bounds [S2]. Performance depends heavily on the choice of density and sparsity thresholds.
*   **When to use:** When memory bandwidth is a bottleneck and the volume data is sparse or has irregular non-empty regions.
*   **When not to use:** When the dataset is dense (sparsity below threshold, triggering regular packing) or when optimal packing bounds are required despite higher computational cost [S2].

## Background
Originated in the paper "A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data" [S1, unofficial][S2]. The method addresses the limitations of existing sparse structures like AMR (which suffer from 3D texture padding waste) and out-of-core streaming (which has decoding overheads) [S1, unofficial]. It builds on the observation that 2D rectangle packing is NP-Hard, and while exact solvers are optimal, they are impractical for real-time systems, leading to the adoption of greedy heuristics like MAXRECTS-BSSF [S2].

## Sources
[S1] A Hybrid CPU-GPU Approach for Memory-Efficient Volume ..., https://www.researchgate.net/publication/408581180_A_Hybrid_CPU-GPU_Approach_for_Memory-Efficient_Volume_Rendering_of_Medical_Image_Data, fetched 2026-10-26 (unofficial)
[S2] A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data, https://doi.org/10.24132/csrn.2026-d53, fetched 2026-10-26
