# Slice-based and Proxy Geometry Rendering

Also called: Texture-Based Volume Rendering, Hardware-Accelerated Volume Rendering, Rasterization-Based Volume Rendering

## Overview
Slice-based and Proxy Geometry Rendering is a direct volume rendering technique that replaces the explicit per-pixel ray-marching loop with the standard graphics rasterization pipeline by decomposing the volume into a set of 2D polygonal slices (proxy geometry) [S1][S2, unofficial]. It solves the problem of interactive volume visualization by leveraging the GPU's hardware texture sampling and alpha blending units, which are significantly faster than software ray casting for large datasets [S2, unofficial][S5, unofficial]. It is distinguished from its siblings by its reliance on geometric primitives and fragment shader compositing rather than per-pixel ray traversal, and from surface rendering by its ability to render semi-transparent internal structures [S1][S2, unofficial].

## Shared principles
All methods in this family share the fundamental mechanism of mapping volumetric data to 2D geometry for rasterization. The core computational engine consists of the following stages:

1.  **Proxy Geometry Generation**: The volume is intersected by a series of parallel slicing planes. The intersection of these planes with the volume's bounding box defines the vertices of 2D polygonal primitives (typically triangles or quads) [S1][S2, unofficial]. These primitives are assigned 3D texture coordinates corresponding to their position in the volume space [S1, S7].
2.  **Depth Sorting**: The proxy polygons must be sorted in depth order relative to the camera. For the standard front-to-back compositing (Over operator), polygons are sorted back-to-front [S1][S2, unofficial][S7].
3.  **Rasterization and Fragment Shading**: The sorted polygons are rasterized. For each fragment, the interpolated 3D texture coordinate is used to sample the volumetric data (stored as a 3D texture or 2D texture stack) [S1][S2, unofficial].
4.  **Transfer Function Lookup**: The sampled scalar value is mapped to color $C$ and opacity $\alpha$ using a transfer function, typically implemented as a 1D texture lookup [S1][S2, unofficial].
5.  **Hardware Compositing**: The fragment's color and opacity are composited into the frame buffer using hardware alpha blending. For the Over operator, the blend function is Source = `GL_ONE`, Destination = `GL_ONE_MINUS_SRC_ALPHA` [S1][S2, unofficial].

## Subtopics

### [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md)
This method generates a 2D projection of a 3D scalar volume by treating the volume as a series of 2D slices, applying a transfer function to each slice, and compositing them into a final 2D image using the over operator. It solves the problem of visualizing medical or scientific 3D data (such as CT scans or dose distributions) without extracting surface geometry. It is distinguished from its siblings by its explicit reliance on a CPU-side or software-based iterative slice processing loop, often involving specific data transformations (like logarithmic scaling) and direct array manipulation, rather than optimized GPU rasterization pipelines or hybrid ray-tracing techniques. Unlike [View-Aligned Slicing](6530-view-aligned-slicing.md), which optimizes slice orientation for cache coherence, this method may process slices in a fixed or arbitrary order defined by the camera grid structure.

### [Hybrid Ray Tracing](2308-hybrid-ray-tracing.md)
Hybrid Ray Tracing is a method for mixing geometric and volumetric data by casting rays through the ensemble of both, drawing samples of each, and compositing them in depth-sorted order. It solves the problem of rendering scenes containing both scalar fields (volumes) and polygonal boundaries simultaneously. Unlike pure volume rendering methods that ignore geometry or pure rasterization methods that ignore volume attenuation, Hybrid Ray Tracing explicitly handles the occlusion and attenuation interactions between the two data types. It sets itself apart from [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md) by treating polygons as distinct entities that interrupt volumetric slabs, rather than converting geometry into voxels.

### [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md)
Surface rendering is a volume rendering mode that stops at the first voxel inside a specified threshold and shades it with a normal taken from the value gradient. It generates no mesh or triangles, avoiding invented shapes between voxels. Unlike volume rendering which integrates over slice spacing, surface rendering exposes the slice spacing as terracing artifacts.

### [Mixed Geometry Rendering](3442-mixed-geometry-rendering.md)
Mixed geometry rendering integrates explicit geometric primitives (isosurfaces, bounding boxes, wireframes) with volumetric data in a single image to provide spatial context and structural clarity. It solves the problem of isolating volumetric features from their surrounding environment by allowing users to see both the internal scalar field distribution and the discrete boundaries or grid structures that define the data domain. Unlike pure volume rendering, which treats the dataset as a continuous medium, this method explicitly renders surface geometry, often using the hardware depth buffer to occlude volume data behind opaque surfaces.

### [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md)
Texture-Based Volume Rendering (TBVR) is a GPU-accelerated technique that renders volumetric data by rasterizing a set of 2D polygon primitives (proxy geometry) through the volume and compositing the results using hardware alpha blending. It solves the problem of interactive volume visualization by replacing the explicit CPU-based ray-marching loop with the standard graphics rasterization pipeline, leveraging the GPU's texture sampling and blending units. It is distinguished from direct ray casting by its reliance on geometric primitives and fragment shader compositing rather than per-pixel ray traversal, and from surface rendering by its ability to render semi-transparent internal structures.

### [View-Aligned Slicing](6530-view-aligned-slicing.md)
View-Aligned Slicing is a direct volume rendering technique that replaces the explicit ray-marching loop with the standard graphics rasterization pipeline by generating a set of 2D triangular proxy geometry slices aligned with the view direction. It solves the problem of maintaining consistent sampling density and minimizing sampling artifacts (such as aliasing) that occur when slicing planes are fixed in object space while the camera moves. Unlike standard ray casting which samples the volume texture directly per pixel, this method decomposes the volume into explicit geometry that is rasterized, requiring dynamic regeneration of geometry when the view changes. It builds on [Slice-based and Proxy Geometry Rendering](6225-slice-based-and-proxy-geometry-rendering.md).

### [Image-based rendering integration](6969-image-based-rendering-integration.md)
This method integrates volume rendering into a polygonal virtual environment by treating rendered volume images as image-based primitives (impostors) within the scene. It solves the problem of coupling high-latency volume rendering with interactive polygonal environments by decoupling the rendering computation from the display pipeline. Unlike sibling methods that rasterize proxy geometry directly on the client GPU, this approach distributes the volume rendering to a separate server and transmits the resulting images to the client, where they are mapped onto quads.

### [Vector Icon Volume Rendering](7693-vector-icon-volume-rendering.md)
Vector Icon Volume Rendering is a 3D raster resampling technique that integrates vector field visualization into volume rendering by embedding small billboard images containing vector icons within the splat footprint of scalar data. It solves the problem of visualizing flow direction and magnitude within a volumetric scalar field (such as density clouds) by creating an illusion of coherent flow through stochastic placement and orientation of these icons. It sets itself apart from standard scalar volume rendering by modifying the volume integral to include the rendering of tiny cylinders or icons within the splat neighborhood, and from pure vector visualization by merging the flow representation with the underlying scalar field density.

### 2D Slice Replication Approach
- **Definition**: When data is stored as a set of 2D texture slices, proxy polygons are simply rectangles aligned with the slices [S1].
- **Steps**: Reconstruct slices on the fly to avoid data replication, with some performance overhead [S1]. Add intermediate slices and perform trilinear interpolation with a fragment shader to solve sampling rate dependency [S1].
- **Pitfalls**: Requires three times more memory because data slices need to be replicated along each principal direction [S1]. Sampling rate depends on the resolution of the volume [S1]. Sampling distance changes with the viewpoint, resulting in intensity variations as the camera moves and image-popping artifacts when switching from one set of slices to another [S1].

### Two-Pass Volume Rendering with Shadows
- **Definition**: A two-pass algorithm to render volumetric shadows on the GPU by modifying proxy geometry slicing to render from both the eye and the light's point of view [S2, unofficial].
- **Steps**: Set the slice axis half-way between the view direction and the light direction [S2, unofficial]. First Pass: Render from the eye. In the fragment shader, look up light color from a light buffer bound as a texture and multiply material color by light color [S2, unofficial]. Second Pass: Render from the light. In the fragment shader, only blend alpha values to compute light transmissivity [S2, unofficial].
- **Parameters**: Render order for light: front-to-back [S2, unofficial]. Render order for eye: front-to-back or back-to-front [S2, unofficial].

### Object-Aligned Slicing
- **Steps**: Create three stacks of 2D textures corresponding to the x, y, and z principal directions [S2, unofficial]. Swap the texture stack based on the viewpoint's closest principal direction [S2, unofficial].

## Background
The method originated with the rise of programmable graphics hardware in the early 2000s, which allowed for the offloading of volume rendering computations to the GPU [S2, unofficial]. Key developments include the use of 3D textures for volume storage and the implementation of transfer functions as 1D textures [S1][S2, unofficial]. The approach was popularized by techniques such as those described in "GPU Gems" Chapter 39 and various academic papers on hardware-accelerated volume rendering [S1][S2, unofficial]. It is widely used in medical imaging, scientific visualization, and real-time interactive applications where performance is critical [S2, unofficial].

## Sources
[S1] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S2] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
