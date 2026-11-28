# Jiang et al. Acceleration Structure
Also called: JRZ*16 method

## Overview
The Jiang et al. acceleration structure [S1] is a hybrid object-order and image-order method designed to efficiently locate active cells for both direct and indirect volume visualization [S1]. It differs from pure image-order ray casting by leveraging hierarchical structures to skip empty space, while remaining dependent on rendering resolution like other hybrid approaches [S1].

## How it works
**Inputs:** Scalar volume data; rendering resolution; ray origin and direction.
**Outputs:** Rendered image with active cells rendered via ray marching [S1].

The method constructs a hierarchical acceleration structure to identify active cells (non-empty or relevant data regions) [S1]. The traversal logic follows the shared principles of hierarchical descent [G].

1.  **Structure Construction:** Build a hierarchical data structure (e.g., octree) from the volume data, marking nodes that contain active cells [S1].
2.  **Ray Traversal:** For each pixel, cast a ray into the volume. Traverse the hierarchy starting from the root [G].
3.  **Active Cell Identification:** Use the structure to skip empty subtrees, identifying only the active cells intersected by the ray [S1].
4.  **Ray Marching:** Once active cells are located, perform ray marching within those regions to compute the final color [S1].

**Parameters:**
*   Hierarchy depth: Not given in the sources.
*   Threshold for "active": Not given in the sources.

**Edge cases:**
*   Resolution dependence: As a hybrid approach, performance and behavior depend on the rendering resolution [S1].

## Subtopics

### IsoBAS Acceleration Structure
An acceleration structure introduced by Liu et al. [LCD15] for faster identification of active cells with less memory overhead than previous methods [S1]. It is cited as a precursor or related work to the Jiang et al. method in the context of hybrid approaches [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Efficiently locates active cells for direct and indirect visualization [S1].
*   **Limitations:** Dependent on rendering resolution, classifying it as a hybrid object-order and image-order approach [S1].
*   **Use cases:** Direct and indirect volume visualization where active cell identification is a bottleneck [S1].

## Background
The method was proposed by Jiang et al. [JRZ*16] as part of research into efficient volume rendering techniques [S1]. It is situated in the context of hybrid rendering approaches that combine object-order preprocessing with image-order rendering [S1].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
