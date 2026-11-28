# Decomposition

## Overview
Decomposition is an image-order volume rendering technique that converts a volumetric dataset into a collection of geometric primitives, such as spheres or slices, which are placed, scaled, and colored based on a data mapping [S1][S2, unofficial]. It treats the volume as a set of distinct geometric objects suitable for further processing or manipulation [S1][S2, unofficial].

## How it works
**Inputs:** A volumetric dataset (scalar field $V$) and a mapping function [S1][S2, unofficial].
**Outputs:** A 2D image buffer [G].

**Data Structures:**
The dataset is decomposed into a list of primitives $\mathcal{P} = \{p_1, p_2, \dots, p_n\}$ [S1][S2, unofficial]. Each primitive $p_i$ stores its geometric parameters (position $\mathbf{c}_i$, radius $r_i$ for spheres, or dimensions for slices) and visual attributes (color $\mathbf{c}_i$, opacity $\alpha_i$) [S1][S2, unofficial].

**Steps:**
1. **Decomposition:** Iterate through the volumetric dataset to generate a corresponding geometric primitive for each voxel or region [S1][S2, unofficial].
    - *Spheres:* Each voxel or cluster of voxels becomes a sphere. The radius $r_i$ and position $\mathbf{c}_i$ are determined by the voxel location and potentially the local density or gradient [G].
    - *Slices:* The volume is sliced along a specific axis (e.g., axial, coronal, sagittal). Each slice becomes a 2D texture or height map [S2, unofficial].
2. **Mapping:** Apply the transfer function to determine the visual attributes ($\mathbf{c}_i, \alpha_i$) for each primitive based on its underlying scalar value $s_i$ [S1][S2, unofficial].
3. **Placement:** Transform the primitives from object space to screen space using the Model-View-Projection (MVP) matrix as defined in [Primitive Geometry](6264-primitive-geometry.md) [G].
4. **Rasterization:** Render the primitives to the image buffer. Slices are rendered as pseudo-colored textures or height maps [S2, unofficial]. Spheres are rasterized as circles with appropriate shading or opacity blending [G].
5. **Compositing:** Accumulate the contributions of all primitives. For transparent primitives, standard front-to-back alpha compositing is used [G]:
   $$ C_{out} = C_{in} + \mathbf{c}_i \cdot \alpha_i \cdot (1 - \alpha_{in}) $$
   $$ \alpha_{out} = \alpha_{in} + \alpha_i \cdot (1 - \alpha_{in}) $$
   where $C_{in}$ and $\alpha_{in}$ are the accumulated color and opacity from previously processed primitives [G].

**Parameters:**
- **Primitive Type:** Sphere vs. Slice. Slices are commonly used for further processing like segmentation [S1][S2, unofficial].
- **Mapping Function:** The specific transfer function defining color/opacity vs. scalar value [S1][S2, unofficial].
- **Typical Values:** Not given in the sources.

**Edge Cases:**
- **Occlusion:** Overlapping primitives must be sorted or handled via depth buffering to ensure correct compositing order [G].
- **Artifacting:** Spherical primitives may introduce "stair-step" artifacts if the voxel-to-sphere mapping is not carefully tuned [G].

## Subtopics

### [Curve-Skeleton Decomposition](8486-curve-skeleton-decomposition.md)
Represents a volume as a 1D skeleton to enable interactive manipulation. Unlike direct volume deformation, it separates geometric abstraction from rendering by decomposing the volume into bounding cuboids along the skeleton, allowing independent transformation and rendering of segments. Choose it when fast, minimum-geometric-processing manipulation (bend, move, re-pose) is required.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** When the volume needs to be treated as a collection of distinct geometric objects for further processing or manipulation [S1][S2, unofficial].
- **When not to use:** Not given in the sources.

## Background
Decomposition methods were identified as one of the five main image-order volume rendering techniques in early surveys [S1][S2, unofficial]. They are particularly useful when the volume needs to be treated as a collection of distinct geometric objects for further processing or manipulation [S1][S2, unofficial].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
