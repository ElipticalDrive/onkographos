# Hand-rail explode gesture

Also called: hand-rail explode, sideways hand-rail movement

## Overview
The hand-rail explode gesture creates and manipulates exploded views of volume data by moving the interaction rail sideways, perpendicular to a previously defined cutting plane [S1, unofficial]. It builds on [Hand-rail cut gesture](22303-hand-rail-cut-gesture.md) by transforming a static partition into a separated view, allowing users to inspect internal structures without occlusion [S1, unofficial]. Unlike standard clipping which merely hides data, this gesture actively displaces the partitioned volume portion along a rotational axis determined by the user's movement direction [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Current hand rail position and orientation; hand movement direction vector $\hat{d}_{move}$; existing cut planes (e.g., from [Hand-rail cut gesture](22303-hand-rail-cut-gesture.md) or bimanual 'V'-shape cuts); optional density range selection for threshold-based segmentation.
*   **Output:** Transformed volume visualization where partitioned portions are rotated or translated to form an exploded view.

**Data Structures:**
*   **Cut State:** Stores the geometry of active cutting planes (normal vectors, distances) and the resulting partitioned volume subsets [S1, unofficial].
*   **Explode State:** Stores the current rotation or translation offset applied to each partitioned subset [S1, unofficial].
*   **Density Range:** A pair of scalar values defining the lower and upper bounds of voxel density to be preserved during the explode operation [S1, unofficial].

**Steps:**
1.  **Prerequisite:** Perform a cut gesture (e.g., [Hand-rail cut gesture](22303-hand-rail-cut-gesture.md)) to partition the volume data [S1, unofficial].
2.  **Detect Movement:** Track the hand rail's movement direction $\hat{d}_{move}$ in 3D space [S1, unofficial].
3.  **Determine Explosion Axis:**
    *   Identify all valid rotational axes available for exploding the current partitioned volume (e.g., axes perpendicular to cut plane normals) [S1, unofficial].
    *   Project each candidate rotational axis onto the 2D screen plane [S1, unofficial].
    *   Select the rotational axis whose 2D screen projection is most perpendicular to the hand movement direction $\hat{d}_{move}$ [S1, unofficial].
    *   *Note:* If bimanual input is used, apply this logic independently for each hand to create complex exploded views (e.g., opening a curtain) [S1, unofficial].
4.  **Apply Transformation:**
    *   Rotate the partitioned volume portion around the selected axis based on the magnitude of the hand movement [S1, unofficial].
    *   *Alternative (Parallel-plane exploding):* Instead of rotation, apply a translation gesture (e.g., five-finger pan) to the partitioned volume to explode its internal structures [S1, unofficial].
5.  **Threshold Filtering (Optional):**
    *   If a density range is specified, only voxels outside the selected density range are affected by the cut/exploding transformation; voxels within the range remain untouched [S1, unofficial].

**Formulas:**
*   Let $\vec{a}_i$ be the $i$-th candidate rotational axis.
*   Let $\vec{p}_i$ be the 2D screen projection of $\vec{a}_i$.
*   Let $\hat{d}_{move}$ be the normalized hand movement direction.
*   Select axis $k$ such that:
    $$ k = \arg \min_i | \vec{p}_i \cdot \hat{d}_{move} | $$
    (Minimizing the dot product maximizes perpendicularity) [S1, unofficial].

**Parameters:**
*   **Explode Magnitude:** Proportional to the distance the hand rail moves sideways [S1, unofficial].
*   **Density Range:** User-defined via a double-ended slider on the interface panel before or during the gesture [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Ambiguous Directions:** In complex cuts (e.g., 'V'-shape), multiple explosion directions may be possible. The perpendicularity rule resolves this [S1, unofficial].
*   **Precision:** Screen projection accuracy depends on camera orientation and resolution [G].
*   **Boundary Conditions:** If the hand movement is parallel to all candidate axes' projections, the selection may be unstable [G].

## Subtopics

### Parallel-plane exploding
A variant that creates exploded views by applying translation to partitioned volume portions instead of 3D rotation [S1, unofficial]. It is chosen when linear separation is preferred over rotational opening. Implementation involves cutting the volume and then applying a five-finger pan translation gesture on the partitioned volume to explode its internal structures [S1, unofficial]. It can also follow a 'V'-shape cut [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for axis selection (assuming fixed number of cut planes); $O(N)$ for transforming $N$ voxels in the partitioned subset [G].
*   **Memory Complexity:** $O(N)$ to store the transformed voxel positions or transformation matrices [G].
*   **Performance:** Real-time interaction is feasible on modern GPUs for typical volume resolutions [G].
*   **Strengths:** Intuitive mapping of hand movement to volume separation; supports bimanual interaction for complex views; integrates with threshold-based segmentation [S1, unofficial].
*   **Limitations:** Requires prior cutting; explosion direction is constrained by screen projection geometry [S1, unofficial].
*   **When to Use:** When users need to inspect internal structures of a partitioned volume without removing data.
*   **When Not to Use:** When the volume is not yet partitioned or when rotational separation is not desired.

## Background
Originated in the research on whole-hand multitouch gestures for volume visualization [S1, unofficial]. The technique was developed to provide an easy reversal of actions and flexible exploration of volume data on tabletop interfaces [S1, unofficial]. It is used in interactive volume visualization systems that support bimanual input and threshold-based segmentation [S1, unofficial].

## Sources
[S1] Exploring Volume Visualization with Whole-hand Multitouch Gestures, https://doi.org/10.2312/pe.pg.pg2013short.007-010, fetched 2026-10-26 (unofficial)
