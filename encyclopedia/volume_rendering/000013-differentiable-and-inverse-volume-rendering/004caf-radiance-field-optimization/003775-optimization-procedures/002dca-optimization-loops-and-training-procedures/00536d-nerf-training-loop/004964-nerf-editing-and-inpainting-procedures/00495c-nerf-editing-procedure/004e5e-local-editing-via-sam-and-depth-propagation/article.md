# Local Editing via SAM and Depth Propagation

Also called: Depth-propagated Mask Editing, SAM-based NeRF Editing

## Overview
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information [S1]. It solves the problem where existing methods induce simultaneous changes in both background and foreground, contradicting user intentions to alter distinct objects [S1]. It sets itself apart by using depth-based reprojection to ensure multi-view consistency of the segmentation mask, allowing exclusive modification of the target instance while leaving the background intact [S1].

## How it works
**Inputs:** A trained NeRF, a set of input images (views), depth maps for each view, and a user interaction point on a target object in one or more views.
**Outputs:** A modified NeRF where the target object is edited according to a prompt, while the background remains unchanged.

**Data Structures:**
*   **Segmentation Mask ($M$):** A 2D binary mask per view, where $M_{i,j} = 1$ if pixel $(i,j)$ belongs to the target object, $0$ otherwise.
*   **Depth Map ($D$):** A 2D map per view, where $D_{i,j}$ is the depth value at pixel $(i,j)$.
*   **Camera Parameters:** Intrinsic matrix $K$ and extrinsic matrix $[R|t]$ for each view.

**Steps:**
1.  **Initial Segmentation:** Employ the Segment Anything Model (SAM) to derive masks encompassing all components within each image [S1].
2.  **User Selection:** Users pinpoint the target instance with a single point to isolate it from the SAM output [S1].
3.  **Key View Selection:** Designate nearly all views as key views to facilitate propagation [S1].
4.  **Depth-Based Projection:** Utilize depth information to project the identified mask from a source view to a target view progressively [S1].
    *   For a pixel $(u, v)$ in the source view with depth $d$, compute the 3D point $P$ in camera coordinates: $P = d \cdot K^{-1} \begin{bmatrix} u \\ v \\ 1 \end{bmatrix}$.
    *   Transform $P$ to the target camera coordinate system: $P' = R_{t,s} P + t_{t,s}$.
    *   Project $P'$ to the target image plane: $(u', v') = K P'_{xy} / P'_z$.
    *   Set the mask value at $(u', v')$ in the target view based on the source mask value.
5.  **Reprojection Check:** Continue the projection process as long as the projected mask sufficiently overlaps with the corresponding mask in the current view [S1]. If the overlap is insufficient, the propagation for that pixel or region may be halted or weighted down to maintain consistency [S1].
6.  **Mask Integration:** Integrate the wrapped segmentation mask with the segmented masks in the current view [S1]. This results in a multi-view consistent segmentation mask set.
7.  **Editing:** Apply the editing operation (e.g., via text-to-image diffusion or other NeRF editing techniques) constrained by the integrated masks to modify only the target object [S1].

**Parameters:**
*   **Overlap Threshold:** The threshold for "sufficient overlap" is not explicitly defined in the sources [S1].
*   **Key View Density:** "Nearly all views" are designated as key views, which leads to reduced efficiency compared to sparse key view selection [S1].

**Edge Cases and Pitfalls:**
*   **Incorrect Depth:** The model can manage incorrect depth estimation as long as the NeRF is built successfully, demonstrated on glossy surfaces and sparser views [S1].
*   **Background Leakage:** Without this method, edits often affect the background [S1].

## Subtopics

### MaskEditor
A grid-based 3D editing method using a 3D mask grid trained on 2D masks from SAM [S2]. It introduces a 3D mask grid trained on the 2D masks from SAM to locate the target object accurately [S2]. Instead of utilizing SDS, it employs Variational Score Distillation (VSD) along with composited rendering and a coarse-to-fine editing strategy to improve editing quality [S2].

## Operating characteristics
*   **Efficiency:** Designating nearly all views as key views leads to reduced efficiency [S1].
*   **Robustness:** The method demonstrates robustness to incorrect depth estimation, handling glossy surfaces and sparse input views (e.g., 20% or 50% of views) effectively [S1].
*   **Consistency:** It achieves multi-view consistent segmentation, permitting exclusive modification of the human figure or other objects without changing the background [S1].
*   **Limitations:** The reliance on SAM for initial segmentation and the dense key view selection may limit scalability to very large scenes or high-resolution inputs [S1].

## Background
Originated in ViCA-NeRF (View-Consistency-Aware 3D Editing of Neural Radiance Fields) [S1]. The method addresses the discrepancy in existing methods that fail to isolate foreground edits from background changes [S1]. It builds on the Segment Anything Model (SAM) [S1] and leverages depth information for view consistency.

## Sources
[S1] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
