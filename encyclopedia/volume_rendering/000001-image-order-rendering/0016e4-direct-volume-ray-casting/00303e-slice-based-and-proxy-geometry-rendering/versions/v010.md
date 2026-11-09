# Slice-based and Proxy Geometry Rendering

Also called: Texture-Based Volume Rendering, View-Aligned Slicing, Object-Aligned Slicing, Hybrid Ray Tracing, Surface Rendering Pipeline, Mixed Geometry Rendering, Image-based Rendering Integration, Vector Icon Volume Rendering

## Overview
Slice-based and Proxy Geometry Rendering accelerates direct volume ray casting by rasterizing 2D proxy polygons (slices) through the volume instead of marching rays per pixel [S1, S2, S8, unofficial]. It solves the performance bottleneck of CPU-based ray marching by leveraging the GPU's dedicated texture sampling units and hardware alpha blending to compute samples per fragment [S1, S2, S8, unofficial]. Unlike pure ray casting which computes samples per pixel, this method generates explicit geometry that is rasterized, requiring dynamic regeneration of the proxy mesh when the camera transform changes [S2, unofficial]. It is distinguished from [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md) by its use of GPU-accelerated rasterization and from [Hybrid Ray Tracing](2308-hybrid-ray-tracing.md) by its reliance on fixed-function or programmable rasterization pipelines rather than ray intersection tests [S2, S6, unofficial].

## Shared principles
The method operates on the fundamental loop structure of [Image-order rendering](16-image-order-rendering.md): for each pixel, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, intersected with the volume domain, and sampled/composited along its path [G]. The traversal order is determined by the pixel's position in the image plane [G].

The core computational engine consists of four sequential stages applied to each ray [S5, S10, S37, unofficial]:

1.  **Ray Generation and Discretization**: The ray segment $[t_{start}, t_{end}]$ intersecting the volume is determined, typically by intersecting the ray with the volume's bounding box [S1][S8, S16, unofficial]. The segment is discretized into $N$ samples at equidistant intervals $\Delta t = (t_{end} - t_{start}) / N$ [S5, S10, S17, unofficial].
2.  **Sampling and Interpolation**: At each sample point $\mathbf{p}_i = r(t_i)$, the scalar value $V(\mathbf{p}_i)$ is retrieved from the volume data. This is typically performed by trilinear interpolation of the eight surrounding voxels [S9, S11, S47, unofficial].
3.  **Classification (Transfer Function)**: The scalar value $V(\mathbf{p}_i)$ is mapped to optical properties: color $C_i$ and opacity $\alpha_i$ [S2, S2789]. The opacity is often corrected for the step size $\Delta t$ to ensure physical consistency, using the Beer-Lambert law approximation:
    $$ \alpha_i = 1 - e^{-\sigma_i \Delta t} $$
    where $\sigma_i$ is the extinction coefficient derived from the scalar value [S24, S30, unofficial].
4.  **Compositing**: The samples are accumulated from front to back (closest to the eye to furthest) using the front-to-back alpha compositing equation [S1][S2, S13, unofficial][S61]:
    $$ C_{out} = C_{in} + (1 - \alpha_{in}) \cdot C_{sample} \cdot \alpha_{sample} $$
    $$ \alpha_{out} = \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha_{sample} $$
    where $C_{in}$ and $\alpha_{in}$ are the accumulated color and opacity from previous samples, and $C_{sample}$ and $\alpha_{sample}$ are the color and opacity of the current sample [S52, S60, unofficial]. The process initializes with $C_{in} = 0$ and $\alpha_{in} = 0$ [S15, S46, unofficial].

## Subtopics

### [Hybrid Ray Tracing](2308-hybrid-ray-tracing.md)
Hybrid Ray Tracing is a method for mixing geometric and volumetric data by casting rays through the ensemble of both, drawing samples of each, and compositing them in depth-sorted order. It solves the problem of rendering scenes containing both scalar fields (volumes) and polygonal models simultaneously, ensuring correct visibility and occlusion between the two data types. Unlike pure volume rendering which ignores geometry, or pure polygon rendering which ignores volume density, this method treats polygons as embedded surfaces within the volumetric gel, handling the "hidden-volume" problem locally to avoid the expense of exact intersection tests.

### [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md)
Surface rendering is a volume rendering mode that stops at the first voxel inside a specified threshold and shades it with a normal taken from the value gradient. It generates no mesh or triangles, avoiding invented shapes between voxels. Unlike volume rendering which integrates over slice spacing, surface rendering exposes the slice spacing as terracing artifacts.

### [Mixed Geometry Rendering](3442-mixed-geometry-rendering.md)
Mixed geometry rendering integrates explicit geometric primitives (isosurfaces, bounding boxes, wireframes) with volumetric data in a single image to provide spatial context and structural clarity. It solves the problem of isolating volumetric features from their surrounding environment by allowing users to see both the internal scalar field distribution and the discrete boundaries or grid structures that define the data domain. Unlike pure volume rendering, which treats the dataset as a continuous medium, this method explicitly renders surface geometry, often using the hardware depth buffer to occlude volume data behind opaque surfaces.

### [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md)
Texture-Based Volume Rendering (TBVR) is an image-order rendering technique that accelerates direct volume ray casting by exploiting modern graphics hardware (GPUs) to rasterize proxy geometry. It solves the performance bottleneck of CPU-based ray marching by loading volumetric data into texture memory and intersecting it with parallel slicing planes (proxy geometry) that are rasterized and composited using hardware alpha blending. Unlike pure ray casting which computes samples per pixel, TBVR computes samples per fragment of the proxy polygons, leveraging the GPU's dedicated texture sampling units for trilinear or bilinear interpolation. It is distinguished from [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md) by its use of GPU-accelerated rasterization and from [Hybrid Ray Tracing](2308-hybrid-ray-tracing.md) by its reliance on fixed-function or programmable rasterization pipelines rather than ray intersection tests.

### [View-Aligned Slicing](6530-view-aligned-slicing.md)
View-Aligned Slicing is a direct volume rendering technique that decomposes a 3D volume into a series of 2D proxy polygons (typically quads or triangle fans) aligned with the view direction. Unlike standard ray casting which samples the volume texture directly per pixel, this method generates explicit geometry that is rasterized, requiring dynamic regeneration of the proxy mesh when the camera transform changes. It sets itself apart from siblings like [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md) by explicitly managing slice geometry to maintain consistent sampling distance relative to the camera, thereby reducing view-dependent aliasing artifacts.

### [Image-based rendering integration](6969-image-based-rendering-integration.md)
This method integrates volume rendering into a polygonal virtual environment by treating rendered volume images as image-based primitives (impostors) within the scene. It solves the problem of coupling high-latency volume rendering with real-time interactive navigation by decoupling the rendering workload from the client's display loop. Unlike sibling methods that render volume geometry directly on the client, this approach distributes the heavy computation to a separate server and streams the resulting images to the client, where they are mapped onto simple geometric proxies.

### [Vector Icon Volume Rendering](7693-vector-icon-volume-rendering.md)
Vector Icon Volume Rendering is a technique for visualizing vector fields (such as wind or fluid flow) within a 3D volume by embedding small billboard textures containing vector icons into the splat footprints of a sheet-based volume renderer. It solves the problem of conveying direction and magnitude in scalar volume renderings by stochastically placing oriented icons that create an illusion of coherent motion. It differs from pure scalar volume rendering by adding directional glyphs and from line-based methods (like Line Bundles) by using discrete, billboarded icons rather than continuous streamlines.

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

### Ray-to-Gravity-Center (R-GC) Volume Slicing
- **Definition**: A volume slicing strategy where parallel cutting planes are oriented perpendicular to the straight line connecting the point of reflection to the gravity center of the volume domain, used to generate Layered Polygonal Area Lights (LPALs) [S3].
- **Number**: Visual result: R-RR fails to accurately render the upper part of the reflected appearance when apart from the volume, whereas R-GC handles this correctly [S3].
- **Comparison**: Compared to Ray-of-Regular-Reflection (R-RR) slicing: R-RR fails to capture the upper part of reflected appearance when the R-RR is apart from the volume because LPALs face away from the reflection point. R-GC ensures LPALs face properly to the point of reflection regardless of the regular reflection direction [S3].

## Background
The technique was popularized in the early 2000s as graphics hardware evolved to support 3D textures and hardware alpha blending. Key early works include Lefohn et al. (2004) for on-the-fly slice reconstruction, Rezk-Salama et al. (2000) for trilinear interpolation in fragment shaders, and Kniss et al. (2002b) for addressing intensity variations and popping artifacts. The method is widely used in medical imaging and scientific visualization where interactive frame rates are required for large datasets.

## Sources
[S1] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S2] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S3] Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights, https://doi.org/10.1111/cgf.13851, fetched 2026-10-26
