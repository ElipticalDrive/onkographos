# GPU Multi-Volume Ray Casting Mapper

Also called: vtkGPUMultiVolumeRayCastMapper, GPU Multi Volume Ray Cast Mapper, Multi-volume ray casting

## Overview
The GPU Multi-Volume Ray Casting Mapper is a volume rendering technique that renders multiple volume datasets simultaneously by extending the single-volume `vtkGPUVolumeRayCastMapper` to accept multiple `vtkImageData` datasets and `vtkVolumeProperty` objects, using GPU shaders for ray traversal and compositing [S1, S2, S3, unofficial]. It solves the problem of visualizing co-registered or spatially aligned multi-modal volumetric data (e.g., CT and MRI) in a single pass by compositing multiple 3D scalar fields along a single ray, using coordinate transformation matrices to align samples from different volumes to a common ray space [S1, S3, unofficial]. It sets itself apart from its siblings by explicitly supporting the simultaneous rendering of two or more input datasets with distinct transfer functions and spatial alignments, distinguishing it from the single-volume `vtkGPUVolumeRayCastMapper` and the OpenGL-specific subclass `vtkOpenGLGPUVolumeRayCastMapper` [S1, S2, 2919-opengl-gpu-multi-volume-ray-cast-mapper.md, unofficial].

## How it works
The method extends the direct volume ray casting pipeline defined in [Direct volume ray casting](19-direct-volume-ray-casting.md) to handle $N$ volumes. The core engine remains the four-stage loop (Ray Generation, Sampling, Classification, Compositing), but the sampling and compositing stages are modified to query multiple textures and apply coordinate transformations.

### Inputs and Outputs
**Inputs:**
1.  **Primary Volume:** `vtkImageData` dataset (Input Port 0).
2.  **Additional Volumes:** Up to `MAX_NUMBER_OF_ADDITIONAL_VOLUMES` (typically 3 or 4) `vtkImageData` datasets (Input Ports 1 to $N-1$) [S1, S2, unofficial].
3.  **Properties:** `vtkVolumeProperty` objects for each volume, providing color and opacity transfer functions [S1, unofficial].
4.  **Transforms:** User-defined `vtkTransform` objects for each additional volume to align them with the primary volume's texture coordinate space [S1, unofficial].
5.  **Mask (Optional):** A `vtkImageData` mask input to confine rendering to specific regions [S2, unofficial].

**Outputs:**
*   A final RGBA color buffer representing the composited view of all volumes [S1, unofficial].

### Data Structures
**C++ Class Members (`vtkGPUMultiVolumeRayCastMapper`):**
*   `NUMBER_OF_ADDITIONAL_VOLUMES`: Integer count of extra volumes.
*   `Property[MAX_NUMBER_OF_ADDITIONAL_VOLUMES]`: Array of `vtkVolumeProperty*` pointers [S1, unofficial].
*   `AdditionalInputUserTransform[MAX_NUMBER_OF_ADDITIONAL_VOLUMES]`: Array of `vtkTransform*` pointers for user-defined alignments [S1, unofficial].
*   `TransformedInput`, `TransformedInput2`: Pointers to `vtkImageData` storing the first and second transformed input datasets (cloned/adjusted for GPU constraints) [S2, unofficial].
*   `LastInput`, `LastInput2`: Pointers to track data modification times to trigger updates [S2, unofficial].
*   `ClippedCroppingRegionPlanes[6]`: Array of doubles for X, Y, Z min/max clipping planes [S1, S2, unofficial].
*   `MaskInput`: `vtkImageData*` for label map or binary masking [S2, unofficial].
*   `MaskType`: Enum (`BinaryMaskType` or `LabelMapMaskType`) [S2, unofficial].
*   `MaskBlendFactor`: Float (default 1.0) controlling blending weight for mask-specific CTFs [S2, unofficial].

**GPU Uniforms (GLSL):**
*   `sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES]`: 3D textures for scalar data [S1, unofficial].
*   `sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES]`: 1D textures for opacity transfer functions [S1, unofficial].
*   `mat4 P1toPN[MAX_NUMBER_OF_VOLUMES-1]`: Transformation matrices converting texture coordinates from Volume 1 to Volume $N$ [S1, unofficial].
*   `vec3 lowBounds[MAX_NUMBER_OF_VOLUMES]`, `highBounds[MAX_NUMBER_OF_VOLUMES]`: Bounding box limits for each volume [S1, unofficial].
*   `int Number_Of_Volumes`: Integer count of active volumes [S1, unofficial].
*   `vec3 ClippingplanesNormal[MAX_NUMBER_OF_VOLUMES]`, `ClippingplanesOrigins[MAX_NUMBER_OF_VOLUMES]`: Clipping plane definitions [S1, unofficial].
*   `int clipped[MAX_NUMBER_OF_VOLUMES]`: Flags indicating if clipping is active [S1, unofficial].

### Parameters
*   `MAX_NUMBER_OF_ADDITIONAL_VOLUMES`: Compile-time constant. Value is 3 on Apple platforms, 4 on others [S1, unofficial].
*   `MAX_NUMBER_OF_VOLUMES`: Total max volumes. 4 on Apple, 5 otherwise [S1, unofficial].
*   `AutoAdjustSampleDistances`: Boolean (default 1). Enables dynamic adjustment of `ImageSampleDistance` [S2, unofficial].
*   `ImageSampleDistance`: Base sample distance in image space (default 1.0). Clamped to [0.1, 100.0] [S2, unofficial].
*   `MinimumImageSampleDistance`: Lower bound for adaptive sampling (default 1.0) [S2, unofficial].
*   `MaximumImageSampleDistance`: Upper bound for adaptive sampling (default 10.0) [S2, unofficial].
*   `SampleDistance`: Distance between sample points along the ray (default 1.0) [S2, unofficial].
*   `MaxMemoryInBytes`: Maximum GPU memory allocation. Probed from GPU info; defaults to 128 MB if unavailable [S2, unofficial].
*   `MaxMemoryFraction`: Fraction of `MaxMemoryInBytes` used for textures (default 0.75). Clamped to [0.1, 1.0] [S2, unofficial].
*   `ReportProgress`: Boolean (default true). Blocks rendering with `glFinish()` to report progress. Should be false for small datasets to avoid slowdown [S2, unofficial].
*   `FinalColorWindow` (default 1.0), `FinalColorLevel` (default 0.5): Scaling for final color output [S2, unofficial].
*   `GlobalAlpha` (default 1.0): Overall alpha for blending with polygons [S3, unofficial].
*   `AMRMode`: Boolean (default 0). Enables Adaptive Mesh Refinement mode [S2, unofficial].

### Algorithm Steps

1.  **Initialization and Validation:**
    *   Probe GPU memory to set `MaxMemoryInBytes`. If dedicated memory is 0, use dedicated system memory. If both are 0, default to 128 MB [S2, unofficial].
    *   Validate inputs: Check for null renderer, null volume, and null input datasets. If `Cropping` is enabled, verify that `CroppingRegionPlanes` define a valid volume (min < max for all axes). If invalid, set `goodSoFar = 0` silently [S2, unofficial].
    *   Check color channels (currently requires 3 channels, though this check may be commented out in some implementations) [S2, unofficial].

2.  **Data Preparation and Transformation:**
    *   If `LastInput` or `LastInput2` differ from current inputs, or if input modification time (`MTime`) exceeds transformed input `MTime`, create shallow copies into `TransformedInput` and `TransformedInput2` [S2, unofficial].
    *   **Extent Adjustment Workaround:** GPU ray casting requires extents starting from zero. Adjust the origin by adding `extents[2*cc] * spacing[cc]`. Adjust extents by subtracting `extents[2*cc]` from both min and max for each axis `cc` [S2, unofficial].
    *   **Compute Transform Matrices:** For each additional volume $N$, compute the transformation matrix $P1toPN$ to convert texture coordinates from Volume 1 to Volume $N$. The matrix combines:
        $$ P1toPN = T_{world \to tex_N} \times T_{world \to tex_1}^{-1} \times T_{user} $$
        where $T_{user}$ is the user-defined transform (`AdditionalInputUserTransform`) [S1, unofficial].
    *   **Clipping/Cropping:** Convert clipping and cropping planes from world coordinates to texture coordinates just before rendering. Pass them to shaders via uniforms. Compute `ClippedCroppingRegionPlanes` by intersecting cropping planes with volume bounds [S1, unofficial].

3.  **Shader Execution (Ray Traversal Loop):**
    The core logic resides in the fragment shader `vtkGPUVolumeRayCastMapper_CompositeFS`. The `trace()` function executes the following loop:

    *   **Initialization:**
        *   Calculate initial ray position `pos` for Volume 1.
        *   Calculate initial positions `posX[]` for additional volumes using `posX[iii] = P1toPN[iii] * vec4(pos, 1.0)` [S1, unofficial].
        *   Initialize `destColor` (via `initialColor()`) and `remainOpacity = 1.0 - destColor.a` [S1, unofficial].
        *   Set `t = 0.0`, `tMax` (max ray length) [S1, unofficial].

    *   **Loop:** `while (inside)`
        *   **Inside Condition:** `t < tMax` AND `remainOpacity >= 0.0039` (1/255) AND `pos` is within `lowBounds[0]` and `highBounds[0]` [S1, unofficial].
        *   **Update Positions:**
            *   `pos = pos + rayDir`
            *   For each additional volume `iii`: `posX[iii] = posX[iii] + rayDir_transformed` (where `rayDir_transformed` is computed using `P1toPN` and `rayDir`) [S1, unofficial].
        *   **Bounding Box Test:** Check if `pos` is within `lowBounds[0]`/`highBounds[0]` and each `posX[iii]` is within its respective volume bounds [S1, unofficial].
        *   **Clipping Test:** Check if `pos` and `posX[iii]` are inside the clipped regions. If outside, skip sampling for that volume [S1, unofficial].
        *   **Sampling and Classification:**
            *   For Volume 0: If inside bounds and not clipped, sample `value[0] = texture3D(dataSetTexture[0], pos)`. Compute `scalar[0] = scalarFromValue(value[0])`. Sample `opacity[0] = texture1D(opacityTexture[0], scalar[0])` [S1, unofficial].
            *   For Volumes $1$ to $N-1$: If `posX[xx-1]` is inside bounds and not clipped, sample `value[xx] = texture3D(dataSetTexture[xx], posX[xx-1])`. Compute `scalar[xx]`. Sample `opacity[xx] = texture1D(opacityTexture[xx], scalar[xx])` [S1, unofficial].
        *   **Compositing:**
            *   For each volume `vol` from 0 to $N-1$:
                *   If `opacity[vol].a > 0.0`:
                    *   Compute color: `color[vol] = shade(vol, value[vol]) * opacity[vol].a` (shading may use `InitShade()` with corrected increments) [S1, unofficial].
                    *   Update destination color: `destColor = destColor + color[vol] * remainOpacity` [S1, unofficial].
                    *   Update remaining opacity: `remainOpacity = remainOpacity * (1.0 - opacity[vol].a)` [S1, unofficial].
        *   **Advance Ray:** `t += 1.0` [S1, unofficial].

    *   **Final Output:**
        *   `gl_FragColor = destColor`
        *   `gl_FragColor.a = 1.0 - remainOpacity` [S1, unofficial].

4.  **Masking (Optional):**
    *   If a mask input is provided, the mapper uses `MaskType` (Binary or LabelMap) to determine how to blend colors.
    *   For LabelMap masks, `MaskBlendFactor` controls the blending weight between the standard color transfer function and the mask-specific color transfer function [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the GPU Ray Traversal Loop (GLSL-style logic)

def trace_ray(pos, ray_dir, t_max, bounds, clipping_planes, textures, transforms, opacity_tfs, shading_func):
    """
    pos: vec3, initial position in Volume 1 texture space
    ray_dir: vec3, ray direction in Volume 1 texture space
    t_max: float, max ray length
    bounds: list of (low, high) vec3 pairs for each volume
    clipping_planes: list of plane definitions for each volume
    textures: list of 3D texture samplers
    transforms: list of mat4 P1toPN matrices (index 0 is identity for Vol 0)
    opacity_tfs: list of 1D texture samplers
    shading_func: function to compute color from scalar value and position
    """
    
    # Initialize positions for all volumes
    positions = [pos]
    for i in range(1, len(transforms)):
        # Transform initial position to Volume i's texture space
        p = transforms[i] * vec4(pos, 1.0)
        positions.append(p.xyz)
        
    # Calculate ray directions for additional volumes
    ray_dirs = [ray_dir]
    for i in range(1, len(transforms)):
        # Transform ray direction (w=0)
        d = transforms[i] * vec4(ray_dir, 0.0)
        ray_dirs.append(d.xyz)
        
    dest_color = vec4(0.0, 0.0, 0.0, 0.0)
    remain_opacity = 1.0
    
    t = 0.0
    
    # Hardware loop limit workaround: nested loop or break condition
    while t < t_max and remain_opacity > 0.0039:
        inside = True
        
        # Check bounds and clipping for all volumes
        for i in range(len(positions)):
            if not (bounds[i][0] <= positions[i] <= bounds[i][1]):
                inside = False
                break
            if is_clipped(positions[i], clipping_planes[i]):
                inside = False
                break
                
        if not inside:
            # Advance all positions
            for i in range(len(positions)):
                positions[i] += ray_dirs[i]
            t += 1.0
            continue
            
        # Sample and composite for each volume
        for i in range(len(positions)):
            # Sample scalar value
            value = texture3D(textures[i], positions[i])
            scalar = scalar_from_value(value)
            
            # Sample opacity
            opacity = texture1D(opacity_tfs[i], scalar).a
            
            if opacity > 0.0:
                # Compute color (with shading if enabled)
                color = shading_func(i, value, positions[i]) * opacity
                
                # Front-to-back compositing
                dest_color = dest_color + color * remain_opacity
                remain_opacity = remain_opacity * (1.0 - opacity)
                
        # Advance ray
        for i in range(len(positions)):
            positions[i] += ray_dirs[i]
        t += 1.0
        
    return dest_color, 1.0 - remain_opacity
```

### Edge Cases and Pitfalls
*   **Hardware Loop Limits:** GPU fragment shaders have limits on loop iterations. A single loop over all samples and volumes may exceed these limits. The implementation uses a nested structure or early termination (`remainOpacity` threshold) to mitigate this [S1, unofficial].
*   **Non-Zero Extents:** The underlying GPU ray casting logic cannot handle volume extents starting from non-zero values. A workaround adjusts the origin and shifts extents to start from zero before uploading to the GPU [S2, unofficial].
*   **Invalid Cropping:** If cropping planes define a region with zero or negative volume (e.g., min >= max), the render fails silently (`goodSoFar = 0`) without an error message [S2, unofficial].
*   **ReportProgress Overhead:** Setting `ReportProgress` to true uses `glFinish()`, which blocks the GPU. This significantly slows down rendering of small datasets and should be disabled for them [S2, unofficial].
*   **FinalColorWindow:** Setting `FinalColorWindow` to a negative value reverses the applied color values. It cannot be null [S2, unofficial].
*   **Component Count:** Hardware rendering is unsupported if the input data has more than one independent component (e.g., multi-component vectors) or if the hardware lacks required extensions [S3, unofficial].

## Subtopics

### [OpenGL GPU Multi Volume Ray Cast Mapper](2919-opengl-gpu-multi-volume-ray-cast-mapper.md)
The OpenGL GPU Multi Volume Ray Cast Mapper is an OpenGL-specific subclass of the GPU Multi-Volume Ray Casting Mapper that handles hardware extension checks, OpenGL object lifecycle management, and clipping/cropping plane setup for GPU-accelerated volume rendering [2919-opengl-gpu-multi-volume-ray-cast-mapper.md]. It serves as the concrete implementation responsible for drawing the volume ray cast image to the screen by managing fragment shader programs, framebuffer objects, and texture uploads required for rendering multiple volumes simultaneously [2919-opengl-gpu-multi-volume-ray-cast-mapper.md]. It builds on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) by providing the specific OpenGL API bindings and state management necessary for hardware-accelerated ray casting [2919-opengl-gpu-multi-volume-ray-cast-mapper.md].

### [Label Map Masking for Volume Rendering](4788-label-map-masking-for-volume-rendering.md)
Label Map Masking is a technique that confines volume rendering to specific regions using either a binary mask or a multi-label map, supporting blending between standard and mask-specific color transfer functions [4788-label-map-masking-for-volume-rendering.md]. It sets itself apart from sibling mappers by explicitly defining how mask values (binary or labeled) dictate the selection of color transfer functions (CTFs) and their blending weights, rather than simply acting as a visibility gate [4788-label-map-masking-for-volume-rendering.md].

### [GPU Volume Ray Casting (VTK Composite Blend)](6912-gpu-volume-ray-casting-vtk-composite-blend.md)
GPU Volume Ray Casting (VTK Composite Blend) is a hardware-accelerated direct volume rendering technique implemented in VTK via `vtkGPUVolumeRayCastMapper` that performs ray casting on the GPU to visualize 3D scalar fields [6912-gpu-volume-ray-casting-vtk-composite-blend.md]. It transforms raw volumetric data into a 3D environment by accumulating color and opacity along rays using composite blending [6912-gpu-volume-ray-casting-vtk-composite-blend.md]. It sets itself apart from its siblings by explicitly utilizing the **Composite** blend mode for standard alpha blending during the ray casting process, distinguishing it from the OpenGL-specific multi-volume mapper [2919-opengl-gpu-multi-volume-ray-cast-mapper.md] and the Slicer-specific mapper [7450-vtkslicergpuraycastvolumemapper.md].

### [vtkSlicerGPURayCastVolumeMapper](7450-vtkslicergpuraycastvolumemapper.md)
Renders a volume using ray casting on a 3D texture [7450-vtkslicergpuraycastvolumemapper.md]. It builds on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) to provide a specific implementation for the 3D Slicer environment, supporting adaptive sampling, masking, and AMR modes via the parent's pipeline hooks. It is distinguished from its siblings by its specific integration with CTK transfer functions and its handling of specific OpenGL uniform layouts for ray casting parameters [7450-vtkslicergpuraycastvolumemapper.md].

## Operating characteristics
*   **Time Complexity:** $O(P \cdot N \cdot S)$, where $P$ is the number of pixels, $N$ is the number of volumes, and $S$ is the number of samples per ray. Adaptive sampling reduces $S$ dynamically based on `AutoAdjustSampleDistances` [S2, unofficial].
*   **Memory Complexity:** $O(V \cdot R^3)$, where $V$ is the number of volumes and $R$ is the resolution of each volume. Memory is constrained by `MaxMemoryInBytes` and `MaxMemoryFraction` [S2, unofficial].
*   **Measured Performance:** Performance is heavily dependent on GPU hardware and the number of volumes. Rendering multiple volumes increases the number of texture lookups per sample, potentially reducing frame rates compared to single-volume rendering [S1, unofficial].
*   **Scaling:** Scales linearly with the number of volumes and resolution. Adaptive sampling helps maintain performance at higher resolutions by reducing $S$ [S2, unofficial].
*   **Strengths:** Supports simultaneous visualization of multiple co-registered volumes with distinct transfer functions. Efficient GPU-based implementation using fragment shaders. Supports masking and adaptive sampling [S1, S2, unofficial].
*   **Limitations:** Limited by GPU memory and loop iteration limits. Requires careful handling of coordinate transformations and extent adjustments. Silent failures for invalid cropping planes [S1, S2, unofficial].
*   **When to Use:** When visualizing multi-modal volumetric data (e.g., CT/MRI fusion) where spatial alignment and simultaneous opacity blending are required.
*   **When Not to Use:** For single-volume rendering where simpler mappers suffice. For very large datasets where GPU memory is insufficient, or when hardware does not support the required OpenGL extensions [S3, unofficial].

## Background
The GPU Multi-Volume Ray Casting Mapper was developed as an extension to the Visualization Toolkit (VTK) to support multi-volume rendering on the GPU. It was modified by Carlos Falcón and Karl Krissian from the Computer Vision and Robotics Research Group (CTIM) [S1, unofficial]. The implementation leverages GLSL shaders for ray traversal and compositing, chosen over OpenCL for better performance and broader device support [S1, unofficial]. It builds upon the single-volume `vtkGPUVolumeRayCastMapper` by adding support for multiple input datasets, user-defined transformations, and coordinate space alignment [S1, S2, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
[S3] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-27 (unofficial)
