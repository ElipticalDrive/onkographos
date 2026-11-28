# 3D Texture Slicing

Also called: Volumetric Texture Slicing, 3D Layer Slicing

## Overview
3D Texture Slicing is a family of object-order volume rendering techniques that represent volumetric data as a series of parallel planar slices textured with a 3D texture map . It solves the problem of efficiently rendering dense, heterogeneous volumetric layers (such as vegetation, fur, or continuous canopies) by leveraging hardware texture filtering for smooth antialiasing and correct parallax, which polygonal representations lack . It sets itself apart from sibling methods by generating fragments that are "really 3D" at their proper depth, allowing for natural surface-conforming geometry without forcing slices to face the camera as in view-aligned slicing .

## Shared principles
The 3D Texture Slicing family shares the following mechanism, terms, and mathematical foundations:

1.  **Proxy Geometry:** The volume is intersected by a series of parallel planes . These planes are represented as geometric primitives (typically quads, triangles, or triangle strips) in screen space or object space . The geometry represents the "cut" through the volume [G].
2.  **Intersection Computation:** The core algorithm involves calculating the intersection points between the slicing plane and the volume's bounding box or individual cells [G]. For bounding box approaches, this involves solving line-plane intersection equations [G]. For cell-based approaches, it involves slicing individual grid cells .
3.  **Polygon Tessellation:** The resulting intersection points are sorted and tessellated into triangles or triangle strips suitable for rasterization . Sorting is often done by projecting points onto a 2D plane and computing a pseudo-angle to avoid trigonometric functions .
4.  **Texture Mapping:** The generated polygons are textured using the volume data . Texture coordinates are specified at the polygon corners and interpolated across the plane by the graphics hardware to sample the scalar field .
5.  **Hardware Compositing:** The fragments are composited into the frame buffer using hardware-supported alpha blending . The standard blending equation is:
    $$ C_{dest} = C_{src} \cdot A_{src} + C_{dest} \cdot (1 - A_{src}) $$
    where $C_{src}$ and $A_{src}$ are the color and opacity from the texture sample, and $C_{dest}$ is the current frame buffer color [G]. This implements the discrete volume rendering integral when slices are processed in correct depth order .
6.  **Depth Ordering:** To ensure correct transparency, slices must be rendered in back-to-front (or front-to-back with appropriate blending) order relative to the viewer .

## Subtopics

### [Volumetric textures with textured slices](9800-volumetric-textures-with-textured-slices.md)
Renders a 3D layer (e.g., vegetation) on a surface by mapping a 3D data set as a texture pattern onto parallel slices. It provides perfect parallax and leverages hardware filtering for antialiasing, unlike polygonal data. It differs from view-aligned slicing by not forcing slices to face the camera, allowing surface-conforming geometry but requiring viewpoint adaptation to prevent gaps.

### [3D Texture Generation](10399-3d-texture-generation.md)
Encompasses methods for storing volumetric scalar field data directly into a 3D texture object, allowing the GPU to perform volume rendering operations via hardware-accelerated texture lookups. It focuses on the fundamental mechanism of populating a 3D texture array for direct hardware sampling, offloading composition and interpolation from the CPU.

### [Hardware-Accelerated Texture Mapping](21070-hardware-accelerated-texture-mapping.md)
Exploits GPUs to achieve interactive frame rates for structured grids by utilizing hardware parallel logic, cache coherency, and native support for trilinear interpolation and alpha compositing. It differs from Shear-Warp by avoiding the final warp step and from Custom Hardware Units by using general-purpose programmable GPUs [G].

## Background
The technique originated in the late 1990s as a method to render complex volumetric phenomena like fur and vegetation efficiently on available graphics hardware . It was developed to address the limitations of polygonal rendering for dense, fine-grained structures and the high computational cost of pure ray casting . The approach relies on the maturity of 3D texture mapping and hardware alpha blending capabilities in graphics pipelines .

