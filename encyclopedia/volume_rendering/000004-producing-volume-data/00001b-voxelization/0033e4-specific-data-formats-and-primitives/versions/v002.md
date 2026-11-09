# Specific Data Formats and Primitives

Also called: Volume Graphics, Hypertexture Mesh Amplification

## Overview
This family encompasses specific volumetric data structures and primitive representations used during or after the voxelization process, ranging from hardware-accelerated binary encodings to out-of-core storage formats and feature tensors. It solves the problem of efficiently storing, accessing, and rendering complex volumetric data by defining how geometric or scalar information is mapped into discrete or analytic primitives. Unlike general voxelization algorithms which focus on the conversion of geometry to voxels, this family focuses on the resulting data format (e.g., bit-packed frame buffers, Morton-ordered files, or feature tensors) and specialized primitives (e.g., CSG classification maps, point-based contours).

## Shared principles
The methods in this family share the following mechanisms for representing volumetric data:
1.  **Encoding**: Mapping continuous or logical properties (inside/outside, scalar value, feature vector) into discrete data structures (bitmasks, color indices, tensors).
2.  **Access Optimization**: Using spatial indexing (e.g., Morton order) or hardware look-up tables (e.g., PCM) to enable fast retrieval or classification without iterating through all voxels.
3.  **Primitive Abstraction**: Treating voxels not just as scalar values but as carriers of additional attributes (normals, textures, CSG bits) or as part of larger logical structures (CSG trees, point clouds).

## Subtopics

### [CSG Model Voxelization](2497-csg-model-voxelization.md)
CSG Model Voxelization is a hardware-accelerated algorithm that generates a binary voxelization of a Constructive Solid Geometry (CSG) model by encoding primitive classifications into frame buffer pixel colors and using logical operations to compute the final CSG classification implicitly via a Point-Classification Map (PCM). It solves the problem of efficiently converting complex CSG trees into discrete volumetric data for applications like volume rendering and layered manufacturing. Unlike previous hardware-accelerated methods that required intermediate textures for each Boolean operation node, this method computes boolean operations in image space using frame buffer pixel maps, resulting in faster performance and avoiding hardware restrictions on blending function combinations. It operates slice-by-slice (Z-plane by Z-plane), rendering primitives into a frame buffer where each bit of the pixel color represents the inclusion of a specific primitive.

### [Out-of-Core Point-Based Rendering](6825-out-of-core-point-based-rendering.md)
Out-of-core point-based rendering is a method for the interactive visualization of very large volumetric datasets that exceed available main memory. It solves the problem of real-time interaction with high-resolution data by assuming that voxel density is sufficient to represent isovalue structures (contours) using discrete points rather than continuous meshes or full volume textures. Unlike mesh-based methods which require connectivity and topology information, this approach renders points directly, allowing for full-resolution contour rendering at interactive frame rates without the computational overhead of surface extraction. It builds on [Specific Data Formats and Primitives](7148-specific-data-formats-and-primitives.md) by utilizing out-of-core storage and point-based primitives.

### [Point Feature Generation](6972-point-feature-generation.md)
Point Feature Generation constructs the initial feature tensors for ray-based volume rendering by indexing spatial data and integrating image-derived attributes. It differs from sibling methods by explicitly separating general input features from color volume data, allowing for flexible integration of auxiliary image features alongside standard geometric ray points.

#### Hypertexture Mesh Amplification
**Usage**: Used to define a volume by the interior of a mesh. Currently, animation is a big problem for this volume data type [S1, unofficial].

#### Volume Graphics
**Definition**: An approach concerned with the synthesis, manipulation, and rendering of volumetric geometric objects, where geometric models are voxelized (3D scan-converted) into a set of voxels stored in a volume buffer with pre-computed view-independent attributes [S2].
**Steps**:
1.  Voxelize the geometric model (3D scan-convert) into a set of voxels that best approximate the model [S2].
2.  Store each voxel in the volume buffer along with pre-computed view-independent attributes such as texture, anti-aliasing, and normal vector [S2].
**Parameters**: Attributes: Pre-computed view-independent attributes stored per voxel, including texture, anti-aliasing, and normal vector [S2].
**Comparison**: Volume graphics focuses on modeled geometric scenes represented in a volume buffer, whereas volume visualization focuses on sampled and computed datasets [S2].

## Background
The concept of "Volume Graphics" was introduced by Kaufman et al. in 1993 as an emerging approach to synthesize, manipulate, and render volumetric geometric objects [S2]. It distinguishes itself from volume visualization by focusing on modeled geometric scenes rather than sampled datasets [S2]. The method involves 3D scan-converting geometric models into voxels stored in a volume buffer with pre-computed attributes [S2]. This approach offers advantages over surface graphics, including viewpoint independence and suitability for constructive solid modeling [S2].

The use of specific data formats like Morton-ordered out-of-core files for point-based rendering and bit-packed frame buffers for CSG voxelization represents later developments in handling large-scale or complex volumetric data, as seen in works such as "Ray-Guided Streaming for Efficient and Detailed Voxel Rendering" [S1, unofficial].

## Sources
[S1] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S2] Volume visualization, https://dl.acm.org/doi/pdf/10.1145/234313.234383, fetched 2026-11-01
