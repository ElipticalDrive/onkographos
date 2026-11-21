# Texture slicing

Also called: volume slicing, hardware-assisted volume rendering, proxy geometry rendering

## Overview
Texture slicing is an object-order volume rendering technique that maps volumetric data into 2D or 3D textures and renders the volume as a series of parallel planes (proxy geometry) that slice through the texture data [S4, book]. It solves the problem of efficiently visualizing 3D scalar fields on graphics hardware by leveraging rasterization pipelines and hardware alpha blending, avoiding the computational cost of per-pixel ray casting [S3, book][S4, book]. It sets itself apart from siblings like [Splatting](23-splatting.md) by using explicit polygonal geometry for slices rather than point-based splats, and from [Ray casting](21-ray-casting.md) (image-order) by relying on hardware-accelerated scan-conversion and compositing rather than iterative sampling along view rays [S4, book].

## Shared principles
The texture slicing family shares the following mechanism, terms, and mathematical foundations:

1.  **Proxy Geometry:** The volume is intersected by a series of parallel planes. These planes are represented as geometric primitives (typically quads, triangles, or triangle strips) in screen space or object space. The geometry represents the "cut" through the volume [S1, unofficial][S2, unofficial][S3, book].
2.  **Intersection Computation:** The core algorithm involves calculating the intersection points between the slicing plane and the volume's bounding box or individual cells. For bounding box approaches, this involves solving line-plane intersection equations [S1, unofficial][S2, unofficial]. For cell-based approaches, it involves slicing individual grid cells [S1, S2, unofficial][S3, book].
3.  **Polygon Tessellation:** The resulting intersection points are sorted and tessellated into triangles or triangle strips suitable for rasterization. Sorting is often done by projecting points onto a 2D plane and computing a pseudo-angle to avoid trigonometric functions [S2, unofficial][S4, book].
4.  **Texture Mapping:** The generated polygons are textured using the volume data. This typically involves mapping the 3D position of the polygon vertices to 3D texture coordinates to sample the scalar field [S1, unofficial][S2, unofficial].
5.  **Hardware Compositing:** The fragments are composited into the frame buffer using hardware-supported alpha blending. The standard blending equation is:
    $$ C_{dest} = C_{src} \cdot A_{src} + C_{dest} \cdot (1 - A_{src}) $$
    where $C_{src}$ and $A_{src}$ are the color and opacity from the texture sample, and $C_{dest}$ is the current frame buffer color [S3, book]. This implements the discrete volume rendering integral when slices are processed in correct depth order [S3, book].
6.  **Depth Ordering:** To ensure correct transparency, slices must be rendered in back-to-front (or front-to-back with appropriate blending) order relative to the viewer [S1, unofficial][S3, S4, book].

## Subtopics

### [AR mapping to 3D texture](884-ar-mapping-to-3d-texture.md)
Uses a linear mapping for texture coordinates to allow parallel polygons to be legally rotated without recalculating coordinates, enabling the use of display lists for performance.

### [Alpha Dithering via Texture Map](888-alpha-dithering-via-texture-map.md)
Expands the effective range of allowable opacity values on graphics hardware restricted to 8 bits per channel by dynamically adjusting the transfer function stored in the texture map to compensate for slice spacing.

### [2D Texture-Based Volume Rendering](916-2d-texture-based-volume-rendering.md)
Decomposes the 3D scalar field into a set of axis-aligned 2D slices, storing each slice as a 2D texture, and compositing them back-to-front using hardware alpha blending. It trades mathematical accuracy for compatibility with older graphics hardware and reduced geometry generation overhead.

### [3-Pass Texture Mapping with Compositing](1354-3-pass-texture-mapping-with-compositing.md)
Accelerates geometric transformation by pre-computing scalar values into intermediate 3-D datasets, then re-sampling separately in three orthogonal directions to reduce computational expense.

### [2D Volume Cross-Section Rendering](1764-2d-volume-cross-section-rendering.md)
Extracts a single two-dimensional slice from a three-dimensional volume dataset by intersecting it with an arbitrarily positioned plane, rendering a single opaque plane without depth accumulation.

### [Slice Polygon Computation (3D Texture)](5680-slice-polygon-computation-3d-texture.md)
Generates proxy geometry (polygons) that intersect a 3D volume texture along planes perpendicular to the viewing direction, storing the entire volume in a single 3D texture object rather than replicating data into three 2D texture stacks.

### [Palette Texture Volume Rendering](6502-palette-texture-volume-rendering.md)
An acceleration technique for mobile devices that maps 8-bit indexed data to RGBA values via a palette, reducing texture memory footprint to approximately one-fourth of the original size.

### [View-Aligned Slicing with 3D Textures](6687-view-aligned-slicing-with-3d-textures.md)
Renders the volume as a series of parallel planes orthogonal to the current view direction, minimizing overdraw and maintaining consistent pixel coverage regardless of camera orientation.

### [Slice Geometry Generation](6883-slice-geometry-generation.md)
Procedurally creates proxy geometric primitives (polygons) that represent the intersection of slicing planes with a volumetric dataset, allowing for complex geometric operations like dynamic lighting.

### [Cabral Texture Mapping](7257-cabral-texture-mapping.md)
Not given in the sources.

#### Image-Aligned Kernel-Slicing
Proposed by Mueller and Crawfis to avoid popping artifacts when slice direction changes, using image-aligned kernel-slicing and traversal order [S5, unofficial]. However, the GPU implementation has high computational cost and needs future investigation [S5, unofficial].

#### 3D Texture Generation via Slicing
Generates a 3D texture by rendering 2D slices of a scene to a 2D texture, reading pixels, and assembling them into a 3D texture [S6, unofficial].

#### Spherical Coordinate Volume Rendering
A method for planet-scaled atmospheric data that avoids accuracy loss from Cartesian-to-geodetic coordinate conversion by operating directly in spherical coordinates [S7, unofficial].

#### Fiber Surface Texturing
Assigns each vertex a parameter $t$ in $[0,1]$ based on its position on the FSCP edge, mapping vertex parameters to global range $[0,1]$ for texture lookup, and storing a dotted line in the texture map where black values indicate a fiber to be drawn [S8].

## Background
Volume slicing was first described by Wilson et al. in 1994 [WGW94], using hardware-assisted 3D texture mapping for direct volume rendering. The method maps voxels through classification functions into RGBA values, stores them in a 3D texture map, and slices the texture with parallel planes [S1, unofficial]. Early graphics boards only supported 2D textures, requiring data replication in three axis directions to allow quick construction of geometry for the axis most aligned with the viewpoint [S4, book]. The advent of 3D texturing capabilities improved the method by allowing proxy geometry consisting of parallel polygons orthogonal to the viewing direction [S4, book]. The NVIDIA GeForce3 family, released in 2001, fulfilled the prediction that 3D texture mapping in hardware would become common, enabling interactive frame rates for volumes up to $256 \times 256 \times 256$ [S1, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] khizarsiddiqui/Volume-Rendering, https://github.com/khizarsiddiqui/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S3] Implementing volume rendering using 3D texture slicing - OpenGL - Build ..., https://www.oreilly.com/library/view/opengl-build/9781788296724/ch07s02.html, fetched 2026-10-27 (book)
[S4] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S5] Hardwareaccelerated adaptive ewa volume splatting, https://core.ac.uk/works/16783031, fetched 2026-10-28 (unofficial)
[S6] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
[S7] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
[S8] Fast and Exact Fiber Surfaces for Tetrahedral Meshes, https://hal.science/hal-01206120v2, fetched 2026-10-26
