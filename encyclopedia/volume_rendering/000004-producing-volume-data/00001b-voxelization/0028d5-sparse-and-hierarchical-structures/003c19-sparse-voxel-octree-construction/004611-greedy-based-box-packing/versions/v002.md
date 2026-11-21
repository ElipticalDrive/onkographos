# Greedy-based Box Packing

Also called: Greedy-based Approach

## Overview
Greedy-based Box Packing is a volume rendering pre-processing method that partitions a 3D volume into variably-sized bounding boxes based on alpha values and packs these boxes into 2D texture slices to bypass 3D texture padding waste [S1, unofficial]. It distinguishes itself from its sibling [Factoring Tree Construction](9647-factoring-tree-construction.md) by explicitly compacting isolated non-empty voxels into 2D texture slices, thereby avoiding the severe 3D texture padding waste of Adaptive Mesh Refinement (AMR) structures and the complex decoding overheads of out-of-core streaming [S1, unofficial]. Unlike [Li03a], which partitions based on density and gradient magnitude, this method applies only to alpha values as density magnitude for a streamlined process [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (specifically alpha values), a user-defined density threshold ($T_{density}$), and a user-defined sparsity threshold ($T_{sparsity}$).
*   **Output:** A set of 2D texture slices containing packed rectangles, along with a lookup table storing the coordinates and orientation of each packed rectangle for GPU sampling [S1, unofficial].

**Data Structures:**
1.  **Seed Map:** A binary map of the same dimensions as the input volume. Each voxel is marked as a "seed" (1) if its alpha value $\alpha > T_{density}$, indicating an important voxel; otherwise, it is marked as unimportant (0) [S1, unofficial].
2.  **Bounding Boxes:** A collection of 3D axis-aligned bounding boxes. Each box is defined by its minimum and maximum coordinates $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$. Initially, each box is $1 \times 1 \times 1$.
3.  **2D Rectangles:** For each 2D slice, a list of 2D rectangles derived from the intersection of the 3D bounding boxes with that slice. Each rectangle is defined by position, width, height, and orientation (normal/rotated) [S1, unofficial].

**Algorithm Steps:**

1.  **Seed Map Creation:** Iterate through all voxels in the volume. If $\alpha_{voxel} > T_{density}$, mark the voxel as a seed in the Seed Map. Voxels with $\alpha_{voxel} \le T_{density}$ are considered empty/unimportant [S1, unofficial].

2.  **Sparse Check:** Calculate the dataset sparsity $S$ (ratio of unimportant voxels to total voxels).
    *   If $S < T_{sparsity}$, select the **Greedy-based Approach**.
    *   If $S \ge T_{sparsity}$, select the **Regular 2D Packing Approach** (details in [Hybrid Greedy/Regular 2D Packing](11866-hybrid-greedy-regular-2d-packing.md)) [S1, unofficial].

3.  **Greedy Partitioning (Box Expansion):**
    *   Initialize an empty list of boxes.
    *   Iterate through each voxel in the Seed Map. If the voxel is a seed and not yet marked as "taken":
        a.  Initialize a new box $B$ of size $1 \times 1 \times 1$ centered at or containing the current seed voxel. Mark the seed voxel as "taken" [S1, unofficial].
        b.  Evaluate the **Profit** of expanding $B$ by one unit in each of the six directions: $x$ (left/right), $y$ (up/down), $z$ (forward/backward) [S1, unofficial].
        c.  The Profit for a direction is calculated as:
            $$ \text{Profit} = N_{new\_seeds} - N_{empty} $$
            where $N_{new\_seeds}$ is the number of new seed voxels encompassed by the expansion, and $N_{empty}$ is the number of empty (unimportant) voxels encompassed [S1, unofficial].
        d.  Identify the direction with the maximum positive profit.
        e.  If the maximum profit is $> 0$, expand $B$ in that direction. Mark all newly encompassed seed voxels as "taken". Repeat step (b).
        f.  If no direction yields a positive profit, finalize $B$ and add it to the list of boxes [S1, unofficial].

4.  **2D Slice Intersection:**
    *   For each 2D slice $k$ of the volume (e.g., along the Z-axis):
        a.  Intersect the list of 3D bounding boxes with the plane of slice $k$.
        b.  Generate a list of 2D rectangles $R_k$ representing the cross-sections of the boxes within that slice [S1, unofficial].

5.  **2D Greedy Packing:**
    *   Sort the rectangles $R_k$ in descending order of area [S1, unofficial].
    *   Initialize the packed area of the slice as empty.
    *   For each rectangle $r$ in the sorted list:
        a.  Identify available empty regions (gaps) adjacent to the currently packed area [S1, unofficial].
        b.  Recursively evaluate four placement scenarios for each gap:
            i.  Place right.
            ii. Place top.
            iii. Place right-transposed.
            iv. Place top-transposed.
        c.  Calculate the growth in the slice's total bounding area for each scenario.
        d.  Select the placement that minimizes the growth of the slice's total bounding area.
        e.  Place $r$ in the selected orientation and position. Update the list of empty regions [S1, unofficial].

6.  **Lookup Table Generation:** Store the coordinates, dimensions, and orientation of each packed rectangle in a lookup table associated with its slice index [S1, unofficial].

**Parameters:**
*   $T_{density}$: User-defined density threshold for alpha values. Typical values mentioned in sources include 0.05 and 0.5 [S1, unofficial].
*   $T_{sparsity}$: User-defined sparsity threshold to switch between greedy and regular packing [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The profit calculation relies on integer counts of voxels, avoiding floating-point precision issues during expansion [S1, unofficial].
*   **Boundaries:** Expansion is limited by the volume boundaries.
*   **Degenerate Input:** If no voxels exceed $T_{density}$, no boxes are generated. If all voxels are seeds, the greedy algorithm may produce large boxes, potentially reducing packing efficiency if the sparsity check is not tuned correctly [S1, unofficial].

## Subtopics

### [Hybrid Greedy/Regular 2D Packing](11866-hybrid-greedy-regular-2d-packing.md)
Hybrid Greedy/Regular 2D Packing is a volume rendering pre-processing method that partitions a 3D volume into non-empty bounding boxes and packs them into 2D texture slices to bypass 3D texture padding waste. It distinguishes itself from sibling methods by dynamically selecting between a recursive greedy packing strategy and a regular 2D packing strategy based on dataset sparsity, followed by post-packing rotation optimization to minimize memory overhead. Unlike [Li03a], which partitions based on density and gradient magnitude, this method applies only to alpha values as density magnitude for a streamlined process.

### Maximal Rectangles Best Short Side Fit (MAXRECTS-BSSF)
Maximal Rectangles Best Short Side Fit (MAXRECTS-BSSF) is a 2D packing algorithm identified as a top-tier variant for empty-space management, used to resolve spatial inefficiency in 2D rectangle packing by managing maximal rectangles [S2]. It translates theoretical 2D packing algorithms into a practical 3D volume rendering pipeline by applying fast 2D packing strategies directly to non-empty voxels [S2]. Exact solvers guarantee optimal bounds but have massive computational cost, making them impractical for real-time systems [S2]. Simple Shelf algorithms run extremely fast but waste significant surface area whenever a rectangle's height does not match the shelf ceiling [S2].

## Operating characteristics
*   **Time Complexity:** The greedy partitioning step is linear with respect to the number of seed voxels, as each voxel is visited and marked once. The 2D packing step is NP-Hard in the general case, but the greedy heuristic (sorting by area and evaluating four placements) provides a practical approximation suitable for real-time systems [S1, unofficial][S2].
*   **Memory Complexity:** Memory usage is proportional to the number of non-empty voxels and the resulting bounding boxes, significantly less than a full regular grid or AMR structure for sparse volumes [S1, unofficial].
*   **Measured Performance:** On a 128x128 slice, the method condensed the original slice into a smaller dimension (e.g., 54x196), demonstrating effective space compression [S1, unofficial].
*   **Strengths:** Bypasses 3D texture padding waste; avoids complex decoding overheads of out-of-core streaming; streamlined process using only alpha values [S1, unofficial].
*   **Limitations:** The 2D packing is an approximation; the choice of $T_{density}$ and $T_{sparsity}$ requires tuning; the greedy expansion may not produce globally optimal box shapes [S1, unofficial].
*   **When to use:** Suitable for sparse 3D volumes where memory bandwidth is a bottleneck and real-time performance is required [S1, unofficial].
*   **When not to use:** Not suitable for dense volumes where the overhead of packing may outweigh the benefits, or where exact optimal packing is required (though exact solvers are too slow) [S2].

## Background
The Greedy-based Box Packing method was proposed in a paper titled "A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data" [S1, unofficial][S2]. The work addresses the limitations of AMR structures and out-of-core streaming in volume rendering. It builds on the observation that 2D rectangle packing is an NP-Hard problem and leverages the MAXRECTS-BSSF algorithm as a reference for efficient empty-space management [S2]. The method was developed to bridge the gap between theoretical 2D packing algorithms and practical 3D volume rendering pipelines [S1, unofficial].

## Sources
[S1] A Hybrid CPU-GPU Approach for Memory-Efficient Volume ..., https://www.researchgate.net/publication/408581180_A_Hybrid_CPU-GPU_Approach_for_Memory-Efficient_Volume_Rendering_of_Medical_Image_Data, fetched 2026-10-26 (unofficial)
[S2] A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data, https://doi.org/10.24132/csrn.2026-d53, fetched 2026-10-26
