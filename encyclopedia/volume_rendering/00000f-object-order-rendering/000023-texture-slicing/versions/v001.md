# Texture slicing

Also called: Volume slicing, 3D texture slicing, proxy geometry slicing

## Overview
Texture slicing is a hardware-accelerated object-order volume rendering technique that approximates the volume-density function by mapping volumetric data into 2D or 3D textures and slicing the volume with parallel planes (proxy geometry) [S1, unofficial][S3, book]. These slices are rasterized and composited back-to-front (or front-to-back) using hardware-supported blending, enabling real-time interactivity by offloading the rendering pipeline to the GPU [S1, unofficial][S2, book]. It is distinguished from its siblings by its reliance on texture hardware for both data storage and sampling, whereas Splatting uses point primitives, Shear-warp uses affine transformations, Cell projection renders explicit polygons, Voxel rasterization treats voxels as geometry, and Domain-transform methods operate in frequency or transformed spaces [S3, book].

## Shared principles
The family shares a common mechanism: volumetric data is represented as a texture (2D or 3D) in graphics hardware, and the volume is sampled by intersecting it with a set of parallel planes [S1, unofficial][S3, book].

1.  **Data Representation**: The scalar volume data is mapped to RGBA values via classification functions and stored in a texture map [S1, unofficial]. For 3D texture hardware, the data is stored directly in a 3D texture [S1, unofficial]. For 2D texture hardware, data is represented as stacks of 2D images, often replicated in three axis directions to allow quick construction of geometry for the axis most aligned with the viewpoint [S3, book].
2.  **Proxy Geometry**: A set of parallel planes (proxy geometry) slices the volume [S1, unofficial][S3, book].
    *   In 3D texture slicing, the proxy geometry consists of parallel polygons orthogonal to the viewing direction (aligned with the image plane) [S3, book].
    *   In 2D texture slicing, the proxy geometry consists of axis-aligned quadrilaterals [S3, book].
3.  **Rendering Order**: The slices are rendered in back-to-front or front-to-back order to correctly accumulate color and opacity [S1, unofficial][S2, S3, book].
4.  **Compositing**: Hardware-supported blending is used to composite the proxy slices into the final image [S2, book]. The color and opacity contributions are accumulated during rasterization [S1, unofficial].
5.  **Efficiency**: The method relies on rasterization hardware, making it very fast on modern GPUs [S2, book]. It is an object-space approach, efficient for modern graphics processors, making it the choice for interactivity [S3, book].

## Subtopics

### [2D Texture-Based Volume Rendering](916-2d-texture-based-volume-rendering.md)
A GPU rendering method where volumetric data is stored in 2D texture images, projected onto the image plane as object-aligned slices, and blended back-to-front. It differs from 3D texture slicing by using 2D textures and axis-aligned quadrilaterals, often requiring data replication.

### [GPU-based Cut-off Slice Generation](948-gpu-based-cut-off-slice-generation.md)
An algorithm to calculate intersection polygons between a slicing plane and the bounding box of volumetric data, implemented in a vertex shader to offload CPU calculation. It optimizes the generation of the proxy geometry.

## Background
Texture slicing was described in 1994 by Wilson et al. as a method using hardware-assisted 3D texture mapping for direct volume rendering [S1, unofficial][S3, book]. It was also independently described by Cullip and Neumann, Cabral et al., and Guan and Lipes in 1993-1994 [S3, book]. The method became widely adopted as 3D texturing hardware became available in consumer-grade graphics accelerators, such as the NVIDIA GeForce3 family released in 2001 [S1, unofficial]. It is used in applications where interactivity is critical, such as segmentation and medical visualization [S1, unofficial].

## Sources

[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] Implementing volume rendering using 3D texture slicing - OpenGL - Build ..., https://www.oreilly.com/library/view/opengl-build/9781788296724/ch07s02.html, fetched 2026-10-27 (book)
[S3] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S4] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
