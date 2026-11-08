# Slice Geometry Generation
Also called: Proxy Geometry Generation, Slice Polygon Generation

## Overview
Slice Geometry Generation is the procedural or pre-computed creation of proxy geometric primitives (typically polygons or triangle strips) that represent the intersection of slicing planes with a volumetric dataset. It solves the problem of mapping 3D scalar data to 2D screen pixels using hardware rasterization by converting volume intersections into explicit geometry [G]. It differs from pure texture slicing (which relies on hardware texture sampling of 3D textures or 2D stacks) by generating explicit polygonal meshes for each slice, allowing for more complex geometric operations like dynamic lighting and per-pixel normal estimation, while differing from ray casting by leveraging spatial coherence and hardware alpha blending [G].

## Shared principles
Not given in the sources. Builds on [Texture slicing](25-texture-slicing.md).

## Subtopics

### [Plane Slicing](845-plane-slicing.md)
Generates explicit polygons by slicing unstructured or curvilinear grid cells on evenly spaced planes. It differs from other methods by operating on individual cells rather than a bounding box, making it suitable for unstructured grids. Choose it when rendering unstructured data with high geometric fidelity.

### [GPU-based Cut-off Slice Generation](948-gpu-based-cut-off-slice-generation.md)
Computes proxy geometry (polygons) representing the intersection of slicing planes with the volume's bounding box directly on the GPU. It differs from CPU-based generation by offloading intersection and sorting to the vertex shader for real-time interactivity. Choose it for dynamic, interactive rotation of structured volumes.

### [Sliced Volume Rendering](3568-sliced-volume-rendering.md)
Generates slice geometry on the fly to enable real-time lighting and flexible transfer functions via GLSL. It differs from standard slicing by decoupling geometric generation from rasterization to allow per-pixel lighting calculations. Choose it when dynamic lighting effects are required.

### [Perpendicular Slice Extraction Geometry](7503-perpendicular-slice-extraction-geometry.md)
Computes the geometric properties of a slice perpendicular to a specific axis at a given index. It sets itself apart by enforcing strict axis-aligned perpendicularity and handling the coordinate transformation between RAS world space and IJK volume index space explicitly. Choose it when precise axis-aligned slice extraction with explicit coordinate mapping is needed.

## Background
The concept of slicing volumes into 2D cross-sections for rendering dates back to the early days of volume visualization [G]. Plane Slicing for unstructured grids was introduced by Levoy in 1990 [G]. GPU-based cut-off slice generation techniques were developed to leverage the increasing power of graphics hardware for real-time interaction [G]. Sliced Volume Rendering with dynamic lighting emerged with the advent of programmable shaders (GLSL), allowing for more physically based rendering effects [G].

## Sources
[G] General graphics and volume rendering knowledge.
