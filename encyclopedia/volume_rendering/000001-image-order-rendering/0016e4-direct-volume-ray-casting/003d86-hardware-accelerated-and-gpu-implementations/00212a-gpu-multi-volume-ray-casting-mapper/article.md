# GPU Multi-Volume Ray Casting Mapper

Also called: vtkGPUMultiVolumeRayCastMapper, GPU Multi Volume Ray Cast Mapper, Multi-volume ray casting

## Overview
The GPU Multi-Volume Ray Casting Mapper is a volume rendering technique that renders multiple volume datasets simultaneously by extending the single-volume `vtkGPUVolumeRayCastMapper` to accept multiple `vtkImageData` datasets and `vtkVolumeProperty` objects, using GPU shaders for ray traversal and compositing [S1, unofficial]. It sets itself apart from its siblings by compositing multiple 3D scalar fields along a single ray, using coordinate transformation matrices to align samples from different volumes to a common ray space [S1, unofficial]. It builds on [Hardware-Accelerated and GPU Implementations](9525-hardware-accelerated-and-gpu-implementations.md) by providing the specific logic for multi-input data management and shader uniform updates required for simultaneous rendering [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two or more `vtkImageData` datasets (scalar fields) and corresponding `vtkVolumeProperty` objects (transfer functions). The primary input is at port 0; additional inputs are at ports 1 to $N$.
*   **Outputs:** A 2D color buffer (fragment color) with accumulated opacity.

**Data Structures:**
*   **Transforms:** `TextureCoord_1toN` array of `vtkTransform` objects. Each entry $N$ stores the matrix $M_{1 \to N}$ converting texture coordinates from volume 1 to volume $N$.
    *   $M_{1 \to N} = T_{world \to tex, N} \cdot T_{world \to tex, 1}^{-1} \cdot T_{user, N}$
    *   Where $T_{world \to tex}$ is the world-to-texture transform for a specific volume, and $T_{user, N}$ is an optional user-defined transform [S1, unofficial].
*   **Uniforms (GLSL):**
    *   `sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES]`: 3D textures for scalar data.
    *   `sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES]`: 1D textures for opacity transfer functions.
    *   `mat4 P1toPN[MAX_NUMBER_OF_VOLUMES-1]`: Pre-computed $M_{1 \to N}$ matrices.
    *   `vec3 lowBounds[MAX_NUMBER_OF_VOLUMES]`, `vec3 highBounds[MAX_NUMBER_OF_VOLUMES]`: Bounding box limits in texture space.
    *   `vec3 ClippingplanesNormal[MAX_NUMBER_OF_VOLUMES]`, `vec3 ClippingplanesOrigins[MAX_NUMBER_OF_VOLUMES]`: Clipping plane definitions.
    *   `int clipped[MAX_NUMBER_OF_VOLUMES]`: Flags indicating if clipping is active.
    *   `int Number_Of_Volumes`: Count of active volumes.
*   **Internal State:**
    *   `TransformedInput`, `TransformedInput2`: Shallow copies of input data, adjusted for GPU constraints (see Pitfalls).
    *   `LastInput`, `LastInput2`: Pointers to previous inputs to detect changes via `MTime`.
    *   `ClippedCroppingRegionPlanes[6]`: Cropping planes clipped by volume bounds.

**Parameters:**
*   `MAX_NUMBER_OF_ADDITIONAL_VOLUMES`: Compile-time constant. 3 on Apple platforms, 4 otherwise [S1, unofficial].
*   `MAX_NUMBER_OF_VOLUMES`: 4 on Apple platforms, 5 otherwise [S1, unofficial].
*   `ImageSampleDistance`: Base sample distance in XY. Default 1.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `MinimumImageSampleDistance`: Lower bound for adaptive sampling. Default 1.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `MaximumImageSampleDistance`: Upper bound for adaptive sampling. Default 10.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `AutoAdjustSampleDistances`: Boolean (0/1). Enables dynamic adjustment of `ImageSampleDistance` based on render time [S2, unofficial].
*   `MaxMemoryInBytes`: Max GPU memory for textures. Defaults to dedicated video memory; if 0, uses dedicated system memory; if 0, defaults to 128 MB [S2, unofficial].
*   `MaxMemoryFraction`: Fraction of `MaxMemoryInBytes` used. Default 0.75. Range [0.1, 1.0] [S2, unofficial].
*   `ReportProgress`: Boolean. If true, calls `glFinish()` to report progress. Default true [S2, unofficial].
*   `MaskType`: Enum (`BinaryMaskType` or `LabelMapMaskType`). Default `LabelMapMaskType` [S2, unofficial].
*   `MaskBlendFactor`: Blending weight for mask CTF. Default 1.0. Range [0.0, 1.0] [S2, unofficial].
*   `AMRMode`: Adaptive Mesh Refinement mode. Default 0 [S2, unofficial].

**Steps:**

1.  **Initialization:**
    *   Probe GPU info to set `MaxMemoryInBytes`. If dedicated video memory is 0, use dedicated system memory. If both are 0, set to 128 MB [S2, unofficial].
    *   Initialize `ClippedCroppingRegionPlanes` with `VTK_DOUBLE_MAX` for mins and `VTK_DOUBLE_MIN` for maxs [S2, unofficial].

2.  **Input Validation & Update:**
    *   Check for null renderer or volume.
    *   Validate inputs: `Input(0)` and `Input(1)` must not be NULL. If `Cropping` is enabled, ensure `CroppingRegionPlanes[0] < Planes[1]`, `Planes[2] < Planes[3]`, and `Planes[4] < Planes[5]`. If invalid, set `goodSoFar = 0` silently [S2, unofficial].
    *   If `LastInput != Input` or `Input->MTime > TransformedInput->MTime`, update `LastInput` and create a shallow copy into `TransformedInput`. Repeat for Input 2 [S2, unofficial].

3.  **Extent Adjustment (Workaround):**
    *   The GPU mapper cannot handle non-zero extents. For each `TransformedInput`:
        *   Get `extents`, `origin`, `spacing`.
        *   Adjust origin: `origin[cc] += extents[2*cc] * spacing[cc]` for each axis $cc$.
        *   Adjust extents: `extents[2*cc] -= extents[2*cc]` and `extents[2*cc+1] -= extents[2*cc]` (effectively shifting min to 0) [S2, unofficial].
        *   Set new origin and extent on the cloned data [S2, unofficial].

4.  **Transform Setup:**
    *   For each additional volume $N$ (from 1 to `Number_Of_Volumes-1`):
        *   Compute $M_{1 \to N} = T_{world \to tex, N} \cdot T_{world \to tex, 1}^{-1} \cdot T_{user, N}$ [S1, unofficial].
        *   Store in `P1toPN[N-1]` uniform [S1, unofficial].

5.  **Clipping/Cropping Setup:**
    *   Compute `ClippedCroppingRegionPlanes` by intersecting user-defined cropping planes with volume bounds [S1, unofficial].
    *   Convert clipping planes from world coordinates to texture coordinates.
    *   Pass `ClippingplanesNormal`, `ClippingplanesOrigins`, and `clipped` flags to shaders [S1, unofficial].

6.  **Ray Casting Loop (Fragment Shader `trace()`):**
    *   **Initialization:**
        *   Calculate `pos` (entry position in volume 1 texture space), `rayDir` (incremental vector in texture space), `tMax` (max ray length).
        *   Initialize `destColor` via `initialColor()` and `remainOpacity = 1.0 - destColor.a` [S1, unofficial].
    *   **Loop:**
        *   While `inside` is true:
            *   `inside` is true if `t < tMax`, `remainOpacity >= 0.0039` (1/255), and `pos` is within `lowBounds[0]` and `highBounds[0]` [S1, unofficial].
            *   For each additional volume $iii$ from 0 to `Number_Of_Volumes-2`:
                *   Compute transformed position: `posX[iii] = vec3(P1toPN[iii] * vec4(pos, 1.0))` [S1, unofficial].
            *   **Volume 0 Sampling:**
                *   If `pos` is within bounds and not clipped:
                    *   `value[0] = texture3D(dataSetTexture[0], pos)`
                    *   `scalar[0] = scalarFromValue(value[0])`
                    *   `opacity[0] = texture1D(opacityTexture[0], scalar[0])`
            *   **Additional Volumes Sampling:**
                *   For volume $xx$ from 1 to `Number_Of_Volumes-1`:
                    *   If `posX[xx-1]` is within bounds and not clipped:
                        *   `value[xx] = texture3D(dataSetTexture[xx], posX[xx-1])`
                        *   `scalar[xx] = scalarFromValue(value[xx])`
                        *   `opacity[xx] = texture1D(opacityTexture[xx], scalar[xx])`
            *   **Compositing:**
                *   For each volume $vol$ from 0 to `Number_Of_Volumes-1`:
                    *   If `opacity[vol].a > 0.0`:
                        *   `color[vol] = shade(vol, value[vol]) * opacity[vol].a` (Shading uses `P1toPN` to correct increments) [S1, unofficial].
                        *   `destColor = destColor + color[vol] * remainOpacity`
                        *   `remainOpacity = remainOpacity * (1.0 - opacity[vol].a)`
            *   **Advance Ray:**
                *   `pos = pos + rayDir`
                *   `t += 1.0`
    *   **Output:**
        *   `gl_FragColor = destColor`
        *   `gl_FragColor.a = 1.0 - remainOpacity` [S1, unofficial].

7.  **Finalization:**
    *   Stop timer, record `TimeToDraw`.
    *   Invoke `VolumeMapperRenderEndEvent` [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render(renderer, volume):
    if generating_canonical_view:
        canonical_view_render(renderer, volume)
        return

    invoke_event(VolumeMapperRenderStartEvent)
    timer.start()

    if not validate_render(renderer, volume):
        timer.stop()
        return

    # Input Validation
    input0 = get_input(0)
    input1 = get_input(1)
    if input0 is None or input1 is None:
        log_error("Input is NULL")
        return

    # Check Cropping Planes
    if cropping_enabled:
        if (cropping_planes[0] >= cropping_planes[1] or
            cropping_planes[2] >= cropping_planes[3] or
            cropping_planes[4] >= cropping_planes[5]):
            good_so_far = False
            return

    # Update Transformed Inputs if changed
    if (input0 != last_input or input0.mtime > transformed_input.mtime or
        input1 != last_input2 or input1.mtime > transformed_input2.mtime):
        last_input = input0
        transformed_input = shallow_copy(input0)
        adjust_extents(transformed_input) # Shift origin/extents to zero-based

        last_input2 = input1
        transformed_input2 = shallow_copy(input1)
        adjust_extents(transformed_input2)

    # Setup Transforms
    for n in range(1, number_of_volumes):
        m_1_to_n = world_to_tex_transform[n] * inverse(world_to_tex_transform[0]) * user_transform[n]
        set_uniform_mat4("P1toPN", n-1, m_1_to_n)

    # Setup Clipping/Cropping
    compute_clipped_cropping_planes()
    convert_planes_to_texture_space()
    set_uniforms_for_clipping()

    # Execute GPU Render (Shader trace loop)
    gpu_render(renderer, volume)

    timer.stop()
    time_to_draw = timer.elapsed_time()
    invoke_event(VolumeMapperRenderEndEvent)

def adjust_extents(image_data):
    extents = image_data.get_extents()
    origin = image_data.get_origin()
    spacing = image_data.get_spacing()
    for cc in range(3):
        origin[cc] += extents[2*cc] * spacing[cc]
        extents[2*cc] -= extents[2*cc]
        extents[2*cc+1] -= extents[2*cc]
    image_data.set_origin(origin)
    image_data.set_extents(extents)
```

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
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of volumes. The ray traversal loop runs for each pixel, sampling $N$ volumes per step.
*   **Memory Complexity:** Proportional to the sum of the memory requirements of all volume textures. Limited by `MaxMemoryInBytes` (default 128 MB if GPU info unavailable) [S2, unofficial].
*   **Measured Performance:** Performance is heavily dependent on GPU texture sampling bandwidth and shader execution units. Adaptive sampling (`AutoAdjustSampleDistances`) can significantly improve frame rates by reducing sample density when render time exceeds allocated limits [S2, unofficial].
*   **Scaling:** Scales with the number of volumes, but each additional volume adds a transformation matrix lookup and a texture sample per ray step. The limit is `MAX_NUMBER_OF_VOLUMES` (4 on Apple, 5 otherwise) [S1, unofficial].
*   **Strengths:** Simultaneous rendering of multiple volumes with correct spatial alignment via transform matrices; hardware-accelerated sampling; adaptive sampling for performance.
*   **Limitations:** Limited number of supported volumes; requires non-zero extent workaround; `ReportProgress` can cause significant slowdowns due to `glFinish()` blocking [S2, unofficial].
*   **When to use:** When multiple volume datasets need to be rendered together in a single pass with correct spatial registration.
*   **When not to use:** When rendering more than `MAX_NUMBER_OF_VOLUMES` volumes; when small datasets are rendered with `ReportProgress` enabled (causes slowdown); when input data has more than one independent component (hardware unsupported) [S3, unofficial].

## Background
The GPU Multi-Volume Ray Casting Mapper was developed as an extension to the Visualization Toolkit (VTK) to support multi-volume rendering on the GPU. It was modified by Carlos Falcón and Karl Krissian, building upon the original `vtkGPUVolumeRayCastMapper` designed by Ken Martin, Will Schroeder, and Bill Lorensen [S1, unofficial]. The implementation uses GLSL shaders instead of OpenCL for general GPU programming because shaders are faster and supported by more devices [S1, unofficial]. It is used in VTK-based applications for visualizing multiple 3D scalar fields simultaneously.

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
[S3] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-27 (unofficial)
