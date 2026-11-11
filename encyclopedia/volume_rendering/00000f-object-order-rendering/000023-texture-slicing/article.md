# Texture slicing

Also called: volume slicing, hardware-assisted volume rendering, proxy geometry rendering

## Overview
Texture slicing is an object-order volume rendering technique that maps volumetric data into 2D or 3D textures and renders the volume as a series of parallel planes (proxy geometry) that slice through the texture data [S4, book]. It solves the problem of efficiently visualizing 3D scalar fields on graphics hardware by leveraging rasterization pipelines and hardware alpha blending, avoiding the computational cost of per-pixel ray casting [S3, book][S4, book]. It sets itself apart from siblings like [Splatting](23-splatting.md) by using explicit polygonal geometry for slices rather than point-based splats, and from [Ray casting](21-ray-casting.md) (image-order) by relying on hardware-accelerated scan-conversion and compositing rather than iterative sampling along view rays [G].

## Shared principles
The texture slicing family shares the following mechanism, terms, and mathematical foundations:

1.  **Proxy Geometry:** The volume is intersected by a series of parallel planes. These planes are represented as geometric primitives (typically quads, triangles, or triangle strips) in screen space or object space. The geometry represents the "cut" through the volume [S1, unofficial][S3, book].
2.  **Intersection Computation:** The core algorithm involves calculating the intersection points between the slicing plane and the volume's bounding box or individual cells [G]. For bounding box approaches, this involves solving line-plane intersection equations [G]. For cell-based approaches, it involves slicing individual grid cells [S1, unofficial][S3, book].
3.  **Polygon Tessellation:** The resulting intersection points are sorted and tessellated into triangles or triangle strips suitable for rasterization. Sorting is often done by projecting points onto a 2D plane and computing a pseudo-angle to avoid trigonometric functions [S2, unofficial][S4, book].
4.  **Texture Mapping:** The generated polygons are textured using the volume data. Texture coordinates are specified at the polygon corners and interpolated across the plane by the graphics hardware to sample the scalar field [S1, unofficial].
5.  **Hardware Compositing:** The fragments are composited into the frame buffer using hardware-supported alpha blending. The standard blending equation is:
    $$ C_{dest} = C_{src} \cdot A_{src} + C_{dest} \cdot (1 - A_{src}) $$
    where $C_{src}$ and $A_{src}$ are the color and opacity from the texture sample, and $C_{dest}$ is the current frame buffer color [G]. This implements the discrete volume rendering integral when slices are processed in correct depth order [S3, book].
6.  **Depth Ordering:** To ensure correct transparency, slices must be rendered in back-to-front (or front-to-back with appropriate blending) order relative to the viewer [S1, unofficial][S3, book][S4, book].

## Subtopics

### [View-Aligned Slicing with 3D Textures](870-view-aligned-slicing-with-3d-textures.md)
View-aligned slicing with 3D textures is a direct volume rendering method that represents a volumetric dataset as a single 3D texture object, intersected by a stack of parallel proxy planes oriented perpendicular to the viewer's line of sight. It solves the sampling artifacts and orientation limitations of axis-aligned 2D texture slicing by maintaining a constant Euclidean distance between slices along any light ray, regardless of viewing angle. Unlike ray casting, it offloads integration and interpolation to hardware rasterization, enabling interactive frame rates for moderate-sized volumes, though it cannot skip unseen voxels and is limited by texture memory capacity.

### [Alpha Dithering via Texture Map](888-alpha-dithering-via-texture-map.md)
Alpha Dithering via Texture Map is an optimization technique for texture-sliced volume rendering that expands the effective range of allowable opacity values on graphics hardware restricted to 8 bits per channel. It solves the problem where typical scientific volume renderings require very low per-slice opacities to maintain transparency; as slice density increases, these values often become too small to register on standard 8-bit hardware, causing visual artifacts or loss of detail. It sets itself apart from standard texture slicing by dynamically adjusting the transfer function stored in the texture map to compensate for slice spacing, rather than relying solely on fixed hardware blending.

### [3-Pass Texture Mapping with Compositing](1354-3-pass-texture-mapping-with-compositing.md)
This method accelerates the geometric transformation of volume data by pre-computing scalar values into intermediate 3-D datasets, thereby simplifying the image generation problem to the geometric transformation of two values per voxel (color and opacity) followed by compositing. It distinguishes itself from standard 2-pass texture mapping by re-sampling separately in three orthogonal directions to reduce computational expense, allowing for rapid interactive manipulation of volume data.

### [2D Volume Cross-Section Rendering](1764-2d-volume-cross-section-rendering.md)
2D Volume Cross-Section Rendering extracts a single two-dimensional slice from a three-dimensional volume dataset by intersecting it with an arbitrarily positioned plane. It solves the problem of visualizing internal structures in specific anatomical or geometric orientations (coronal, sagittal, or oblique) without the computational overhead of full volume rendering. Builds on [Texture slicing](25-texture-slicing.md), this method renders a single opaque plane, distinguishing it from siblings like [3-Pass Texture Mapping with Compositing](1354-3-pass-texture-mapping-with-compositing.md) which handles depth accumulation.

### [Slice Polygon Computation (3D Texture)](5680-slice-polygon-computation-3d-texture.md)
Slice Polygon Computation (3D Texture) is a texture-based volume rendering method that generates proxy geometry by intersecting slicing planes with the volume's bounding box, then mapping the resulting polygons using 3D texture coordinates. It differs from sibling methods like [View-Aligned Slicing with 3D Textures](870-view-aligned-slicing-with-3d-textures.md) by computing slice geometry dynamically based on the viewing direction rather than using pre-aligned planes, and from [Object Aligned Planes Volume Rendering](11411-object-aligned-planes-volume-rendering.md) by relying on 3D texture lookups instead of pre-extracted 2D textures.

### [Slice Geometry Generation](6883-slice-geometry-generation.md)
Slice Geometry Generation is the procedural or pre-computed creation of proxy geometric primitives (typically polygons or triangle strips) that represent the intersection of slicing planes with a volumetric dataset. It solves the problem of mapping 3D scalar data to 2D screen pixels using hardware rasterization by converting volume intersections into explicit geometry [G]. It differs from pure texture slicing (which relies on hardware texture sampling of 3D textures or 2D stacks) by generating explicit polygonal meshes for each slice, allowing for more complex geometric operations like dynamic lighting and per-pixel normal estimation, while differing from ray casting by leveraging spatial coherence and hardware alpha blending [G].

### [Volumetric textures with textured slices](9800-volumetric-textures-with-textured-slices.md)
Volumetric textures with textured slices render a 3D layer (such as continuous vegetation or fur) on a surface by mapping a 3D data set as a texture pattern onto a series of parallel slices. It solves the problem of efficiently rendering dense, heterogeneous volumetric layers that would otherwise require excessive polygon counts or suffer from popping artifacts in point-based methods. It sets itself apart from sibling methods by generating fragments that are "really 3D" at their proper depth, providing perfect parallax and leveraging hardware texture filtering for smooth antialiasing, which polygonal data lacks. Unlike view-aligned slicing, it does not force slices to face the camera, allowing for more natural surface-conforming geometry but requiring careful viewpoint adaptation to prevent gaps.

### [2D Texture Mapping Approaches](9842-2d-texture-mapping-approaches.md)
2D Texture Mapping Approaches is a family of object-order volume rendering techniques that decompose a 3D scalar field into stacks of 2D texture slices, projecting them onto proxy geometry and compositing them using hardware-accelerated alpha blending. This method solves the problem of rendering volumetric data on graphics hardware that lacks native 3D texture support or trilinear interpolation, relying instead on standard 2D texture mapping and bilinear interpolation. It sets itself apart from sibling methods like [View-Aligned Slicing with 3D Textures](870-view-aligned-slicing-with-3d-textures.md) by avoiding 3D texture units entirely, and from [Cabral Texture Mapping](7257-cabral-texture-mapping.md) by using standard ray-casting compositing rather than filtered back projection.

### [3D Texture Generation](10399-3d-texture-generation.md)
Not given in the sources.

### [Vegetation-Specific Slicing](11648-vegetation-specific-slicing.md)
Vegetation-Specific Slicing is a family of volume rendering techniques tailored for the efficient visualization of plant structures, such as forests and individual trees, by decomposing the geometry into planar slices or fiber surfaces. It solves the problem of high geometric complexity in vegetation rendering by replacing detailed polygonal meshes with image-based representations (slices or textured surfaces) that exploit hardware compositing and texture mapping. It is distinguished from general texture slicing by its specific handling of vegetation topology (fibers in tetrahedral meshes) and large-scale environmental constraints (ground-aligned texcells with non-linear pre-filtering and grazing angle corrections).

#### Image-Aligned Kernel-Slicing
Proposed by Mueller and Crawfis to avoid popping artifacts when slice direction changes, using image-aligned kernel-slicing and traversal order [S5, unofficial]. However, the GPU implementation has high computational cost and needs future investigation [S5, unofficial].

#### Spherical Coordinate Volume Rendering
A method for planet-scaled atmospheric data that avoids accuracy loss from Cartesian-to-geodetic coordinate conversion by operating directly in spherical coordinates [S6, unofficial].

#### Alpha Texturing (Billboarding)
Using RGBA texture maps to render complex geometry (like trees) as simple primitives (rectangles), where alpha=1 represents solid parts (leaves/branches) and alpha=0 represents transparent gaps [S7, book].

#### SGI Octane 2 Hardware Implementation
Renders the slice to a scratch/auxiliary buffer using `glDrawPixels()` or `glCopyPixels()`, pixel textures the slice from the scratch buffer, and blends it with previously rendered slices in the frame buffer [S8, unofficial]. Updating the transfer function uses raw data values for volume texture and pre-computed RGBA shade volume. When rotating volume, it updates the shade volume once rotation completes and generates new quantized normal pixel textures. Updating the shade volume in hardware requires that quantized normal slices are always smaller than the scratch buffer's dimensions [S8, unofficial].

#### Spherical Slicing
Provides a more accurate visualization in perspective projection compared to planar slicing, especially when the observer is very close to the model [S9]. Polygonization may become the performance bottleneck, but using a parallel algorithm for polygonization on multiple processors helps maintain performance levels [S9].

## Background
Volume slicing was first described by Wilson et al. in 1994 [WGW94], using hardware-assisted 3D texture mapping for direct volume rendering [S1, unofficial]. The method maps voxels through classification functions into RGBA values, stores them in a 3D texture map, and slices the texture with parallel planes [S1, unofficial]. Early graphics boards only supported 2D textures, requiring data replication in three axis directions to allow quick construction of geometry for the axis most aligned with the viewpoint [S4, book]. The advent of 3D texturing capabilities improved the method by allowing proxy geometry consisting of parallel polygons orthogonal to the viewing direction [S4, book]. The NVIDIA GeForce3 family, released in 2001, fulfilled the prediction that 3D texture mapping in hardware would become common, enabling interactive frame rates for volumes up to $128 \times 128 \times 128$, while GeForce4 cards support $256 \times 256 \times 256$ volumes [S1, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] khizarsiddiqui/Volume-Rendering, https://github.com/khizarsiddiqui/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S3] Implementing volume rendering using 3D texture slicing - OpenGL - Build ..., https://www.oreilly.com/library/view/opengl-build/9781788296724/ch07s02.html, fetched 2026-10-27 (book)
[S4] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S5] Hardwareaccelerated adaptive ewa volume splatting, https://core.ac.uk/works/16783031, fetched 2026-10-28 (unofficial)
[S6] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
[S7] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S8] https://www.researchgate.net/publication/2399866_Interactive_Volume_Rendering_Using_Multi-Dimensional_Transfer_Functions_and_Direct_Manipulation_Widgets, https://www.researchgate.net/publication/2399866_Interactive_Volume_Rendering_Using_Multi-Dimensional_Transfer_Functions_and_Direct_Manipulation_Widgets, fetched 2026-11-01 (unofficial)
[S9] Chapter 4. Texture Mapping Render Action, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch04.html, fetched 2026-10-27
