# Slice-based and Proxy Geometry Rendering

Also called: Texture-Based Volume Rendering, View-Aligned Slicing, Object-Aligned Slicing

## Overview
Slice-based and Proxy Geometry Rendering is an image-order technique that approximates volume ray casting by rasterizing a set of 2D proxy polygons (slices) through the volume, which are textured with the volumetric scalar field and composited using hardware alpha blending [S1][S2, unofficial][S7]. It solves the problem of interactive volume visualization on consumer hardware by replacing the CPU-intensive ray-marching loop with the graphics pipeline's texture sampling and fragment compositing stages [S3, unofficial][S5, unofficial]. It is distinguished from [Image-order rendering](16-image-order-rendering.md) methods (like ray marching in fragment shaders) by using object-order rasterization of explicit geometry, and from [Neural volume rendering](1576-neural-volume-rendering.md) by its reliance on explicit geometric primitives rather than implicit neural fields.

## Shared principles
All methods in this family share the following mechanism:
1.  **Proxy Geometry Generation**: The volume bounding box is intersected with a series of parallel slicing planes. The resulting polygons (typically quads or triangle fans) serve as proxies for the volume data. Vertices are assigned 3D texture coordinates $(u, v, w)$ that map the volume space $[0,1]^3$ to the polygon surface [S1][S2, unofficial].
2.  **Texture Mapping**: The volumetric scalar data is stored in GPU memory as a 3D texture or a stack of 2D textures [S1][S2, unofficial][S7]. The proxy geometry is textured with this volume data.
3.  **Transfer Function Lookup**: A fragment shader samples the 3D texture to retrieve the scalar value $v$, then performs a dependent lookup into a 1D transfer function texture to obtain color $C$ and opacity $\alpha$ [S1][S2, unofficial][S7].
4.  **Hardware Compositing**: The fragments are composited into the frame buffer using hardware alpha blending. For the standard Over operator (front-to-back or back-to-front depending on sort order), the blend function is typically `GL_ONE` for source and `GL_ONE_MINUS_SRC_ALPHA` for destination [S1][S2, unofficial].
5.  **Sorting**: The proxy polygons must be sorted in depth order relative to the camera before rendering to ensure correct alpha blending [S1][S2, unofficial].

## Subtopics

### [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md)
This method generates a 2D projection of a 3D scalar volume by treating the volume as a series of 2D slices, applying a transfer function to each slice, and compositing them into a final 2D image using the over operator. It is distinguished from its siblings by its explicit reliance on a CPU-side or software-based iterative slice processing loop, often involving specific data transformations (like logarithmic scaling) and direct array manipulation, rather than optimized GPU rasterization pipelines or hybrid ray-tracing techniques. Unlike [View-Aligned Slicing](6530-view-aligned-slicing.md), which optimizes slice orientation for cache coherence, this method may process slices in a fixed or arbitrary order defined by the camera grid structure.

### [Hybrid Ray Tracing](2308-hybrid-ray-tracing.md)
The Hybrid Ray Tracer is a method for rendering mixtures of volumetric sample arrays and polygonal databases by simultaneously tracing rays through both data structures. It resamples volumetric data and polygonal intersections at equally spaced intervals along viewing rays, then composites the resulting colors and opacities in depth-sorted order. It sets itself apart from pure volume rendering by explicitly handling geometric primitives embedded within the volume, and from pure ray tracing by leveraging the regular structure of volumetric slabs for efficient compositing.

### [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md)
Surface rendering is a volume rendering mode that stops at the first voxel inside a specified threshold and shades it with a normal taken from the value gradient. It generates no mesh or triangles, avoiding invented shapes between voxels. Unlike volume rendering which integrates over slice spacing, surface rendering exposes the slice spacing as terracing artifacts.

### [Mixed Geometry Rendering](3442-mixed-geometry-rendering.md)
Mixed Geometry Rendering combines direct volume ray casting with explicit geometric primitives (such as wireframes or surfaces) to provide structural context to volumetric data. It solves the problem of depth perception and structural clarity in scalar fields by rendering the volume's internal density while simultaneously displaying an external or internal geometric skeleton. Unlike pure volume rendering, which relies solely on shading and opacity for shape cues, this method uses a standard Z-buffer to render geometry that occludes the volume behind it, while the volume correctly transmits light from geometry rendered in front.

### [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md)
Texture-Based Volume Rendering (TBVR) is a GPU-accelerated technique that renders volumetric data by rasterizing a set of 2D proxy polygons (slices) through the volume, which are textured with the volumetric scalar field and composited using hardware alpha blending. It solves the problem of interactive volume visualization on consumer hardware by replacing the CPU-intensive ray-marching loop with the graphics pipeline's texture sampling and fragment compositing stages. It is distinguished from [Image-order rendering](52-name.md) methods (like ray marching in fragment shaders) by using object-order rasterization of explicit geometry, and from [Slice-based and Proxy Geometry Rendering](6225-slice-based-and-proxy-geometry-rendering.md) siblings like [View-Aligned Slicing](6530-view-aligned-slicing.md) by its specific reliance on 3D textures or replicated 2D texture stacks to map data to proxy geometry.

### [View-Aligned Slicing](6530-view-aligned-slicing.md)
View-aligned slicing is a proxy geometry method that divides a 3D volume into parallel slices and rotates the texture coordinates to align these slices with the view plane, ensuring consistent sampling distance regardless of camera orientation. Unlike static slice methods that suffer from intensity variations and changing sampling distances as the camera moves, this approach maintains uniform resolution by keeping the proxy geometry stationary while transforming the texture lookup.

### [Image-based rendering integration](6969-image-based-rendering-integration.md)
This method integrates volume rendering into a polygonal virtual environment by treating rendered volume images as image-based primitives (impostors) within the scene. It solves the problem of maintaining interactive frame rates for complex volume data in distributed or latency-sensitive VR applications by decoupling the heavy volume rendering computation from the client-side polygonal rendering loop. Unlike [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md) which renders the volume geometry directly on the client, this approach sends pre-rendered 2D images to the client, allowing the virtual environment to remain responsive even when volume updates are delayed.

### [Vector Icon Volume Rendering](7693-vector-icon-volume-rendering.md)
Vector Icon Volume Rendering is a 3D raster resampling technique that integrates vector field visualization into scalar volume rendering by embedding tiny cylinder icons within splat footprints. It solves the problem of visualizing flow direction and magnitude within volumetric data (such as wind fields or climate simulations) by creating an illusion of coherent motion through stochastic placement and animation. Unlike standard direct volume rendering which only displays scalar density, this method overlays oriented billboard textures that indicate vector orientation, distinguishing it from its siblings by combining scalar density clouds with discrete, animated vector indicators.

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
The method originated with the advent of 3D texture hardware support in GPUs. Early implementations relied on object-aligned slicing, where data was replicated along three principal axes to allow fast rasterization from any viewpoint [S1][S2, unofficial]. This approach suffered from high memory usage (3x) and artifacts during texture stack swaps [S1][S2, unofficial]. View-aligned slicing was introduced to maintain consistent sampling distances by rotating texture coordinates rather than geometry, though it required more complex matrix calculations [S2, unofficial]. The technique became a standard for interactive volume rendering on consumer hardware in the early 2000s, as documented in *GPU Gems* [S1] and various academic presentations [S2, unofficial].

## Sources
[S1] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S2] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
