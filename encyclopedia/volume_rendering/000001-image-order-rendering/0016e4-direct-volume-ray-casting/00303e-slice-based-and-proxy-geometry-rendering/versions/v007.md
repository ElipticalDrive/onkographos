# Slice-based and Proxy Geometry Rendering

Also called: Texture-Based Volume Rendering, Hardware-Accelerated Volume Rendering, Slice Rendering

## Overview
Slice-based and Proxy Geometry Rendering is a direct volume ray casting technique that accelerates rendering by mapping volumetric data to GPU textures and sampling it via rasterized proxy geometry (polygonal slices) [S1][S2, unofficial][S7]. It solves the problem of efficient interactive visualization by leveraging the graphics pipeline's hardware texture sampling and alpha blending capabilities, avoiding the need for explicit ray-marching loops in shaders [S3, unofficial][S5, unofficial]. It is distinguished from pure ray-marching methods by its reliance on geometric primitives and hardware compositing, and from object-aligned slice methods by its ability to use view-aligned slices for consistent sampling [S1][S2, unofficial].

## Shared principles
All methods in this family share the fundamental mechanism of replacing the explicit ray-marching loop with the standard graphics rasterization pipeline. The volume is represented as a 3D texture (or a stack of 2D textures) on the GPU [S1][S2, unofficial]. The rendering process is decomposed into three stages: Initialize, Update, and Draw [S2, unofficial][S7].

1.  **Initialization**: Volumetric data is loaded into CPU memory, processed (e.g., gradient computation, downsampling), and uploaded to GPU memory as a 3D texture or a stack of 2D textures [S2, unofficial][S7]. A 1D texture is created to store the transfer function, mapping scalar values to RGBA [S1][S2, unofficial].
2.  **Update**: When the view or parameters change, proxy geometry is generated. This involves intersecting slicing planes with the volume bounding box, sorting vertices, and constructing polygon primitives (e.g., triangle fans) [S2, unofficial][S4, unofficial]. The polygons are sorted in depth order relative to the camera [S1][S2, unofficial]. If the sampling rate changes, opacity correction of the transfer function textures is performed [S7].
3.  **Draw**: The rendering state is configured to disable lighting and culling, and enable alpha blending [S2, unofficial][S7]. For the Over operator, the blend function is set to Source = `GL_ONE`, Destination = `GL_ONE_MINUS_SRC_ALPHA` [S1][S2, unofficial]. The proxy polygons are drawn in sorted order. For each fragment, the fragment shader interpolates 3D texture coordinates, samples the volume data, performs a dependent lookup into the transfer function texture, and outputs the color and opacity for hardware compositing [S2, unofficial][S7].

## Subtopics

### [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md)
This method generates a 2D projection of a 3D scalar volume by treating the volume as a series of 2D slices, applying a transfer function to each slice, and compositing them into a final 2D image using the over operator. It solves the problem of visualizing medical or scientific 3D data (such as CT scans or dose distributions) without extracting surface geometry. It is distinguished from its siblings by its explicit reliance on a CPU-side or software-based iterative slice processing loop, often involving specific data transformations (like logarithmic scaling) and direct array manipulation, rather than optimized GPU rasterization pipelines or hybrid ray-tracing techniques. Unlike [View-Aligned Slicing](6530-view-aligned-slicing.md), which optimizes slice orientation for cache coherence, this method may process slices in a fixed or arbitrary order defined by the camera grid structure.

### [Hybrid Ray Tracing](2308-hybrid-ray-tracing.md)
Hybrid Ray Tracing is a method for mixing geometric and volumetric data by casting rays through the ensemble of both, drawing samples of each, and compositing them in depth-sorted order. It solves the problem of rendering scenes containing both polygonal objects (e.g., surgical instruments, skin surfaces) and volumetric data (e.g., CT/MR scans) simultaneously. It sets itself apart from pure volume rendering [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md) by explicitly handling polygon intersections and from pure ray tracing [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md) by integrating volumetric attenuation and emission along the ray path.

### [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md)
Surface rendering is a volume rendering mode that stops at the first voxel inside a specified threshold and shades it with a normal taken from the value gradient. It generates no mesh or triangles, avoiding invented shapes between voxels. Unlike volume rendering which integrates over slice spacing, surface rendering exposes the slice spacing as terracing artifacts.

### [Mixed Geometry Rendering](3442-mixed-geometry-rendering.md)
Mixed Geometry Rendering combines direct volume ray casting with explicit geometric primitives (such as wireframes or surfaces) to provide structural context to volumetric data. It solves the problem of depth perception and structural clarity in scalar fields by rendering the volume's internal density while simultaneously displaying an external or internal geometric skeleton. Unlike pure volume rendering, which relies solely on shading and opacity for shape cues, this method uses a standard Z-buffer to render geometry that occludes the volume behind it, while the volume correctly transmits light from geometry rendered in front.

### [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md)
Texture-Based Volume Rendering (TBVR) is a direct volume ray casting technique that accelerates rendering by mapping volumetric data to GPU textures and sampling it via rasterized proxy geometry (polygonal slices). It solves the problem of efficient interactive visualization by leveraging the graphics pipeline's hardware texture sampling and alpha blending capabilities, avoiding the need for explicit ray-marching loops in shaders. It is distinguished from pure ray-marching methods by its reliance on geometric primitives and hardware compositing, and from object-aligned slice methods by its ability to use view-aligned slices for consistent sampling.

### [View-Aligned Slicing](6530-view-aligned-slicing.md)
View-aligned slicing is a direct volume ray casting technique that divides a 3D volume into a series of parallel slices, which are then rotated to align with the view plane (typically the X-Y plane). This alignment minimizes sampling artifacts and maintains a consistent sampling distance relative to the camera, addressing the intensity variations and variable sampling distances inherent in fixed-grid slicing as the viewpoint changes. It sets itself apart from standard fixed-plane slicing by dynamically rotating the texture coordinate transformation rather than the geometry, ensuring uniform sample spacing regardless of view angle.

### [Image-based rendering integration](6969-image-based-rendering-integration.md)
This method integrates volume rendering into a polygonal virtual environment by treating rendered volume images as image-based primitives (impostors) within the scene. It decouples the computationally expensive volume rendering process from the interactive client, distributing it to a separate server and transmitting the resulting images over a network. Unlike direct hardware-accelerated volume rendering which requires the volume data to reside on the client GPU, this approach allows the client to remain lightweight while leveraging server-side processing power.

### [Vector Icon Volume Rendering](7693-vector-icon-volume-rendering.md)
Vector Icon Volume Rendering is a 3D raster resampling technique that integrates vector field visualization into volume rendering by embedding small billboard images containing vector icons within the splat footprint of scalar data. It solves the problem of visualizing flow direction and magnitude within a volumetric scalar field (such as density clouds) by creating an illusion of coherent flow through stochastic placement and orientation of these icons. It sets itself apart from standard direct volume ray casting by explicitly rendering discrete geometric primitives (cylinders or icons) within the splat footprint to represent vector attributes, rather than relying solely on scalar opacity and color transfer functions.

#### 2D Slice Replication Approach
- **Definition**: When data is stored as a set of 2D texture slices, proxy polygons are simply rectangles aligned with the slices [S1].
- **Steps**: Reconstruct slices on the fly to avoid data replication, with some performance overhead [S1]. Add intermediate slices and perform trilinear interpolation with a fragment shader to solve sampling rate dependency [S1].
- **Pitfalls**: Requires three times more memory because data slices need to be replicated along each principal direction [S1]. Sampling rate depends on the resolution of the volume [S1]. Sampling distance changes with the viewpoint, resulting in intensity variations as the camera moves and image-popping artifacts when switching from one set of slices to another [S1].

#### Two-Pass Volume Rendering with Shadows
- **Definition**: A two-pass algorithm to render volumetric shadows on the GPU by modifying proxy geometry slicing to render from both the eye and the light's point of view [S2, unofficial].
- **Steps**: Set the slice axis half-way between the view direction and the light direction [S2, unofficial]. First Pass: Render from the eye. In the fragment shader, look up light color from a light buffer bound as a texture and multiply material color by light color [S2, unofficial]. Second Pass: Render from the light. In the fragment shader, only blend alpha values to compute light transmissivity [S2, unofficial].
- **Parameters**: Render order for light: front-to-back [S2, unofficial]. Render order for eye: front-to-back or back-to-front [S2, unofficial].

#### Object-Aligned Slicing
- **Steps**: Create three stacks of 2D textures corresponding to the x, y, and z principal directions [S2, unofficial]. Swap the texture stack based on the viewpoint's closest principal direction [S2, unofficial].

## Background
Originating from the need for interactive volume rendering on early graphics hardware, this approach leverages the GPU's texture mapping and blending units. The foundational concepts are detailed in "Volume Rendering Techniques" from *GPU Gems* [S1] and various presentations on hardware-accelerated volume rendering [S2, unofficial]. It represents a shift from software-based ray casting to hardware-accelerated rasterization of proxy geometry.

## Sources
[S1] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S2] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
