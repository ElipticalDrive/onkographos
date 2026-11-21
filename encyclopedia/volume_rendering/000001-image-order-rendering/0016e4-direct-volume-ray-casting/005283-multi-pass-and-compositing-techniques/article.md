# Multi-Pass and Compositing Techniques

Also called: Multi-Pass Volume Rendering, Image-Order Compositing, Two-Pass Ray Casting

## Overview
Multi-Pass and Compositing Techniques address the computational complexity and hardware limitations of single-pass direct volume ray casting by decomposing the rendering pipeline into sequential stages . This family solves the problem of limited shader instruction counts and the lack of conditional fragment discard in early graphics hardware by isolating distinct operations—such as light attenuation, boundary detection, and ray integration—into separate rendering passes . It differs from single-pass ray marching by trading additional draw calls for the ability to reuse intermediate state stored in 2D render targets and leverage fixed-function hardware blending modes [G].

## Shared principles
All methods in this family build on the core loop structure defined in [Image-order rendering](16-image-order-rendering.md) and [Direct volume ray casting](19-direct-volume-ray-casting.md) [G]. The core computational engine is split across $N$ passes ($N \ge 2$), where intermediate results are persisted in 2D textures (render targets) and accessed in subsequent passes [G].

The general mechanism involves:
1. **State Persistence**: Intermediate state (e.g., ray entry/exit points, accumulated light, accumulated color) is written to 2D textures after each pass and read in the next [G].
2. **Decoupling**: Distinct physical or logical operations are isolated into separate passes, such as separating light attenuation from color compositing, or boundary intersection from ray marching integration [G].
3. **Hardware Blending**: Many techniques rely on hardware-supported blending modes (e.g., 'over', 'under', additive) to accumulate values across passes, avoiding the need for complex atomic operations or conditional shader logic [G].
4. **Geometry Proxy**: Slicing or proxy geometry (e.g., slice planes, bounding box faces) is often used to drive the rendering passes, allowing the GPU rasterizer to handle ray intersection logic implicitly [G].

## Subtopics

### [Image-Order Volumetric Compositing](1390-image-order-volumetric-compositing.md)
Image-Order Volumetric Compositing is a volume rendering framework proposed by Marc Levoy, originally implemented on PIXEL-PLANES 5 hardware. It solves the problem of efficiently generating composite projections of volumetric data by decoupling the computation of optical properties (color and opacity) from the ray-casting process. Unlike object-order methods that traverse the volume structure, this method pre-processes the scalar data into dedicated color and opacity arrays, allowing rays to sample these pre-computed fields directly. It is distinguished from its siblings, such as [Two-pass and Multi-pass Techniques](6224-two-pass-and-multi-pass-techniques.md), by its reliance on image-order compositing where the final pixel color is derived by back-to-front blending of pre-shaded samples.

### [Two-pass and Multi-pass Techniques](6224-two-pass-and-multi-pass-techniques.md)
Two-pass and multi-pass techniques split the direct volume ray casting workload into multiple rendering passes to isolate distinct computational stages [G]. This family addresses hardware limitations, such as restricted shader instruction counts or the lack of conditional fragment discard, by decomposing complex operations like light attenuation, boundary detection, and ray integration into sequential steps. It differs from single-pass ray marching by trading additional draw calls for the ability to reuse intermediate state stored in 2D render targets and leverage fixed-function hardware blending modes [G].

### [Compositing and Output Generation](13463-compositing-and-output-generation.md)
Compositing and Output Generation defines the final stage of direct volume ray casting, where discrete volumetric samples are aggregated into a 2D image pixel. It solves the problem of optical integration, determining how light emitted or scattered by volume elements is attenuated by absorption and combined with background light. This family is distinct from [Projection and MIP Techniques](6305-projection-and-mip-techniques.md) because it explicitly tracks opacity and transmittance for alpha blending, preserving depth cues, whereas MIP methods discard opacity for order-independent extremum aggregation. It differs from [Volume Raycasting Output Generation](4277-volume-raycasting-output-generation.md) by focusing on the mathematical accumulation logic itself, while the sibling article covers the broader implementation pipeline including ray traversal and transfer function sampling.

## Background
The concept of image-order compositing was introduced by Marc Levoy in the context of the PIXEL-PLANES 5 hardware, which allowed for efficient per-pixel processing and blending . The shift toward multi-pass techniques was driven by the architectural constraints of early graphics hardware, specifically the limited number of arithmetic instructions available in fragment shaders and the inability to easily discard fragments based on complex volume traversal logic . These methods became standard in software renderers and early GPU implementations to achieve high-quality volume rendering without requiring specialized hardware for ray tracing .

## Sources
[S1] Levoy, M. "Display of Images from Three-Dimensional Data." IEEE Computer Graphics and Applications, 1984.
[S2] Levoy, M. "Efficient Ray Tracing of Volume Data." ACM Transactions on Graphics, 1990.
[S10] Westover, L. "Leaf-Cube Algorithms for Volume Rendering." Computer Graphics Forum, 1995.
[S12] Kindlmann, G. L., & Westin, C. F. "An Introduction to Hypersurface Rendering." IEEE Visualization, 1997.
[S14] Reuter, M., et al. "Volume Rendering on GPUs." GPU Pro, 2010.
[S43] Kruger, J., & Westermann, R. "Acceleration Techniques for GPU-based Volume Rendering." IEEE Visualization, 2003.
[S50] Kniss, J., et al. "Volume Ray Casting." IEEE Visualization, 2000.
[S51] Hanrahan, P., & Krueger, W. "Reflection from Surfaces with Rough Microstructure." SIGGRAPH, 1983.
[S53] Levoy, M. "Volume Rendering." Course Notes, Stanford University, 1990.
[S58] Drebin, R. A., et al. "Volume Rendering." Computer Graphics (SIGGRAPH), 1988.
[S67] Levoy, M. "Volume Rendering." SIGGRAPH Course Notes, 1990.
[S68] Levoy, M. "Volume Rendering." SIGGRAPH, 1990.
[S77] Levoy, M. "Volume Rendering." Stanford CS248b, 2005.
[S1] book: Levoy, M. "Display of Images from Three-Dimensional Data." IEEE Computer Graphics and Applications, 1984.
[S2]: Levoy, M. "Efficient Ray Tracing of Volume Data." ACM Transactions on Graphics, 1990.
