# Two-pass and Multi-pass Techniques

Also called: Multi-pass Volume Rendering, Decoupled Volume Rendering

## Overview
Two-pass and Multi-pass Techniques decouple the computation of light attenuation (or ray bounds) from the final color compositing to optimize performance on graphics hardware [S1, 1292-two-pass-volume-rendering.md, 5965-two-pass-ray-marching-volume-rendering.md, 7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md]. Unlike single-pass ray marching which processes all samples sequentially in one shader invocation, these methods split the workload into distinct rendering passes, allowing for efficient reuse of precomputed data (such as light transmissivity or ray exit depths) and leveraging hardware features like early Z-testing or specific blend modes to reduce redundant texture fetches and arithmetic operations [1292-two-pass-volume-rendering.md, 5965-two-pass-ray-marching-volume-rendering.md, 7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md].

## Shared principles
The core mechanism involves separating the volume rendering pipeline into at least two distinct rendering passes, each targeting specific intermediate buffers or the final frame buffer.

1.  **Decoupling of Concerns:** The primary goal is to isolate computationally expensive or data-dependent calculations from the final accumulation. Common decouplings include:
    *   **Light vs. Eye:** Separating the computation of light attenuation (shadowing/transmissivity) from the eye's view compositing [1292-two-pass-volume-rendering.md].
    *   **Bounds vs. Marching:** Separating the determination of ray entry and exit points from the actual ray marching and color accumulation [5965-two-pass-ray-marching-volume-rendering.md, 7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md].
    *   **Steps vs. Passes:** Splitting a long ray traversal into multiple shorter passes to adhere to hardware shader instruction limits or to manage early termination logic [7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md].

2.  **Intermediate Buffers:** These techniques rely on off-screen render targets (textures) to store intermediate results between passes. Common buffers include:
    *   **Light/Transmissivity Buffer:** Stores accumulated opacity or light intensity from a light-pass perspective [1255-half-angle-slicing.md, 1292-two-pass-volume-rendering.md].
    *   **Depth/Position Buffer:** Stores the exit point or depth of the volume along the ray to constrain the marching range in the subsequent pass [5965-two-pass-ray-marching-volume-rendering.md, 7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md].
    *   **Direction/Step Buffer:** Stores ray directions and lengths to facilitate multi-step traversal [7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md].
    *   **Result Buffer:** Accumulates color and opacity across multiple main traversal passes [7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md].

3.  **Blend Modes:** Hardware blend modes are critical for correct accumulation.
    *   **Premultiplied Over:** Used in light passes to accumulate opacity/transmissivity front-to-back relative to the light source [1292-two-pass-volume-rendering.md].
    *   **Under Blend:** Used in eye passes to composite slice colors using precomputed light attenuation [1292-two-pass-volume-rendering.md].
    *   **Additive/Alpha-to-Weight:** Used in specific variants for light scattering or accumulation [2373-alpha-to-weight-computation-forward.md, 2515-compositing-and-blending.md].

4.  **Opacity Correction:** When rendering with a discrete number of slices or steps, the total opacity may vary with the count. An opacity correction factor is often applied to maintain image intensity invariance [1292-two-pass-volume-rendering.md, 5965-two-pass-ray-marching-volume-rendering.md].

## Subtopics

### [Half-angle slicing](1255-half-angle-slicing.md)
Half-angle slicing is a two-pass per-slice rendering technique for volumetric shadows that orients proxy slice planes perpendicular to the half-angle vector between the eye and light directions [1255-half-angle-slicing.md]. This orientation allows a single slice to be rendered from both the observer's and the light's points of view, minimizing the number of slices required to resolve lighting details compared to standard view-aligned or light-aligned slicing [1255-half-angle-slicing.md]. Unlike standard shadow mapping which requires pre-computation or large 3D shadow volumes, this method achieves high-resolution shadow effects using only a single additional 2D buffer [1255-half-angle-slicing.md]. It is chosen when high-quality volumetric shadows are needed with limited slice counts. [1255-half-angle-slicing.md]

### [Two-pass volume rendering](1292-two-pass-volume-rendering.md)
Two-pass volume rendering decouples light attenuation computation from color compositing by separating the rendering workload into a light pass and an eye pass [1292-two-pass-volume-rendering.md]. It differs from single-pass ray marching by allowing efficient reuse of precomputed light attenuation and leveraging hardware 'under' blending for front-to-back compositing [1292-two-pass-volume-rendering.md]. It is chosen when lighting effects (shadows) are critical and the hardware supports efficient multi-pass rendering with custom blend modes. [1292-two-pass-volume-rendering.md]

### [Two-Pass Ray Marching Volume Rendering](5965-two-pass-ray-marching-volume-rendering.md)
Two-Pass Ray Marching Volume Rendering is a GPU-accelerated technique that decouples the determination of ray entry and exit points from the color accumulation phase to optimize ray marching efficiency [5965-two-pass-ray-marching-volume-rendering.md]. Unlike standard single-pass ray casting which may march through empty space or recompute depth bounds per pixel, this method first renders the volume's back-face to a depth buffer to establish the ray's exit point, then uses that information in a second pass to march rays from the camera only through the occupied volume [5965-two-pass-ray-marching-volume-rendering.md]. It is chosen to reduce unnecessary sample calculations in empty space and to simplify ray traversal logic. [5965-two-pass-ray-marching-volume-rendering.md]

### [Multi-pass GPU Ray-Casting with Early Ray Termination](7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md)
This method implements direct volume ray casting on DirectX 9-level graphics hardware using the Pixel Shader 2.0 API [7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md]. It solves the performance bottleneck of per-fragment ray traversal by splitting the workload into multiple rendering passes, utilizing early ray termination and empty-space skipping to reduce texture fetches and arithmetic operations [7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md]. Unlike single-pass approaches, it decouples ray direction computation and opacity accumulation into distinct stages, allowing the hardware's early Z-test to cull fragments that have already reached opacity saturation or exited the volume [7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md]. It is chosen for older hardware with limited shader instruction counts or when strict early termination is required to save fill-rate. [7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md]

### Multi-pass Stencil Occlusion for Volumetric Light Scattering
#### definition
An implementation approach for fixed-function hardware or Shader Model 2.0 limits, using additive frame-buffer blending over multiple passes with a stencil occlusion method [S1, book].
#### parameter
Shader Model 3.0 is required for the single-pass version because the number of texture samples needed exceeds the limits of Shader Model 2.0 [S1, book].

## Background
The concept of decoupling volume rendering tasks into multiple passes emerged as a response to the limited instruction counts and texture sampling capabilities of early programmable graphics hardware (e.g., Shader Model 2.0) [7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md, S1]. Techniques like half-angle slicing and two-pass volume rendering were developed to efficiently compute complex lighting effects like volumetric shadows without the overhead of full ray tracing or large shadow volumes [1255-half-angle-slicing.md, 1292-two-pass-volume-rendering.md]. Multi-pass ray marching with early termination was specifically designed to work within the constraints of DirectX 9-level hardware, using stencil buffers and intermediate textures to manage ray traversal state [7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md]. These methods remain relevant for optimizing performance on mobile GPUs or legacy systems, and their principles (decoupling bounds, lighting, and accumulation) are foundational in modern hybrid rendering pipelines.

## Sources
[S1] Volumetric Light Scattering as a Post-Process, https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-13-volumetric-light-scattering-post-process, fetched 2026-10-27 (book)
