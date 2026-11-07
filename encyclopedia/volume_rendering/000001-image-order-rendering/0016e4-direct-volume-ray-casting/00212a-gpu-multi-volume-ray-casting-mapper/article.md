# GPU Multi-Volume Ray Casting Mapper

Also called: vtkGPUMultiVolumeRayCastMapper, GPU Multi Volume Ray Cast Mapper, Multi-volume ray casting

## Overview
The GPU Multi-Volume Ray Casting Mapper is a volume rendering technique that renders multiple volume datasets simultaneously by extending the single-volume `vtkGPUVolumeRayCastMapper` to accept multiple `vtkImageData` datasets and `vtkVolumeProperty` objects, using GPU shaders for ray traversal and compositing [S1, S2, unofficial]. It solves the problem of visualizing co-registered or spatially aligned multi-modal volumetric data (e.g., CT and MRI) in a single pass by compositing multiple 3D scalar fields along a single ray, using coordinate transformation matrices to align samples from different volumes to a common ray space [S1, unofficial]. It sets itself apart from its siblings by explicitly managing the transformation of ray positions across multiple distinct texture coordinate spaces within the fragment shader loop, whereas standard mappers assume a single texture space [S1, unofficial].

## How it works
The method implements the emission-absorption optical model, computing the light intensity reaching the eye by integrating radiative energy along a ray through the scalar volume [S1, S5, S9, unofficial]. The process follows a strict pipeline for each pixel: ray generation, sampling, classification (transfer function application), shading, and compositing [S4, S34, unofficial].

### Inputs and Outputs
**Inputs:**
1.  **Primary Volume:** A `vtkImageData` dataset (Input Port 0) containing the primary scalar field.
2.  **Additional Volumes:** Up to `MAX_NUMBER_OF_ADDITIONAL_VOLUMES` (3 on Apple platforms, 4 otherwise) `vtkImageData` datasets connected to subsequent input ports [S1, unofficial].
3.  **Properties:** A `vtkVolumeProperty` for the primary volume, and an array of `vtkVolumeProperty` objects (`Property[MAX_NUMBER_OF_ADDITIONAL_VOLUMES]`) for the additional volumes, defining their transfer functions and shading parameters [S1, unofficial].
4.  **Transforms:** User-defined `vtkTransform` objects (`AdditionalInputUserTransform[i]`) for each additional volume to align them with the primary volume's texture coordinate space [S1, unofficial].
5.  **Clipping/Cropping:** Up to 6 cropping planes and 1 clipping plane per volume, defined in world coordinates [S1, unofficial].

**Outputs:**
*   A single RGBA color value per pixel (`gl_FragColor`), representing the front-to-back composite of all volumes [S1, unofficial].

### Data Structures
**Class Members (`vtkGPUMultiVolumeRayCastMapper`):**
*   `NUMBER_OF_ADDITIONAL_VOLUMES`: Integer, typically 3 or 4 depending on platform [S1, unofficial].
*   `TransformedInput`, `TransformedInput2`: Pointers to `vtkImageData` storing the first and second transformed input datasets respectively [S2, unofficial].
*   `LastInput`, `LastInput2`: Pointers to `vtkImageData` used to check if input data has changed since the last `Render()` call [S2, unofficial].
*   `ClippedCroppingRegionPlanes[6]`: Array of doubles storing cropping planes clipped by volume bounds. Indices 0-1: X min/max, 2-3: Y min/max, 4-5: Z min/max [S1, S2, unofficial].
*   `TextureCoord_1toN[]`: Array of `vtkTransform` objects holding transformation matrices that convert texture coordinates of the first volume to all other volumes [S1, unofficial].
*   `MaskInput`: Optional `vtkImageData` for label map masking [S2, unofficial].
*   `MaskType`: Enum specifying mask type (`BinaryMaskType` or `LabelMapMaskType`) [S2, unofficial].
*   `MaskBlendFactor`: Float [0.0, 1.0], blending weight for mask color transfer function [S2, unofficial].

**Shader Uniforms:**
*   `sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES]`: 3D textures for scalar data [S1, unofficial].
*   `sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES]`: 1D textures for opacity transfer functions [S1, unofficial].
*   `mat4 P1toPN[MAX_NUMBER_OF_VOLUMES-1]`: Transformation matrices from volume 1 to N [S1, unofficial].
*   `vec3 lowBounds[MAX_NUMBER_OF_VOLUMES]`, `vec3 highBounds[MAX_NUMBER_OF_VOLUMES]`: Bounding box info for each volume, modified by cropping [S1, unofficial].
*   `int Number_Of_Volumes`: Count of active volumes [S1, unofficial].
*   `vec3 ClippingplanesNormal[MAX_NUMBER_OF_VOLUMES]`, `vec3 ClippingplanesOrigins[MAX_NUMBER_OF_VOLUMES]`: Clipping plane definitions in texture space [S1, unofficial].
*   `int clipped[MAX_NUMBER_OF_VOLUMES]`: Flags indicating if clipping is active for each volume [S1, unofficial].

### Parameters
*   `MAX_NUMBER_OF_ADDITIONAL_VOLUMES`: Compile-time constant. 3 on Apple, 4 on others [S1, unofficial].
*   `AutoAdjustSampleDistances`: Boolean (0/1). Enables automatic adjustment of sample distances. Default: 1 (true) [S1, S2, unofficial].
*   `ImageSampleDistance`: Base sample distance in image space. Default: 1.0. Clamped to [0.1, 100.0] [S2, unofficial].
*   `MinimumImageSampleDistance`: Lower bound for adaptive sample distance. Default: 1.0. Clamped to [0.1, 100.0] [S2, unofficial].
*   `MaximumImageSampleDistance`: Upper bound for adaptive sample distance. Default: 10.0. Clamped to [0.1, 100.0] [S2, unofficial].
*   `SampleDistance`: Distance between sample points along the ray. Default: 1.0 [S2, unofficial].
*   `MaxMemoryInBytes`: Maximum GPU memory allocation. Probed from GPU info; defaults to 128MB if unavailable [S1, S2, unofficial].
*   `MaxMemoryFraction`: Fraction of `MaxMemoryInBytes` used. Default: 0.75. Range [0.1, 1.0] [S1, S2, unofficial].
*   `ReportProgress`: Boolean. Blocks rendering with `glFinish()` to report progress. Default: true. Should be false for small datasets to avoid slowdown [S1, S2, unofficial].
*   `FinalColorWindow`: Window value for final color scaling. Default: 1.0 [S2, unofficial].
*   `FinalColorLevel`: Level value for final color scaling. Default: 0.5 [S2, unofficial].
*   `GlobalAlpha`: Overall alpha for volume rendering result. Default: 1.0 [S3, unofficial].
*   `Clipping`: Enables/disables clipping. Default: 0 [S3, unofficial].
*   `Technique` / `TechniqueFg`: Selects rendering technique for background/foreground. Default: 0 [S3, unofficial].
*   `ColorOpacityFusion`: Sets multi-volume color/opacity fusion method. Default: 0 [S3, unofficial].
*   `FgBgRatio`: Ratio between foreground and background volumes. Default: 0.5 [S3, unofficial].
*   `DepthPeelingThreshold`: Threshold for depth peeling. Not given in the sources.
*   `AMRMode`: Adaptive Mesh Refinement mode. Default: 0 [S1, S2, unofficial].
*   `SmallVolumeRender`: Boolean indicating if volume is small. Default: 0 [S2, unofficial].

### Algorithm Steps

1.  **Initialization and Validation:**
    *   Probe GPU list to determine `MaxMemoryInBytes`. If dedicated memory is 0, use dedicated system memory. If both are 0, set to 128MB. Ignore shared system memory [S2, unofficial].
    *   Validate inputs: Check for null renderer, null volume, and null inputs (Input 0 and Input 1 must exist) [S2, unofficial].
    *   Validate cropping planes: If `Cropping` is true, check that `Planes[0] < Planes[1]`, `Planes[2] < Planes[3]`, and `Planes[4] < Planes[5]`. If invalid, set `goodSoFar` to 0 silently [S2, unofficial].
    *   Update inputs if `LastInput` or `LastInput2` differ from current inputs or if `MTime` has changed [S2, unofficial].
    *   Transform inputs: Create shallow copies into `TransformedInput` and `TransformedInput2` if data changed [S2, unofficial].
    *   Extent Adjustment Workaround: For cloned inputs, adjust origin by adding `extents[2*cc]*spacing[cc]`. Adjust extents by subtracting `extents[2*cc]` from both min and max. Set new origin and extent. Repeat for input2 [S2, unofficial].

2.  **Transform Setup:**
    *   For each additional volume $N$, compute transformation matrix $P1toPN[N-1]$ combining:
        *   World-to-texture-N transform ($T_N$)
        *   Inverse world-to-texture-1 transform ($T_1^{-1}$)
        *   User-defined transform ($U_N$)
        *   $P1toPN[N-1] = T_N \times T_1^{-1} \times U_N$ [S1, unofficial].
    *   Store these in `TextureCoord_1toN[]` array [S1, unofficial].

3.  **Clipping/Cropping Initialization:**
    *   Initialize clipping planes (up to 6 cropping, 1 clipping) per volume [S1, unofficial].
    *   Convert planes from world coordinates to texture coordinates just before rendering [S1, unofficial].
    *   Compute `ClippedCroppingRegionPlanes` by clipping cropping planes by volume bounds [S1, unofficial].

4.  **Ray Casting Loop (Fragment Shader `trace()`):**
    *   **Initialization:**
        *   Calculate initial ray position `pos` for the first volume [S1, unofficial].
        *   Calculate initial positions `posX[]` for additional volumes: `posX[iii] = vec3(P1toPN[iii] * vec4(pos, 1.0))` [S1, unofficial].
        *   Initialize `rayDir` (incremental vector in texture space), `tMax` (max ray length), `destColor` (via `initialColor()`), and `remainOpacity = 1.0 - destColor.a` [S1, unofficial].
    *   **Traversal Loop:**
        *   While `inside` is true:
            *   `inside` is true if `t < tMax`, `remainOpacity >= 0.0039` (1/255), and `pos` is within `lowBounds[0]` and `highBounds[0]` [S1, unofficial].
            *   Update `pos` and `posX[]` using `rayDir` [S1, unofficial].
            *   Test `pos` and `posX[]` against `lowBounds[]` and `highBounds[]` for each volume [S1, unofficial].
            *   Test clipping flags. If ray position is not inside the clipped part, sample the volume texture [S1, unofficial].
            *   **Sampling and Classification:**
                *   For Volume 0: If inside bounds and not clipped, sample `value[0] = texture3D(dataSetTexture[0], pos)`. Compute `scalar[0] = scalarFromValue(value[0])`. Sample `opacity[0] = texture1D(opacityTexture[0], scalar[0])` [S1, unofficial].
                *   For Volumes 1 to $N-1$: If `posX[xx-1]` is inside bounds and not clipped, sample `value[xx] = texture3D(dataSetTexture[xx], posX[xx-1])`. Compute `scalar[xx] = scalarFromValue(value[xx])`. Sample `opacity[xx] = texture1D(opacityTexture[xx], scalar[xx])` [S1, unofficial].
            *   **Shading:**
                *   If shading is enabled, compute color contribution using `shade(vol, value[vol])` which uses gradient-based normals [S1, unofficial].
            *   **Compositing:**
                *   For each volume `vol` with `opacity[vol].a > 0.0`:
                    *   Compute `color[vol] = shade(vol, value[vol]) * opacity[vol].a` [S1, unofficial].
                    *   Update `destColor = destColor + color[vol] * remainOpacity` [S1, unofficial].
                    *   Update `remainOpacity = remainOpacity * (1.0 - opacity[vol].a)` [S1, unofficial].
            *   **Advance Ray:**
                *   `pos = pos + rayDir`
                *   `t += 1.0` [S1, unofficial].
    *   **Final Output:**
        *   `gl_FragColor = destColor`
        *   `gl_FragColor.a = 1.0 - remainOpacity` [S1, unofficial].

5.  **Masking (Optional):**
    *   If `MaskInput` is provided, apply mask blending based on `MaskType` and `MaskBlendFactor` [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Shader Code: vtkGPUVolumeRayCastMapper_CompositeFS

# Uniforms
uniform sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES];
uniform sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES];
uniform mat4 P1toPN[MAX_NUMBER_OF_VOLUMES-1];
uniform vec3 lowBounds[MAX_NUMBER_OF_VOLUMES];
uniform vec3 highBounds[MAX_NUMBER_OF_VOLUMES];
uniform int Number_Of_Volumes;
uniform vec3 ClippingplanesNormal[MAX_NUMBER_OF_VOLUMES];
uniform vec3 ClippingplanesOrigins[MAX_NUMBER_OF_VOLUMES];
uniform int clipped[MAX_NUMBER_OF_VOLUMES];

# Local Variables
vec3 pos;
vec3 posX[MAX_NUMBER_OF_VOLUMES-1];
vec3 rayDir;
float tMax;
vec4 destColor;
float remainOpacity;

# Initialization (called before trace loop)
def init_trace():
    pos = calculate_initial_pos_for_volume_0()
    for iii in range(Number_Of_Volumes - 1):
        posX[iii] = vec3(P1toPN[iii] * vec4(pos, 1.0))
    rayDir = calculate_ray_direction_in_texture_space()
    tMax = calculate_max_ray_length()
    destColor = initial_color()
    remainOpacity = 1.0 - destColor.a

# Helper: Check if point is inside bounds
def is_inside_bounds(point, bounds_low, bounds_high):
    return all(point[i] >= bounds_low[i] and point[i] <= bounds_high[i] for i in range(3))

# Helper: Check clipping
def is_clipped(point, plane_normal, plane_origin):
    # Returns True if point is outside the clipped region
    # Assuming plane defines a half-space; point is clipped if dot(point - origin, normal) < 0
    return dot(point - plane_origin, plane_normal) < 0.0

# Helper: Sample volume
def sample_volume(vol_index, point):
    value = texture3D(dataSetTexture[vol_index], point)
    scalar = scalar_from_value(value)
    opacity = texture1D(opacityTexture[vol_index], scalar)
    return opacity

# Helper: Shade
def shade(vol_index, value):
    # Uses P1toPN to compute correct increments for shading
    # Returns RGB color
    return compute_phong_shading(vol_index, value)

# Main Trace Loop
def trace():
    init_trace()
    
    while True:
        # Check termination conditions
        if t >= tMax:
            break
        if remainOpacity < 0.0039: # 1/255
            break
        if not is_inside_bounds(pos, lowBounds[0], highBounds[0]):
            break
            
        # Sample Volume 0
        opacity_0 = vec4(0.0)
        if is_inside_bounds(pos, lowBounds[0], highBounds[0]):
            if not (clipped[0] and is_clipped(pos, ClippingplanesNormal[0], ClippingplanesOrigins[0])):
                opacity_0 = sample_volume(0, pos)
        
        if opacity_0.a > 0.0:
            color_0 = shade(0, texture3D(dataSetTexture[0], pos)) * opacity_0.a
            destColor = destColor + color_0 * remainOpacity
            remainOpacity = remainOpacity * (1.0 - opacity_0.a)
            
        # Sample Additional Volumes
        for iii in range(Number_Of_Volumes - 1):
            vol_index = iii + 1
            opacity_vol = vec4(0.0)
            if is_inside_bounds(posX[iii], lowBounds[vol_index], highBounds[vol_index]):
                if not (clipped[vol_index] and is_clipped(posX[iii], ClippingplanesNormal[vol_index], ClippingplanesOrigins[vol_index])):
                    opacity_vol = sample_volume(vol_index, posX[iii])
            
            if opacity_vol.a > 0.0:
                color_vol = shade(vol_index, texture3D(dataSetTexture[vol_index], posX[iii])) * opacity_vol.a
                destColor = destColor + color_vol * remainOpacity
                remainOpacity = remainOpacity * (1.0 - opacity_vol.a)
                
        # Advance Ray
        pos = pos + rayDir
        t += 1.0
        
        # Update transformed positions for additional volumes
        for iii in range(Number_Of_Volumes - 1):
            posX[iii] = vec3(P1toPN[iii] * vec4(pos, 1.0))

    gl_FragColor = destColor
    gl_FragColor.a = 1.0 - remainOpacity
```

### Edge Cases and Pitfalls
*   **Hardware Loop Limits:** GPU hardware may have limits on loop iterations. The implementation uses a nested while loop structure to handle this [S1, unofficial].
*   **Non-Zero Extents:** The GPUVolumeRayCastMapper cannot handle extents starting from non-zero values. A workaround adjusts the origin and shifts extents to start from zero [S2, unofficial].
*   **Invalid Cropping:** If cropping planes define a region with zero or negative volume, the render fails silently (`goodSoFar=0`) without an error message [S2, unofficial].
*   **ReportProgress Slowdown:** `ReportProgress` uses `glFinish()`, which blocks the GPU and can significantly slow down rendering of small datasets; it should be disabled for small datasets [S2, unofficial].
*   **FinalColorWindow:** Cannot be null, but setting it to a negative value reverses the applied color values [S2, unofficial].
*   **Hardware Rendering Unsupported:** If input data has more than one independent component or if hardware lacks required extensions, hardware rendering is unsupported [S3, unofficial].

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
*   **Time Complexity:** $O(P \cdot N \cdot S)$, where $P$ is the number of pixels, $N$ is the number of volumes, and $S$ is the number of samples per ray. The complexity is linear with respect to the number of volumes due to the per-volume sampling and compositing in the shader loop [S1, unofficial].
*   **Memory Complexity:** $O(V \cdot R^3)$, where $V$ is the number of volumes and $R$ is the resolution of the volumes. Memory is dominated by the 3D textures for each volume and the associated 1D opacity textures [S1, S2, unofficial].
*   **Measured Performance:** Performance is highly dependent on GPU hardware and the number of volumes. Rendering multiple volumes increases the number of texture fetches and compositing operations per pixel, potentially reducing frame rates compared to single-volume rendering [S1, unofficial].
*   **Scaling:** Scales linearly with the number of volumes, but the constant factor increases due to the additional sampling and compositing steps [S1, unofficial].
*   **Strengths:** Allows simultaneous visualization of multiple co-registered volumes with correct spatial alignment and compositing. Leverages GPU parallelism for efficient ray casting [S1, unofficial].
*   **Limitations:** Limited by the maximum number of additional volumes supported by the shader and hardware (typically 3-4 additional volumes). Performance degrades with increasing number of volumes and resolution. Requires careful management of GPU memory [S1, S2, unofficial].
*   **When to Use:** When visualizing multiple co-registered volumetric datasets (e.g., multi-modal medical imaging) is required, and real-time or near-real-time interaction is needed [S1, unofficial].
*   **When Not to Use:** When the number of volumes exceeds hardware limits, or when the scene complexity is too high for the available GPU memory [S1, S2, unofficial].

## Background
The GPU Multi-Volume Ray Casting Mapper was developed as an extension to the Visualization Toolkit (VTK) to support multi-volume rendering on the GPU. It was modified by Carlos Falcón and Karl Krissian from CTIM [S1, unofficial]. The project builds on the original `vtkGPUVolumeRayCastMapper` which was designed for single volume rendering [S1, unofficial]. The implementation uses GLSL shaders instead of OpenCL for general GPU programming because shaders are faster and supported by more devices [S1, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
[S3] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-27 (unofficial)
