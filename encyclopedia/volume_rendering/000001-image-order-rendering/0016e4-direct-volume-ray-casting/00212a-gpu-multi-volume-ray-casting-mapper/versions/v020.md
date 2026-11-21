# GPU Multi-Volume Ray Casting Mapper

Also called: vtkGPUMultiVolumeRayCastMapper, GPU Multi Volume Ray Cast Mapper, Multi-volume ray casting

## Overview
The GPU Multi-Volume Ray Casting Mapper is a volume rendering technique that renders multiple volume datasets simultaneously by extending the single-volume `vtkGPUVolumeRayCastMapper` to accept multiple `vtkImageData` datasets and `vtkVolumeProperty` objects, using GPU shaders for ray traversal and compositing [S1, S2, unofficial]. It sets itself apart from its siblings by compositing multiple 3D scalar fields along a single ray, using coordinate transformation matrices to align samples from different volumes to a common ray space [S1, unofficial]. Unlike the standard `vtkGPUVolumeRayCastMapper` which handles one volume, this mapper manages the synchronization of ray positions across $N$ volumes, applying front-to-back alpha blending to accumulate color and opacity from all active volumes at each sample step [S1, S2, unofficial].

## How it works
The method implements the emission-absorption optical model, computing the light intensity reaching the eye by integrating radiative energy along a ray through the scalar volume [S1, S5, S9, unofficial]. The process follows a strict pipeline for each pixel: ray generation, sampling, classification (transfer function application), shading, and compositing [S4, S34, unofficial].

### Inputs and Outputs
**Inputs:**
1.  **Primary Volume:** `vtkImageData` dataset (Input Port 0).
2.  **Additional Volumes:** Up to `MAX_NUMBER_OF_ADDITIONAL_VOLUMES` (3 on Apple, 4 otherwise) `vtkImageData` datasets connected to subsequent ports [S1, S2, unofficial].
3.  **Properties:** `vtkVolumeProperty` objects for each volume, accessed via `GetAdditionalProperty(volNumber)` [S1, unofficial].
4.  **Transforms:** User-defined `vtkTransform` objects for additional volumes via `SetAdditionalInputUserTransform(volNumber, transform)` [S1, unofficial].
5.  **Mask (Optional):** A `vtkImageData` mask input (binary or label map) and `MaskBlendFactor` [S2, unofficial].

**Outputs:**
*   A 2D image buffer (fragment color) where each pixel contains the accumulated color and opacity from the composite ray cast [S1, unofficial].

### Data Structures
**Class Members (`vtkGPUMultiVolumeRayCastMapper`):**
*   `NUMBER_OF_ADDITIONAL_VOLUMES`: Integer, typically 3 or 4 depending on platform [S1, unofficial].
*   `TransformedInput`, `TransformedInput2`: Pointers to `vtkImageData` holding shallow copies of inputs for rendering [S2, unofficial].
*   `LastInput`, `LastInput2`: Pointers to `vtkImageData` used to detect data changes via `MTime` comparison [S2, unofficial].
*   `ClippedCroppingRegionPlanes[6]`: Array of doubles storing cropping planes (X min/max, Y min/max, Z min/max) clipped by volume bounds [S1, S2, unofficial].
*   `Property[MAX_NUMBER_OF_ADDITIONAL_VOLUMES]`: Array of `vtkVolumeProperty*` pointers [S1, unofficial].
*   `AdditionalTransformedInput[MAX_NUMBER_OF_ADDITIONAL_VOLUMES]`: Array of `vtkImageData*` for transformed additional volumes [S1, unofficial].

**Shader Uniforms (GLSL):**
*   `sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES]`: 3D textures for scalar data [S1, unofficial].
*   `sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES]`: 1D textures for opacity transfer functions [S1, unofficial].
*   `mat4 P1toPN[MAX_NUMBER_OF_VOLUMES-1]`: Transformation matrices converting texture coordinates from Volume 1 to Volume N [S1, unofficial].
*   `vec3 lowBounds[MAX_NUMBER_OF_VOLUMES]`, `vec3 highBounds[MAX_NUMBER_OF_VOLUMES]`: Bounding box limits for each volume in texture space [S1, unofficial].
*   `int Number_Of_Volumes`: Count of active volumes [S1, unofficial].
*   `vec3 ClippingplanesNormal[MAX_NUMBER_OF_VOLUMES]`, `vec3 ClippingplanesOrigins[MAX_NUMBER_OF_VOLUMES]`: Clipping plane definitions [S1, unofficial].
*   `int clipped[MAX_NUMBER_OF_VOLUMES]`: Flags indicating if clipping is active [S1, unofficial].

### Steps

1.  **Initialization and Memory Allocation:**
    *   Probe the GPU list to determine dedicated video memory. If dedicated memory is 0, use dedicated system memory. If both are 0, set `MaxMemoryInBytes` to 128 MB. Ignore shared system memory [S2, unofficial].
    *   Set `MaxMemoryFraction` to 0.75 (default) to determine the fraction of `MaxMemoryInBytes` used for the 3D texture [S2, unofficial].

2.  **Input Validation and Transformation Setup:**
    *   Validate that inputs are not null and renderer/volume are valid [S2, unofficial].
    *   Check cropping planes: if `Cropping` is true and any min plane $\ge$ max plane, set `goodSoFar` to 0 (silent failure) [S2, unofficial].
    *   If input data has changed (`MTime` > `TransformedInput` `MTime`), create shallow copies into `TransformedInput` [S2, unofficial].
    *   **Extent Adjustment Workaround:** GPU ray casting requires extents starting from zero. For each input:
        *   Get extents, origin, and spacing.
        *   Adjust origin: `new_origin[cc] = origin[cc] + extents[2*cc] * spacing[cc]` [S2, unofficial].
        *   Adjust extents: `extents[2*cc] -= extents[2*cc]`; `extents[2*cc+1] -= extents[2*cc]` [S2, unofficial].
        *   Set new origin and extent [S2, unofficial].
    *   **Transform Matrix Calculation:** For each additional volume $N$, compute `P1toPN` matrix:
        $$ P1toPN_N = T_{world \to tex_N} \times T_{world \to tex_1}^{-1} \times T_{user} $$
        where $T_{user}$ is the user-defined transform [S1, unofficial].

3.  **Clipping/Cropping Initialization:**
    *   Compute `ClippedCroppingRegionPlanes` by intersecting user-defined cropping planes with volume bounds [S1, S2, unofficial].
    *   Convert clipping planes from world coordinates to texture coordinates just before rendering and pass them to shaders via uniforms [S1, unofficial].

4.  **Ray Casting Loop (Fragment Shader `trace()`):**
    *   **Initialization:**
        *   Calculate initial ray position `pos` for Volume 1.
        *   Calculate initial positions `posX[]` for additional volumes: `posX[iii] = P1toPN[iii] * vec4(pos, 1.0)` [S1, unofficial].
        *   Initialize `destColor` (via `initialColor()`) and `remainOpacity = 1.0 - destColor.a` [S1, unofficial].
        *   Set `t = 0.0`, `tMax` (max ray length) [S1, unofficial].
    *   **Traversal Loop:**
        *   While `inside` is true:
            *   `inside` is true if `t < tMax`, `remainOpacity >= 0.0039` (1/255), and `pos` is within `lowBounds[0]` and `highBounds[0]` [S1, unofficial].
            *   **Sample Volumes:**
                *   For Volume 0: If `pos` is within bounds and not clipped, sample `value[0] = texture3D(dataSetTexture[0], pos)`. Compute `scalar[0] = scalarFromValue(value[0])`. Sample `opacity[0] = texture1D(opacityTexture[0], scalar[0])` [S1, unofficial].
                *   For Volumes $1$ to $N-1$: If `posX[xx-1]` is within bounds and not clipped, sample `value[xx] = texture3D(dataSetTexture[xx], posX[xx-1])`. Compute `scalar[xx]`. Sample `opacity[xx]` [S1, unofficial].
            *   **Shading:**
                *   If shading is enabled, `InitShade()` computes corrected x, y, z increments using `P1toPN` for each volume to estimate gradients/normals [S1, unofficial].
                *   Compute `color[vol] = shade(vol, value[vol]) * opacity[vol].a` [S1, unofficial].
            *   **Compositing:**
                *   If `opacity[vol].a > 0.0`:
                    $$ destColor = destColor + color[vol] \times remainOpacity $$
                    $$ remainOpacity = remainOpacity \times (1.0 - opacity[vol].a) $$
                *   Note: The sources describe compositing all volumes at the current sample step before advancing the ray [S1, unofficial].
            *   **Advance Ray:**
                *   `pos = pos + rayDir`
                *   `posX[iii] = posX[iii] + rayDir` (transformed direction) [S1, unofficial].
                *   `t += 1.0` [S1, unofficial].
    *   **Output:**
        *   `gl_FragColor = destColor`
        *   `gl_FragColor.a = 1.0 - remainOpacity` [S1, unofficial].

5.  **Masking (Optional):**
    *   If `MaskInput` is provided, blend the standard color transfer function with the mask-specific color transfer function using `MaskBlendFactor` [S2, unofficial].

### Parameters
*   `MAX_NUMBER_OF_ADDITIONAL_VOLUMES`: 3 (Apple), 4 (Others) [S1, unofficial].
*   `AutoAdjustSampleDistances`: Boolean (default 1). Enables automatic adjustment of `ImageSampleDistance` [S2, unofficial].
*   `ImageSampleDistance`: Float (default 1.0). Base sample distance in image space. Clamped to [0.1, 100.0] [S2, unofficial].
*   `MinimumImageSampleDistance`: Float (default 1.0). Lower bound for adaptive sampling [S2, unofficial].
*   `MaximumImageSampleDistance`: Float (default 10.0). Upper bound for adaptive sampling [S2, unofficial].
*   `SampleDistance`: Float (default 1.0). Distance between sample points along the ray [S2, unofficial].
*   `MaxMemoryInBytes`: vtkIdType. Max GPU memory for volume data. Defaults to 128 MB if GPU info unavailable [S2, unofficial].
*   `MaxMemoryFraction`: Float (default 0.75). Fraction of `MaxMemoryInBytes` used [S2, unofficial].
*   `ReportProgress`: Boolean (default true). Uses `glFinish()` to block and report progress. Should be false for small datasets to avoid slowdown [S2, unofficial].
*   `FinalColorWindow`: Float (default 1.0). Window for final color scaling [S2, unofficial].
*   `FinalColorLevel`: Float (default 0.5). Level for final color scaling [S2, unofficial].
*   `MaskBlendFactor`: Float (default 1.0). Blending weight for mask CTF [S2, unofficial].
*   `MaskType`: Enum (default `LabelMapMaskType`). Can be `BinaryMaskType` or `LabelMapMaskType` [S2, unofficial].
*   `ClippedCroppingRegionPlanes[6]`: Array of doubles. Defaults: mins to `VTK_DOUBLE_MAX`, maxs to `VTK_DOUBLE_MIN` [S2, unofficial].

### Edge Cases and Pitfalls
*   **Hardware Loop Limits:** GPU hardware limits on loop iterations require a nested while loop structure instead of a single loop over all samples and volumes [S1, unofficial].
*   **Non-Zero Extents:** The mapper cannot handle extents starting from non-zero values natively. The extent adjustment workaround (Step 2) is required [S2, unofficial].
*   **Invalid Cropping:** If cropping planes define a region with zero or negative volume, the render fails silently (`goodSoFar=0`) without an error message [S2, unofficial].
*   **ReportProgress Overhead:** Using `glFinish()` for progress reporting blocks the GPU and significantly slows down rendering of small datasets; it should be disabled for small datasets [S2, unofficial].
*   **FinalColorWindow:** Setting `FinalColorWindow` to a negative value reverses the applied color values [S2, unofficial].
*   **Color Channels:** Currently requires 3 color channels (RGB). 4-component rendering may not work correctly [S2, unofficial].

## Subtopics

### [OpenGL GPU Multi Volume Ray Cast Mapper](2919-opengl-gpu-multi-volume-ray-cast-mapper.md)
The OpenGL GPU Multi Volume Ray Cast Mapper is an OpenGL-specific subclass of the GPU Multi-Volume Ray Casting Mapper that handles hardware extension checks, OpenGL object lifecycle management, and clipping/cropping plane setup for GPU-accelerated volume rendering. It serves as the concrete implementation responsible for drawing the volume ray cast image to the screen by managing fragment shader programs, framebuffer objects, and texture uploads required for rendering multiple volumes simultaneously. It builds on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) by providing the specific OpenGL API bindings and state management necessary for hardware-accelerated ray casting.

### [Label Map Masking for Volume Rendering](4788-label-map-masking-for-volume-rendering.md)
Label Map Masking is a technique that confines volume rendering to specific regions using either a binary mask or a multi-label map, supporting blending between standard and mask-specific color transfer functions. It sets itself apart from sibling mappers by explicitly defining how mask values (binary or labeled) dictate the selection of color transfer functions (CTFs) and their blending weights, rather than simply acting as a visibility gate.

### [GPU Volume Ray Casting (VTK Composite Blend)](6912-gpu-volume-ray-casting-vtk-composite-blend.md)
GPU Volume Ray Casting (VTK Composite Blend) is a hardware-accelerated direct volume rendering technique implemented in VTK via `vtkGPUVolumeRayCastMapper` that performs ray casting on the GPU to visualize 3D scalar fields. It transforms raw volumetric data into a 3D environment by accumulating color and opacity along rays using composite blending. It sets itself apart from its siblings by explicitly utilizing the **Composite** blend mode for standard alpha blending during the ray casting process, distinguishing it from the OpenGL-specific multi-volume mapper [2919-opengl-gpu-multi-volume-ray-cast-mapper.md] and the Slicer-specific mapper [7450-vtkslicergpuraycastvolumemapper.md].

### [vtkSlicerGPURayCastVolumeMapper](7450-vtkslicergpuraycastvolumemapper.md)
Renders a volume using ray casting on a 3D texture. It builds on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) to provide a specific implementation for the 3D Slicer environment, supporting adaptive sampling, masking, and AMR modes via the parent's pipeline hooks. It is distinguished from its siblings by its specific integration with CTK transfer functions and its handling of specific OpenGL uniform layouts for ray casting parameters.

## Operating characteristics
*   **Time Complexity:** $O(P \times N \times S)$, where $P$ is the number of pixels, $N$ is the number of volumes, and $S$ is the number of samples per ray. Adaptive sampling can reduce $S$ dynamically based on allocated render time [S2, unofficial].
*   **Memory Complexity:** $O(V \times D^3)$, where $V$ is the number of volumes and $D$ is the voxel grid dimension. Memory is limited by `MaxMemoryInBytes` [S2, unofficial].
*   **Performance:** Uses GLSL shaders for parallel execution on the GPU, which is faster and more widely supported than OpenCL for this task [S1, unofficial].
*   **Strengths:** Simultaneous rendering of multiple volumes with correct compositing; support for masking and adaptive sampling; hardware-accelerated shading [S1, S2, unofficial].
*   **Limitations:** Limited by GPU memory for large datasets; silent failure on invalid cropping; requires extent adjustment for non-zero origins; color channel restriction (RGB) [S1, S2, unofficial].
*   **When to Use:** When visualizing multiple co-registered or spatially related volume datasets (e.g., multi-modal medical imaging) that need to be composited together in a single pass [S1, unofficial].
*   **When Not to Use:** When datasets are too large to fit in GPU memory; when single-volume rendering is sufficient; when non-RGB color channels are required [S2, unofficial].

## Background
The GPU Multi-Volume Ray Casting Mapper was developed as an extension to the Visualization Toolkit (VTK) to support multi-volume rendering on the GPU. It was modified by Carlos Falcón and Karl Krissian to extend the single-volume `vtkGPUVolumeRayCastMapper` [S1, unofficial]. The implementation relies on GLSL shaders for ray traversal and compositing, chosen for their speed and broad hardware support compared to OpenCL [S1, unofficial]. It is part of the VTK volume rendering pipeline, integrating with `vtkVolumeMapper`, `vtkVolume`, and `vtkVolumeProperty` classes [S1, S2, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
[S3] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-27 (unofficial)
