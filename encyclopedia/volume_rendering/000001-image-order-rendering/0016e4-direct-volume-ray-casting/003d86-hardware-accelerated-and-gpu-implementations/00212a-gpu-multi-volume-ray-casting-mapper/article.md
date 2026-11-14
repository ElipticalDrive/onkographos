# GPU Multi-Volume Ray Casting Mapper

Also called: vtkGPUMultiVolumeRayCastMapper, GPU Multi Volume Ray Cast Mapper, Multi-volume ray casting

## Overview
The GPU Multi-Volume Ray Casting Mapper is a hardware-accelerated direct volume rendering technique that renders multiple 3D scalar fields simultaneously by compositing them along a single ray [S1, unofficial]. It extends the single-volume `vtkGPUVolumeRayCastMapper` to accept multiple `vtkImageData` datasets and `vtkVolumeProperty` objects, using GPU fragment shaders for ray traversal and front-to-back alpha blending [S1, unofficial]. It sets itself apart from its siblings by explicitly managing coordinate transformations between multiple volume textures and supporting up to four additional volumes (five total) on non-Apple platforms, whereas standard mappers handle only one [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Two or more `vtkImageData` datasets (scalar fields) and corresponding `vtkVolumeProperty` objects (transfer functions). The primary input is at port 0; additional inputs are at ports 1 to $N$.
- **Outputs:** A 2D color buffer (framebuffer) with the composited volume image.

**Data Structures:**
- **Textures:** Each volume is uploaded to the GPU as a `sampler3D` (`dataSetTexture[i]`) and a 1D opacity lookup table (`opacityTexture[i]`).
- **Uniforms:**
  - `P1toPN[i]`: A $4 \times 4$ matrix transforming texture coordinates from the first volume to the $i$-th additional volume. Computed as $T_{1 \to N} = T_{world \to N} \cdot T_{world \to 1}^{-1} \cdot T_{user}$ [S1, unofficial].
  - `lowBounds[i]`, `highBounds[i]`: Bounding box limits in texture coordinates for volume $i$, adjusted for cropping.
  - `ClippingplanesNormal[i]`, `ClippingplanesOrigins[i]`, `clipped[i]`: Clipping plane definitions for volume $i$.
- **State Variables:**
  - `TransformedInput`, `TransformedInput2`: Shallow copies of input data used to track modification times.
  - `LastInput`, `LastInput2`: Pointers to the previous input data to detect changes.
  - `ClippedCroppingRegionPlanes[6]`: Array storing cropping planes clipped by volume bounds [S1, unofficial].

**Parameters:**
- `MAX_NUMBER_OF_ADDITIONAL_VOLUMES`: 3 on Apple platforms, 4 otherwise [S1, unofficial].
- `ImageSampleDistance`: Base sample distance in image space. Default 1.0. Clamped to [0.1, 100.0] [S2, unofficial].
- `MinimumImageSampleDistance`: Lower bound for adaptive sampling. Default 1.0 [S2, unofficial].
- `MaximumImageSampleDistance`: Upper bound for adaptive sampling. Default 10.0 [S2, unofficial].
- `AutoAdjustSampleDistances`: Boolean flag (default 1) to vary `ImageSampleDistance` based on allocated render time [S2, unofficial].
- `MaxMemoryInBytes`: Maximum GPU memory for textures. Defaults to dedicated video memory, or 128 MB if unavailable [S2, unofficial].
- `MaxMemoryFraction`: Fraction of `MaxMemoryInBytes` used. Default 0.75 [S2, unofficial].
- `ReportProgress`: Boolean flag (default true) to call `glFinish()` for progress reporting. Should be false for small datasets to avoid slowdown [S2, unofficial].
- `MaskType`: Enum for masking (Binary or LabelMap). Default LabelMap [S2, unofficial].
- `MaskBlendFactor`: Blending weight for mask color transfer function. Default 1.0 [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    - Probe GPU info to set `MaxMemoryInBytes`. If dedicated memory is 0, use dedicated system memory. If both are 0, set to 128 MB [S2, unofficial].
    - Initialize `TransformedInput` and `TransformedInput2` as shallow copies of inputs.
    - Compute `P1toPN` matrices for each additional volume using world-to-texture transforms and user-defined transforms [S1, unofficial].
    - Compute `ClippedCroppingRegionPlanes` by intersecting user-defined cropping planes with volume bounds [S1, unofficial].

2.  **Validation:**
    - Check for null renderer, volume, or inputs.
    - Check cropping validity: if `Cropping` is true and any min plane $\ge$ max plane, fail silently (`goodSoFar = 0`) [S2, unofficial].
    - If inputs changed (checked via `MTime` or pointer comparison), update `TransformedInput`/`TransformedInput2` [S2, unofficial].

3.  **Ray Generation:**
    - For each pixel, compute ray origin (`pos`) and direction (`rayDir`) in the texture coordinate space of the first volume [S1, unofficial].
    - Compute initial positions for additional volumes: `posX[i] = P1toPN[i] * vec4(pos, 1.0)` [S1, unofficial].

4.  **Ray Traversal Loop (Fragment Shader):**
    - Initialize `remainOpacity = 1.0`, `destColor = initialColor()` [S1, unofficial].
    - Loop while `inside` is true:
      - `inside` is true if `t < tMax`, `remainOpacity >= 0.0039` (1/255), and `pos` is within `lowBounds[0]` and `highBounds[0]` [S1, unofficial].
      - For each volume $v$ from 0 to $N$:
        - Determine position: `pos` for $v=0$, `posX[v-1]` for $v>0$.
        - Check bounds: if position is within `lowBounds[v]` and `highBounds[v]`.
        - Check clipping: if position is not inside the clipped region (using `ClippingplanesNormal` and `ClippingplanesOrigins`) [S1, unofficial].
        - Sample:
          - `value = texture3D(dataSetTexture[v], position)`
          - `scalar = scalarFromValue(value)`
          - `opacity = texture1D(opacityTexture[v], scalar)`
        - Composite:
          - If `opacity.a > 0.0`:
            - `color = shade(v, value) * opacity.a`
            - `destColor = destColor + color * remainOpacity`
            - `remainOpacity = remainOpacity * (1.0 - opacity.a)` [S1, unofficial]
      - Advance ray: `pos = pos + rayDir`, `t += 1.0` [S1, unofficial].

5.  **Output:**
    - `gl_FragColor = destColor`
    - `gl_FragColor.a = 1.0 - remainOpacity` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader: Compute ray parameters
# Fragment Shader: vtkGPUVolumeRayCastMapper_CompositeFS

uniform sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES];
uniform sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES];
uniform mat4 P1toPN[MAX_NUMBER_OF_VOLUMES - 1];
uniform vec3 lowBounds[MAX_NUMBER_OF_VOLUMES];
uniform vec3 highBounds[MAX_NUMBER_OF_VOLUMES];
uniform vec3 ClippingplanesNormal[MAX_NUMBER_OF_VOLUMES];
uniform vec3 ClippingplanesOrigins[MAX_NUMBER_OF_VOLUMES];
uniform int clipped[MAX_NUMBER_OF_VOLUMES];
uniform int Number_Of_Volumes;

vec3 pos;       # Entry position in texture space of volume 0
vec3 rayDir;    # Incremental vector in texture space
float tMax;     # Max ray length
vec4 destColor; # Accumulated color
float remainOpacity;

# Initialization
pos = compute_entry_position();
rayDir = compute_ray_direction();
tMax = compute_max_length();
destColor = initial_color();
remainOpacity = 1.0 - destColor.a;

# Pre-compute initial positions for additional volumes
posX = vec3[MAX_NUMBER_OF_VOLUMES - 1]();
for i in range(Number_Of_Volumes - 1):
    posX[i] = (P1toPN[i] * vec4(pos, 1.0)).xyz;

# Ray Traversal Loop
# Note: Hardware loop limits may require nested loop structure
while t < tMax and remainOpacity >= 0.0039:
    inside = True
    for v in range(Number_Of_Volumes):
        # Determine current position
        if v == 0:
            current_pos = pos
        else:
            current_pos = posX[v - 1]
        
        # Check bounds
        if not (all(current_pos >= lowBounds[v]) and all(current_pos <= highBounds[v])):
            inside = False
            continue
            
        # Check clipping
        if clipped[v] and not is_inside_clipping(current_pos, ClippingplanesNormal[v], ClippingplanesOrigins[v]):
            continue
            
        # Sample volume
        value = texture3D(dataSetTexture[v], current_pos)
        scalar = scalar_from_value(value)
        opacity_sample = texture1D(opacityTexture[v], scalar)
        
        # Composite
        if opacity_sample.a > 0.0:
            color_sample = shade(v, value) * opacity_sample.a
            destColor = destColor + color_sample * remainOpacity
            remainOpacity = remainOpacity * (1.0 - opacity_sample.a)
    
    # Advance ray
    pos = pos + rayDir
    for i in range(Number_Of_Volumes - 1):
        posX[i] = posX[i] + rayDir # Simplified: assumes same rayDir in all texture spaces for simplicity, 
                                    # actual implementation may require transforming rayDir per volume
    t += 1.0

gl_FragColor = destColor
gl_FragColor.a = 1.0 - remainOpacity
```

## Subtopics

### [OpenGL GPU Multi Volume Ray Cast Mapper](2919-opengl-gpu-multi-volume-ray-cast-mapper.md)
An OpenGL-specific subclass that handles hardware extension checks, OpenGL object lifecycle management, and clipping/cropping plane setup for GPU-accelerated volume rendering. It serves as the concrete implementation responsible for drawing the volume ray cast image to the screen by managing fragment shader programs, framebuffer objects, and texture uploads required for rendering multiple volumes simultaneously. It builds on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) by providing the specific OpenGL API bindings and state management necessary for hardware-accelerated ray casting.

### [Label Map Masking for Volume Rendering](4788-label-map-masking-for-volume-rendering.md)
Confines volume rendering to specific regions using either a binary mask or a multi-label map, supporting blending between standard and mask-specific color transfer functions. It differs from sibling mappers by explicitly mapping mask values to distinct color transfer function components and blending weights, rather than acting solely as a visibility gate.

### [GPU Volume Ray Casting (VTK Composite Blend)](6912-gpu-volume-ray-casting-vtk-composite-blend.md)
A hardware-accelerated direct volume rendering technique implemented in VTK via `vtkGPUVolumeRayCastMapper` that performs ray casting on the GPU to visualize 3D scalar fields. It transforms raw volumetric data into a 3D environment by accumulating color and opacity along rays using composite blending. It sets itself apart from its siblings by explicitly utilizing the **Composite** blend mode for standard alpha blending during the ray casting process, distinguishing it from the OpenGL-specific multi-volume mapper [2919-opengl-gpu-multi-volume-ray-cast-mapper.md] and the Slicer-specific mapper [7450-vtkslicergpuraycastvolumemapper.md].

### [vtkSlicerGPURayCastVolumeMapper](7450-vtkslicergpuraycastvolumemapper.md)
Renders a volume using ray casting on a 3D texture. It sets itself apart by providing specific parameters for distance-based color blending and ICPE (Intensity Contrast Preserving Enhancement) scaling, building on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) which manages the general GPU-accelerated pipeline.

### Mask Type Configuration
Methods to set the mask type for volume rendering, either Binary or LabelMap [S2, unofficial]. To set binary mask: assign `MaskType = vtkGPUMultiVolumeRayCastMapper::BinaryMaskType`. To set label map mask: assign `MaskType = vtkGPUMultiVolumeRayCastMapper::LabelMapMaskType` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples per ray and the number of volumes. Adaptive sampling (`AutoAdjustSampleDistances`) can reduce samples for distant or simple regions, improving performance [S2, unofficial].
- **Memory Complexity:** Proportional to the size of the 3D textures for all volumes. Limited by `MaxMemoryInBytes` [S2, unofficial].
- **Performance:** Hardware-accelerated via fragment shaders, leveraging GPU texture sampling hardware for trilinear interpolation [S1, unofficial]. Performance is dependent on GPU memory bandwidth and shader execution units.
- **Strengths:** Supports simultaneous rendering of multiple volumes with different transfer functions; efficient use of GPU hardware; supports adaptive sampling and early ray termination [S1, unofficial].
- **Limitations:** Limited to a small number of additional volumes (up to 4 extra) due to shader uniform limits and loop constraints; non-zero extents in input data require workarounds (origin/extent adjustment) [S2, unofficial]; `ReportProgress` can cause significant slowdowns if enabled for small datasets [S2, unofficial].
- **When to Use:** When multiple volumes need to be visualized simultaneously with different properties, and hardware acceleration is available.
- **When Not to Use:** When more than the maximum supported number of volumes are needed, or when running on hardware without sufficient OpenGL support or memory.

## Background
The GPU Multi-Volume Ray Casting Mapper was developed as an extension to the Visualization Toolkit (VTK) to support multi-volume rendering on the GPU. It was modified by Carlos Falcón and Karl Krissian to extend the single-volume `vtkGPUVolumeRayCastMapper` [S1, unofficial]. The implementation uses GLSL shaders instead of OpenCL for better performance and broader device support [S1, unofficial]. It is part of the VTK volume rendering pipeline, building on the principles of direct volume ray casting and hardware-accelerated sampling [5616-hardware-accelerated-volume-rendering.md].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
[S3] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-07 (unofficial)
