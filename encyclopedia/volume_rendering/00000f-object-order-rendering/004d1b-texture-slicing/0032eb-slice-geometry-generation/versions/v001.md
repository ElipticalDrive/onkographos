# Slice Geometry Generation
Also called: Proxy Geometry Generation, Slice Polygon Generation

## Overview
Slice Geometry Generation is the procedural or pre-computed creation of proxy geometric primitives (typically polygons or triangle strips) that represent the intersection of slicing planes with a volumetric dataset. It solves the problem of how to map 3D scalar data to 2D screen pixels using hardware rasterization by converting volume intersections into explicit geometry [S1, unofficial][S2, book][S3, unofficial]. It differs from pure texture slicing (which relies on hardware texture sampling of 3D textures or 2D stacks) by generating explicit polygonal meshes for each slice, allowing for more complex geometric operations like dynamic lighting and per-pixel normal estimation, while differing from ray casting by leveraging spatial coherence and hardware alpha blending [S1, S2][S3, unofficial].

## Shared principles
The Slice Geometry Generation family shares the following mechanism, terms, and mathematical foundations:

1.  **Proxy Geometry:** The volume is intersected by a series of parallel planes. These planes are represented as geometric primitives (typically quads, triangles, or triangle strips) in screen space or object space. The geometry represents the "cut" through the volume [S1, unofficial][S2, book][S3, unofficial].
2.  **Intersection Computation:** The core algorithm involves calculating the intersection points between the slicing plane and the volume's bounding box or individual cells. For bounding box approaches, this involves solving line-plane intersection equations [S1, unofficial][S2, book]. For cell-based approaches, it involves slicing individual grid cells [S1, S2][S3, unofficial].
3.  **Polygon Tessellation:** The resulting intersection points are sorted and tessellated into triangles or triangle strips suitable for rasterization. Sorting is often done by projecting points onto a 2D plane and computing a pseudo-angle to avoid trigonometric functions [S2, book][S4].
4.  **Texture Mapping:** The generated polygons are textured using the volume data. This typically involves mapping the 3D position of the polygon vertices to 3D texture coordinates to sample the scalar field [S1, unofficial][S2, book].
5.  **Hardware Compositing:** The fragments are composited into the frame buffer using hardware-supported alpha blending. The standard blending equation is:
    $$ C_{dest} = C_{src} \cdot A_{src} + C_{dest} \cdot (1 - A_{src}) $$
    where $C_{src}$ and $A_{src}$ are the color and opacity from the texture sample, and $C_{dest}$ is the current frame buffer color [S3, book]. This implements the discrete volume rendering integral when slices are processed in correct depth order [S3, book].
6.  **Depth Ordering:** To ensure correct transparency, slices must be rendered in back-to-front (or front-to-back with appropriate blending) order relative to the viewer [S1, unofficial][S3, S4, book].

## Subtopics

### [Plane Slicing](845-plane-slicing.md)
Generates explicit polygons by slicing unstructured or curvilinear grid cells on evenly spaced planes. It differs from other methods by operating on individual cells rather than a bounding box, making it suitable for unstructured grids. Choose it when rendering unstructured data with high geometric fidelity.

### [GPU-based Cut-off Slice Generation](948-gpu-based-cut-off-slice-generation.md)
Computes proxy geometry (polygons) representing the intersection of slicing planes with the volume's bounding box directly on the GPU. It differs from CPU-based generation by offloading intersection and sorting to the vertex shader for real-time interactivity. Choose it for dynamic, interactive rotation of structured volumes.

### [Sliced Volume Rendering](3568-sliced-volume-rendering.md)
Generates slice geometry on the fly to enable real-time lighting and flexible transfer functions via GLSL. It differs from standard slicing by decoupling geometric generation from rasterization to allow per-pixel lighting calculations. Choose it when dynamic lighting effects are required.

## Background
The concept of slicing volumes into 2D cross-sections for rendering dates back to the early days of volume visualization. Plane Slicing for unstructured grids was introduced by Levoy in 1990 [S1, S2][S3, unofficial]. GPU-based cut-off slice generation techniques were developed to leverage the increasing power of graphics hardware for real-time interaction, with specific algorithms detailed in graphics programming books [S2, book][S4]. Sliced Volume Rendering with dynamic lighting emerged with the advent of programmable shaders (GLSL), allowing for more physically based rendering effects [S1, unofficial].

## Sources
[S1] vl::SlicedVolume implementation notes, URL, date fetched
[S2] Graphics Programming Book, URL, date fetched
[S3] Unofficial notes on Volume Rendering, URL, date fetched
[S4] Graphics Programming Book, URL, date fetched
[S5] Unofficial notes on Alpha Blending, URL, date fetched
