# Decomposition

Also called: X-Ray Decomposition, Primitive Decomposition

## Overview
Decomposition is a volume rendering technique that converts a volumetric data set into explicit geometric primitives, such as spheres or slices, which are then placed, scaled, and colored in space based on a data mapping [S1, unofficial]. It is distinguished from ray-based methods (like MIP or Iso-surface rendering) by its object-order approach, where the volume is explicitly discretized into renderable shapes rather than sampled along rays [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric data set (scalar field) and a mapping function (transfer function) that assigns color and opacity to data values.
*   **Output:** A rasterized image composed of the accumulated contributions of the generated geometric primitives.

**Data Structures:**
The method generates a list of geometric primitives. Each primitive is defined by:
1.  **Geometry:** A shape (e.g., sphere center/radius, or slice plane/extent).
2.  **Attributes:** Position, scale, color, and opacity derived from the underlying data value at the primitive's location.

**Steps:**
1.  **Primitive Generation:** Iterate through the volume data. For each data element (voxel or region), generate a corresponding geometric primitive. Common choices include spheres for point data or slices for planar data [S1, unofficial].
2.  **Placement and Transformation:** Place the primitives in 3D space. Transform them using the Model-View-Projection (MVP) matrix to map them to screen space [S1, unofficial].
3.  **Scaling and Coloring:** Scale the primitives based on the data value and the desired footprint size. Assign color and opacity using the transfer function [S1, unofficial].
4.  **Rasterization:** Rasterize the transformed primitives. For spheres, this often involves rendering point sprites or billboards. For slices, this involves rendering textured quads [S1, unofficial].
5.  **Blending:** Accumulate the contributions of the primitives using alpha blending to produce the final image [S1, unofficial].

**Parameters:**
*   **Primitive Type:** Sphere, slice, etc. [S1, unofficial].
*   **Mapping Function:** Transfer function parameters for color and opacity [S1, unofficial].
*   **Scale Factor:** Determines the size of the primitives relative to the voxel grid [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Occlusion:** Since primitives are explicit, occlusion handling depends on the rasterization order and blending mode. Depth sorting may be required for correct transparency [S1, unofficial].
*   **Artifacting:** Large primitives may cause aliasing or visual artifacts if not properly filtered or if the resolution is insufficient [S1, unofficial].

## Subtopics

### [Curve-Skeleton Decomposition](8486-curve-skeleton-decomposition.md)
Represents volumetric objects as 1D skeletons for interactive manipulation.

## Operating characteristics
*   **Time Complexity:** Depends on the number of primitives generated and the cost of rasterizing each primitive. Generally linear with respect to the number of voxels if one primitive per voxel is used.
*   **Memory Complexity:** Proportional to the number of primitives stored in the scene graph or vertex buffer.
*   **Strengths:** Allows for easy manipulation of individual volume elements (e.g., moving, scaling, recoloring) [S1, unofficial]. Slices can be used for further processing like segmentation [S1, unofficial].
*   **Limitations:** May suffer from aliasing if primitives are too large or too small relative to the screen resolution. Transparency handling can be complex [S1, unofficial].
*   **When to use:** When interactive manipulation of volume elements is required, or when the data has a natural decomposition into primitives (e.g., particles, planar slices) [S1, unofficial].

## Background
Decomposition is one of the five volume rendering techniques that operate in image order, alongside MIP, X-Ray, and Iso-surface rendering [S1, unofficial]. It was described in the context of interactive volume rendering techniques [S1, unofficial].

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
