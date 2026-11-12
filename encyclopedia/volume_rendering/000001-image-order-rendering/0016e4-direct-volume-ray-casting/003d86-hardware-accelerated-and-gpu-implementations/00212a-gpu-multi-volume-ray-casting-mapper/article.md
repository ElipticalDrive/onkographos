# GPU Multi-Volume Ray Casting Mapper

Also called: vtkGPUMultiVolumeRayCastMapper, GPU Multi Volume Ray Cast Mapper, Multi-volume ray casting

## Overview
The GPU Multi-Volume Ray Casting Mapper is a hardware-accelerated direct volume rendering technique that renders multiple 3D scalar fields simultaneously by compositing them along a single ray [S1, unofficial]. It extends the single-volume `vtkGPUVolumeRayCastMapper` to accept multiple `vtkImageData` datasets and `vtkVolumeProperty` objects, using GPU fragment shaders for ray traversal and front-to-back alpha blending [S1, unofficial]. It sets itself apart from its siblings by explicitly managing coordinate transformations between multiple volume textures and compositing them in a single pass, distinguishing it from single-volume mappers and OpenGL-specific subclasses [S1, unofficial][2919-opengl-gpu-multi-volume-ray-cast-mapper.md].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two or more `vtkImageData` datasets (scalar fields) and corresponding `vtkVolumeProperty` objects (transfer functions). The primary input is at port 0; additional inputs are at ports 1 to $N$. A mask input (`vtkImageData`) is optional for label map or binary masking [S1, unofficial][S2, unofficial].
*   **Outputs:** A 2D color buffer (framebuffer) with the composited volume image [S1, unofficial].

**Data Structures:**
*   **Texture Uniforms:**
    *   `sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES]`: 3D textures holding scalar data for each volume [S1, unofficial].
    *   `sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES]`: 1D textures holding opacity transfer function values [S1, unofficial].
    *   `mat4 P1toPN[MAX_NUMBER_OF_VOLUMES-1]`: Transformation matrices converting texture coordinates from the first volume to the $N$-th volume [S1, unofficial].
    *   `vec3 lowBounds[MAX_NUMBER_OF_VOLUMES]`, `vec3 highBounds[MAX_NUMBER_OF_VOLUMES]`: Bounding box limits for each volume in texture space [S1, unofficial].
    *   `vec3 ClippingplanesNormal[MAX_NUMBER_OF_VOLUMES]`, `vec3 ClippingplanesOrigins[MAX_NUMBER_OF_VOLUMES]`: Clipping plane definitions [S1, unofficial].
    *   `int clipped[MAX_NUMBER_OF_VOLUMES]`: Flags indicating if clipping is active [S1, unofficial].
*   **Class Members (CPU-side):**
    *   `TransformedInput`, `TransformedInput2`: Pointers to `vtkImageData` holding shallow copies of input data, adjusted for GPU constraints (e.g., zero-based extents) [S2, unofficial].
    *   `LastInput`, `LastInput2`: Pointers to track input modification time to trigger updates [S2, unofficial].
    *   `ClippedCroppingRegionPlanes[6]`: Array storing cropping planes clipped by volume bounds [S1, unofficial].

**Parameters:**
*   `MAX_NUMBER_OF_ADDITIONAL_VOLUMES`: Compile-time constant. Value is 3 on Apple platforms, 4 on others [S1, unofficial].
*   `MAX_NUMBER_OF_VOLUMES`: Total max volumes. 4 on Apple, 5 otherwise [S1, unofficial].
*   `ImageSampleDistance`: Base sample distance in image space. Default 1.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `MinimumImageSampleDistance`: Lower bound for adaptive sampling. Default 1.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `MaximumImageSampleDistance`: Upper bound for adaptive sampling. Default 10.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `AutoAdjustSampleDistances`: Boolean (0/1). Enables automatic adjustment of `ImageSampleDistance` based on render time [S2, unofficial].
*   `MaxMemoryInBytes`: Maximum GPU memory allocation. Defaults to dedicated video memory, then dedicated system memory, then 128 MB if both are 0 [S2, unofficial].
*   `MaxMemoryFraction`: Fraction of `MaxMemoryInBytes` used for texture storage. Default 0.75. Range [0.1, 1.0] [S2, unofficial].
*   `ReportProgress`: Boolean. If true, uses `glFinish()` to report progress. Should be false for small datasets to avoid slowdown [S2, unofficial].
*   `MaskType`: Enum (`BinaryMaskType` or `LabelMapMaskType`). Default `LabelMapMaskType` [S2, unofficial].
*   `MaskBlendFactor`: Blending weight for mask color transfer function. Default 1.0. Range [0.0, 1.0] [S2, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Probe GPU info to determine `MaxMemoryInBytes`. If dedicated video memory is 0, use dedicated system memory. If both are 0, set to 128 MB. Ignore shared system memory [S2, unofficial].
    *   Initialize `TransformedInput` and `TransformedInput2` as NULL. Initialize `LastInput` and `LastInput2` as NULL [S2, unofficial].
    *   Initialize `ClippedCroppingRegionPlanes` with `VTK_DOUBLE_MAX` for mins and `VTK_DOUBLE_MIN` for maxs [S2, unofficial].

2.  **Input Setup and Validation:**
    *   Connect multiple `vtkImageData` datasets and `vtkVolumeProperty` objects using `SetInput(int port, vtkImageData *input)` and `SetAdditionalProperty(int volNumber, vtkVolumeProperty *property)` [S1, unofficial].
    *   In `ValidateRender`:
        *   Check for null renderer or volume.
        *   Check that inputs are not NULL. If so, set `goodSoFar = 0` [S2, unofficial].
        *   Call `Update()` on all input datasets [S2, unofficial].
        *   Check cropping planes: If `Cropping` is true and any min plane >= max plane (e.g., `Planes[0] >= Planes[1]`), set `goodSoFar = 0` silently [S2, unofficial].

3.  **Transform and Data Preparation:**
    *   **Transform Setup:** For each additional volume $N$ (from 1 to `NUMBER_OF_ADDITIONAL_VOLUMES-1`), compute the transformation matrix $P1toPN$ from the first volume's texture coordinates to the $N$-th volume's texture coordinates. This matrix combines:
        $$ P1toPN = T_{world \to tex_N} \times T_{world \to tex_1}^{-1} \times T_{user} $$
        where $T_{user}$ is a user-defined transform (`SetAdditionalInputUserTransform`) [S1, unofficial].
    *   **Extent Adjustment Workaround:** The GPU mapper cannot handle non-zero extents. For each input:
        *   Get extents, origin, and spacing.
        *   Adjust origin: `new_origin[cc] = origin[cc] + extents[2*cc] * spacing[cc]` [S2, unofficial].
        *   Adjust extents: `extents[2*cc] -= extents[2*cc]`, `extents[2*cc+1] -= extents[2*cc]` (effectively shifting min to 0) [S2, unofficial].
        *   Set new origin and extent in the `TransformedInput` shallow copy [S2, unofficial].
    *   **Clipping/Cropping Init:** Compute `ClippedCroppingRegionPlanes` by clipping user-defined cropping planes against volume bounds. Convert these planes from world coordinates to texture coordinates just before rendering and pass them to shaders via uniform variables [S1, unofficial].

4.  **Render Entry:**
    *   If `GeneratingCanonicalView` is true, call `CanonicalViewRender` (skips start/end events and timing) and return [S2, unofficial].
    *   Otherwise, invoke `VolumeMapperRenderStartEvent`, start timer, validate render, call `GPURender`, stop timer, record time, and invoke `VolumeMapperRenderEndEvent` [S2, unofficial].

5.  **Shader Execution (Fragment Shader `trace()`):**
    *   **Initialization:**
        *   Initialize `pos` (entry position in volume 1 texture space), `rayDir` (incremental vector in texture space), `tMax` (max ray length), and `destColor` (initial color via `initialColor()`) [S1, unofficial].
        *   Set `remainOpacity = 1.0 - destColor.a` [S1, unofficial].
    *   **Loop:** While `inside` is true:
        *   `inside` is true if `t < tMax`, `remainOpacity >= 0.0039` (1/255), and `pos` is within `lowBounds[0]` and `highBounds[0]` [S1, unofficial].
        *   **Transform Positions:** For each additional volume `iii` from 0 to `Number_Of_Volumes-2`, compute transformed position:
            $$ posX[iii] = \text{vec3}(P1toPN[iii] \times \text{vec4}(pos, 1.0)) $$
            [S1, unofficial].
        *   **Sample Volume 0:** If `pos` is within bounds and not clipped (or clipping check passes):
            *   Sample scalar: `value[0] = texture3D(dataSetTexture[0], pos)` [S1, unofficial].
            *   Compute scalar: `scalar[0] = scalarFromValue(value[0])` [S1, unofficial].
            *   Sample opacity: `opacity[0] = texture1D(opacityTexture[0], scalar[0])` [S1, unofficial].
        *   **Sample Volumes 1 to N:** For each volume `xx` from 1 to `Number_Of_Volumes-1`:
            *   If `posX[xx-1]` is within bounds and not clipped:
                *   Sample scalar: `value[xx] = texture3D(dataSetTexture[xx], posX[xx-1])` [S1, unofficial].
                *   Compute scalar: `scalar[xx] = scalarFromValue(value[xx])` [S1, unofficial].
                *   Sample opacity: `opacity[xx] = texture1D(opacityTexture[xx], scalar[xx])` [S1, unofficial].
        *   **Compositing:**
            *   For each volume `vol` from 0 to `Number_Of_Volumes-1`:
                *   If `opacity[vol].a > 0.0`:
                    *   Compute color: `color[vol] = shade(vol, value[vol]) * opacity[vol].a` [S1, unofficial].
                    *   Update destination color: `destColor = destColor + color[vol] * remainOpacity` [S1, unofficial].
                    *   Update remaining opacity: `remainOpacity = remainOpacity * (1.0 - opacity[vol].a)` [S1, unofficial].
        *   **Advance Ray:**
            *   `pos = pos + rayDir` [S1, unofficial].
            *   `t += 1.0` [S1, unofficial].
    *   **Final Output:**
        *   `gl_FragColor = destColor` [S1, unofficial].
        *   `gl_FragColor.a = 1.0 - remainOpacity` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Shader trace() function structure
def trace(pos, rayDir, tMax, lowBounds, highBounds, P1toPN, dataSetTexture, opacityTexture, clippingFlags):
    remainOpacity = 1.0
    destColor = vec4(0.0, 0.0, 0.0, 0.0)
    t = 0.0
    
    # Pre-calculate transformed positions for all additional volumes
    posX = []
    for iii in range(len(P1toPN)):
        posX.append(vec3(P1toPN[iii] * vec4(pos, 1.0)))
        
    while t < tMax and remainOpacity >= 0.0039 and is_inside(pos, lowBounds[0], highBounds[0]):
        # Sample Volume 0
        if is_inside(pos, lowBounds[0], highBounds[0]) and not is_clipped(pos, clippingFlags[0]):
            value0 = texture3D(dataSetTexture[0], pos)
            scalar0 = scalarFromValue(value0)
            opacity0 = texture1D(opacityTexture[0], scalar0).a
            if opacity0 > 0.0:
                color0 = shade(0, value0) * opacity0
                destColor = destColor + color0 * remainOpacity
                remainOpacity = remainOpacity * (1.0 - opacity0)
        
        # Sample Additional Volumes
        for xx in range(1, len(P1toPN) + 1):
            vol_idx = xx
            p_vol = posX[xx-1]
            if is_inside(p_vol, lowBounds[vol_idx], highBounds[vol_idx]) and not is_clipped(p_vol, clippingFlags[vol_idx]):
                value_vol = texture3D(dataSetTexture[vol_idx], p_vol)
                scalar_vol = scalarFromValue(value_vol)
                opacity_vol = texture1D(opacityTexture[vol_idx], scalar_vol).a
                if opacity_vol > 0.0:
                    color_vol = shade(vol_idx, value_vol) * opacity_vol
                    destColor = destColor + color_vol * remainOpacity
                    remainOpacity = remainOpacity * (1.0 - opacity_vol)
        
        # Advance Ray
        pos = pos + rayDir
        t += 1.0
        # Update posX for next iteration? 
        # Note: In GLSL, posX is recalculated or updated based on rayDir transformation.
        # The source implies posX is updated according to rayDir.
        # For implementation, we assume P1toPN transforms the ray direction as well or pos is updated.
        # Simplified: recompute posX if rayDir changes in texture space for other volumes.
        # However, source says "pos and posX[] is updated according to the ray direction (rayDir)".
        # This implies posX[iii] += transform(rayDir, P1toPN[iii])? 
        # Or simply: posX[iii] = vec3(P1toPN[iii] * vec4(pos, 1.0)) using new pos.
        for iii in range(len(P1toPN)):
            posX[iii] = vec3(P1toPN[iii] * vec4(pos, 1.0))

    return destColor, 1.0 - remainOpacity
```

## Subtopics

### [OpenGL GPU Multi Volume Ray Cast Mapper](2919-opengl-gpu-multi-volume-ray-cast-mapper.md)
An OpenGL-specific subclass handling hardware extension checks, OpenGL object lifecycle, and clipping/cropping plane setup. It manages fragment shader programs, framebuffer objects, and texture uploads for rendering multiple volumes. Choose this when implementing the mapper in an OpenGL context.

### [Label Map Masking for Volume Rendering](4788-label-map-masking-for-volume-rendering.md)
Confines rendering to specific regions using binary or multi-label masks, blending standard and mask-specific color transfer functions. Choose this when region-specific visualization with distinct transfer functions is required.

### [GPU Volume Ray Casting (VTK Composite Blend)](6912-gpu-volume-ray-casting-vtk-composite-blend.md)
A hardware-accelerated direct volume rendering technique using the **Composite** blend mode for standard alpha blending. It sets itself apart by explicitly utilizing composite blending, distinguishing it from the multi-volume mapper and Slicer-specific mapper. Choose this for standard single-volume rendering with composite blending.

### [vtkSlicerGPURayCastVolumeMapper](7450-vtkslicergpuraycastvolumemapper.md)
Renders a volume using ray casting on a 3D texture with specific parameters for distance-based color blending and ICPE scaling. Choose this when using 3D Slicer's specific rendering enhancements.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the ray and the number of volumes. Adaptive sampling (`AutoAdjustSampleDistances`) can reduce sample count for complex scenes or to meet time budgets [S2, unofficial].
*   **Memory Complexity:** Proportional to the size of the 3D textures for all input volumes. Limited by `MaxMemoryInBytes` and `MaxMemoryFraction` [S2, unofficial].
*   **Measured Performance:** Performance depends on GPU texture sampling speed and fragment shader complexity. Hardware loop limits may require nested loop structures for large numbers of samples/volumes [S1, unofficial].
*   **Scaling:** Scales with the number of volumes, but limited by `MAX_NUMBER_OF_VOLUMES` (4-5) [S1, unofficial]. Adaptive sampling helps scale with resolution and scene complexity [S2, unofficial].
*   **Strengths:** Hardware-accelerated, supports multiple volumes in a single pass, adaptive sampling, masking support [S1, unofficial][S2, unofficial].
*   **Limitations:** Limited number of volumes, requires 3D texture support, non-zero extents require workaround, `ReportProgress` can cause slowdowns [S1, unofficial][S2, unofficial].
*   **When to Use:** When rendering multiple co-registered volumes simultaneously on the GPU with VTK.
*   **When Not to Use:** When rendering more than `MAX_NUMBER_OF_VOLUMES` volumes, or when non-zero extents cannot be adjusted.

## Background
The GPU Multi-Volume Ray Casting Mapper was developed by Carlos Falcón and Karl Krissian as an extension to the Visualization Toolkit (VTK) to support multi-volume rendering on the GPU [S1, unofficial]. It builds on the single-volume `vtkGPUVolumeRayCastMapper` by adding support for multiple input datasets and coordinate transformations between them [S1, unofficial]. It uses GLSL shaders for ray traversal and compositing, chosen for their speed and broad device support compared to OpenCL [S1, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
[S3] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-07 (unofficial)
