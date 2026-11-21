# Slice-based and Proxy Geometry Rendering

Also called: Texture-Based Volume Rendering, Proxy Geometry, Slice Rendering

## Overview
Slice-based and Proxy Geometry Rendering solves the performance bottleneck of per-pixel ray marching by rasterizing 2D geometric primitives (slices) through the volume data, leveraging the graphics pipeline's texture sampling and alpha blending hardware [S1, S3, S4, S5, unofficial]. It is distinguished from direct ray casting by replacing the iterative per-pixel integration loop with parallel rasterization of view-aligned or object-aligned slices, trading memory efficiency for rendering speed [S1][S2, unofficial].

## Shared principles
All methods in this family replace the ray-marching loop with a rasterization pipeline that processes a set of proxy polygons (slices) intersecting the volume. The core mechanism consists of three stages:

1.  **Proxy Geometry Generation**: The volume bounding box is intersected with a series of parallel planes. These planes define 2D polygonal primitives (quads or triangle fans) that represent slices through the volume. The vertices of these polygons are assigned 3D texture coordinates that map the volume data space to the polygon surface [S2, unofficial][S6].
2.  **Texture Sampling**: The volumetric scalar data is stored in GPU memory as a 3D texture or a stack of 2D textures. During rasterization, the fragment shader samples the scalar value $v$ at the interpolated texture coordinates [S2, unofficial][S6].
3.  **Hardware Compositing**: The sampled scalar values are mapped to color and opacity via a transfer function (often stored in a 1D texture). The resulting fragments are composited into the frame buffer using hardware alpha blending. For back-to-front rendering, the blend function is typically `GL_ONE` for source and `GL_ONE_MINUS_SRC_ALPHA` for destination, implementing the over operator:
    $$
    C_{out} = C_{src} + (1 - \alpha_{src}) C_{dst}
    $$
    $$
    \alpha_{out} = \alpha_{src} + (1 - \alpha_{src}) \alpha_{dst}
    $$
    [S1][S2, unofficial]

## Subtopics

### [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md)
This method generates a 2D projection by treating the volume as a series of 2D slices, applying a transfer function, and compositing them using the over operator. It is distinguished by its explicit reliance on a CPU-side or software-based iterative slice processing loop, often involving specific data transformations like logarithmic scaling, rather than optimized GPU rasterization.

### [Hybrid Ray Tracer](2308-hybrid-ray-tracer.md)
The Hybrid Ray Tracer renders mixtures of volumetric sample arrays and polygonal databases by simultaneously tracing rays through both data structures. It resamples volumetric data at equally spaced intervals and computes polygon intersections independently, then composites the results in depth-sorted order. It sets itself apart by explicitly handling embedded polygons as occluders within the volumetric gel, integrating continuous volumetric attenuation with discrete geometric visibility.

### [Mixed Geometry Rendering](3442-mixed-geometry-rendering.md)
Mixed Geometry Rendering combines direct volume ray casting with explicit geometric primitives (such as wireframes) to provide structural context. It solves the problem of depth perception by rendering the volume's internal density while simultaneously displaying an external or internal geometric skeleton. Unlike pure volume rendering, it uses a standard Z-buffer to render geometry that occludes the volume behind it, while the volume correctly transmits light from geometry rendered in front.

### [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md)
Texture-Based Volume Rendering (TBVR) is a GPU-accelerated technique that rasterizes 2D polygon primitives (proxy geometry) through the volume, which are textured with the volumetric data and composited in sorted order using fragment shaders. It is distinguished from direct ray casting by replacing per-pixel ray integration with parallel rasterization of view-aligned or object-aligned slices, trading memory efficiency for rendering speed.

### [View-Aligned Slicing](6530-view-aligned-slicing.md)
View-aligned slicing is a proxy geometry technique that divides a 3D volume into planar slices aligned with the view plane to minimize sampling artifacts and maintain consistent slice thickness relative to the camera. Unlike standard ray casting which samples along the ray direction, this method renders pre-aligned geometric primitives, offering a trade-off of slower performance but higher memory efficiency compared to texture-based approaches.

### [Image-based rendering integration](6969-image-based-rendering-integration.md)
This method integrates volume rendering into a polygonal virtual environment by treating rendered volume images as image-based primitives (impostors) within the scene. It decouples the computationally expensive volume rendering process from the real-time polygonal rendering loop, distributing the workload to a separate server. Unlike direct ray casting which computes pixels per frame on the client, this approach renders static or slowly changing volume views as textures mapped onto quads that track the viewer's orientation.

### [Vector Icon Volume Rendering](7693-vector-icon-volume-rendering.md)
Vector Icon Volume Rendering is a 3D raster resampling technique that integrates flow direction and magnitude into volume rendering by embedding small billboard images containing vector icons within splat footprints. It solves the problem of visualizing vector fields within scalar volumes by creating an illusion of coherent flow through stochastic placement and compositing of these icons. It sets itself apart from sibling methods by using discrete, billboard-oriented icons rather than continuous anisotropic textures, and from View-Aligned Slicing by explicitly embedding vector geometry into the splat footprint rather than just slicing the volume.

### 2D Slice Replication Approach
- **Definition**: When data is stored as a set of 2D texture slices, proxy polygons are simply rectangles aligned with the slices [S1].
- **Steps**: Reconstruct slices on the fly to avoid data replication, with some performance overhead [S1]. Add intermediate slices and perform trilinear interpolation with a fragment shader to solve sampling rate dependency [S1].
- **Pitfalls**: Requires three times more memory because data slices need to be replicated along each principal direction [S1]. Sampling rate depends on the resolution of the volume [S1]. Sampling distance changes with the viewpoint, resulting in intensity variations as the camera moves and image-popping artifacts when switching from one set of slices to another [S1].

### Two-Pass Volume Rendering with Shadows
- **Definition**: A two-pass algorithm to render volumetric shadows on the GPU by modifying proxy geometry slicing to render from both the eye and the light's point of view [S2, unofficial].
- **Steps**: Set the slice axis half-way between the view direction and the light direction [S2, unofficial]. First Pass: Render from the eye. In the fragment shader, look up light color from a light buffer bound as a texture and multiply material color by light color [S2, unofficial]. Second Pass: Render from the light. In the fragment shader, only blend alpha values to compute light transmissivity [S2, unofficial].
- **Parameters**: Render order for light is front-to-back [S2, unofficial]. Render order for eye is front-to-back or back-to-front [S2, unofficial].

### Object-Aligned Slicing
- **Steps**: Create three stacks of 2D textures corresponding to the x, y, and z principal directions [S2, unofficial]. Swap the texture stack based on the viewpoint's closest principal direction [S2, unofficial].

## Background
Originating from the need to accelerate volume rendering on early graphics hardware, this family of techniques leveraged the emerging 3D texture capabilities of GPUs. The approach was popularized in the early 2000s as a way to bypass the limitations of software ray casting and the lack of hardware ray-tracing support. Key developments included the use of 3D textures for interpolation [S1] and the optimization of slice ordering to minimize overdraw and artifacts [S2, unofficial]. It is widely used in medical imaging and scientific visualization where interactive frame rates are required for large datasets.

## Sources
[S1] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S2] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
