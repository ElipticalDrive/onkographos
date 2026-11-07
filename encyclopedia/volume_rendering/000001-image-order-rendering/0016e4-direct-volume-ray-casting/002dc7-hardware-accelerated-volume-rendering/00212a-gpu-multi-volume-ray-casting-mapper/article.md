# GPU Multi-Volume Ray Casting Mapper

Also called: vtkGPUMultiVolumeRayCastMapper, GPU Multi Volume Ray Cast Mapper, Multi-volume ray casting

## Overview
The GPU Multi-Volume Ray Casting Mapper is a volume rendering technique that renders multiple volume datasets simultaneously by extending the single-volume `vtkGPUVolumeRayCastMapper` to accept multiple `vtkImageData` datasets and `vtkVolumeProperty` objects, using GPU shaders for ray traversal and compositing [S1, S2, unofficial]. It sets itself apart from its siblings by compositing multiple 3D scalar fields along a single ray using coordinate transformation matrices to align samples from different volumes to a common ray space [S1, unofficial]. Unlike the standard single-volume mapper, it supports up to 4 additional volumes (5 total) and handles mask-based blending for specific regions [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Two or more `vtkImageData` datasets (scalar fields) and corresponding `vtkVolumeProperty` objects (transfer functions). The primary input is at port 0; additional inputs are at ports 1 to $N$. A mask input (`vtkImageData`) is optional for label-map or binary masking [S1, S2, unofficial].
- **Outputs:** A 2D color buffer (framebuffer) with accumulated color and opacity per pixel [S1, S2, unofficial].

**Data Structures:**
- **Transforms:** `TextureCoord_1toN[]` is an array of `vtkTransform` objects. For each additional volume $N$, it stores the matrix $M_{1 \to N}$ that converts texture coordinates from the first volume to volume $N$. This matrix is computed as $M_{1 \to N} = T_{world \to tex_N} \cdot T_{world \to tex_1}^{-1} \cdot T_{user}$, where $T_{user}$ is an optional user-defined transform [S1, unofficial].
- **Uniforms (Shader):**
  - `sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES]`: 3D textures for scalar data.
  - `sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES]`: 1D textures for opacity transfer functions.
  - `mat4 P1toPN[MAX_NUMBER_OF_VOLUMES-1]`: Pre-computed transformation matrices $M_{1 \to N}$ passed to the shader.
  - `vec3 lowBounds[MAX_NUMBER_OF_VOLUMES]`, `vec3 highBounds[MAX_NUMBER_OF_VOLUMES]`: Bounding box limits for each volume in texture space.
  - `vec3 ClippingplanesNormal[MAX_NUMBER_OF_VOLUMES]`, `vec3 ClippingplanesOrigins[MAX_NUMBER_OF_VOLUMES]`: Clipping plane definitions in texture space.
  - `int clipped[MAX_NUMBER_OF_VOLUMES]`: Flags indicating if clipping is active for each volume.
  - `int Number_Of_Volumes`: Total number of volumes being rendered [S1, S3, unofficial].
- **Memory Management:** `MaxMemoryInBytes` limits GPU memory usage. It is probed from GPU info (dedicated video memory, then dedicated system memory); if 0, defaults to 128 MB [S2, unofficial]. `MaxMemoryFraction` (default 0.75) determines the fraction of this limit used for textures [S1, S2, unofficial].

**Numbered Steps:**

1. **Initialization:**
   - Probe GPU memory to set `MaxMemoryInBytes`. If unavailable, use 128 MB [S2, unofficial].
   - Initialize `TransformedInput` and `TransformedInput2` (shallow copies of inputs) if inputs change or MTime updates [S2, unofficial].
   - For inputs with non-zero extents, adjust origin and shift extents to start from zero to work around GPU texture coordinate limitations [S2, unofficial].

2. **Transform Setup:**
   - For each additional volume $N$ (from 1 to $MAX\_NUMBER\_OF\_ADDITIONAL\_VOLUMES$), compute $P1toPN[N-1]$ using the world-to-texture transforms and any user-defined transform [S1, unofficial].

3. **Clipping/Cropping Initialization:**
   - Convert clipping and cropping planes from world coordinates to texture coordinates for each volume.
   - Compute `ClippedCroppingRegionPlanes` by intersecting cropping planes with volume bounds. If cropping planes define a region with zero or negative volume, fail silently (`goodSoFar = 0`) [S1, S2, unofficial].

4. **Render Pipeline:**
   - If `GeneratingCanonicalView` is true, call `CanonicalViewRender` (skips timing/events). Otherwise, invoke `VolumeMapperRenderStartEvent`, start timer, call `GPURender`, stop timer, and invoke `VolumeMapperRenderEndEvent` [S2, unofficial].

5. **Shader Execution (Fragment Shader `trace()`):**
   - **Input:** `pos` (ray origin in volume 1 texture space), `rayDir` (ray direction in texture space), `tMax` (max ray length).
   - **Initialization:** Calculate initial `posX[]` for additional volumes: `posX[iii] = vec3(P1toPN[iii] * vec4(pos, 1.0))`. Initialize `destColor` (via `initialColor()`) and `remainOpacity = 1.0 - destColor.a` [S1, unofficial].
   - **Loop:** While `inside` is true:
     - `inside` is true if `t < tMax`, `remainOpacity >= 0.0039` (1/255), and `pos` is within `lowBounds[0]` and `highBounds[0]` [S1, unofficial].
     - For each additional volume `iii` (0 to `Number_Of_Volumes-2`), update `posX[iii] = posX[iii] + rayDir` (transformed appropriately if needed, though typically `rayDir` is in volume 1 space, so `posX` updates are derived from `pos` update) [S1, unofficial]. *Note: Sources indicate `pos` and `posX[]` are updated using `rayDir` and transforms.*
     - **Volume 0:** If `pos` is within bounds and not clipped, sample `value[0] = texture3D(dataSetTexture[0], pos)`. Compute `scalar[0] = scalarFromValue(value[0])`. Sample `opacity[0] = texture1D(opacityTexture[0], scalar[0])`.
     - **Volumes 1 to N-1:** If `posX[xx-1]` is within bounds and not clipped, sample `value[xx] = texture3D(dataSetTexture[xx], posX[xx-1])`. Compute `scalar[xx] = scalarFromValue(value[xx])`. Sample `opacity[xx] = texture1D(opacityTexture[xx], scalar[xx])` [S1, unofficial].
     - **Compositing:** For each volume `vol` with `opacity[vol].a > 0.0`:
       - Compute `color[vol] = shade(vol, value[vol]) * opacity[vol].a` [S1, unofficial].
       - Update `destColor = destColor + color[vol] * remainOpacity`.
       - Update `remainOpacity = remainOpacity * (1.0 - opacity[vol].a)` [S1, unofficial].
     - **Advance:** `pos = pos + rayDir`, `t += 1.0` [S1, unofficial].
   - **Output:** `gl_FragColor = destColor`, `gl_FragColor.a = 1.0 - remainOpacity` [S1, unofficial].

6. **Masking (Optional):**
   - If a mask input is provided, blend between standard and mask-specific color transfer functions based on `MaskBlendFactor` and `MaskType` (Binary or LabelMap) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Shader: vtkGPUVolumeRayCastMapper_CompositeFS

uniform sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES];
uniform sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES];
uniform mat4 P1toPN[MAX_NUMBER_OF_VOLUMES-1];
uniform vec3 lowBounds[MAX_NUMBER_OF_VOLUMES];
uniform vec3 highBounds[MAX_NUMBER_OF_VOLUMES];
uniform int Number_Of_Volumes;
uniform int clipped[MAX_NUMBER_OF_VOLUMES];
uniform vec3 ClippingplanesNormal[MAX_NUMBER_OF_VOLUMES];
uniform vec3 ClippingplanesOrigins[MAX_NUMBER_OF_VOLUMES];

in vec3 pos; // Ray origin in volume 1 texture space
in vec3 rayDir; // Ray direction in volume 1 texture space
in float tMax;

out vec4 gl_FragColor;

void main() {
    // Initialize
    vec3 posX[MAX_NUMBER_OF_VOLUMES-1];
    for (int iii = 0; iii < Number_Of_Volumes - 1; iii++) {
        posX[iii] = (P1toPN[iii] * vec4(pos, 1.0)).xyz;
    }
    
    vec4 destColor = initialColor();
    float remainOpacity = 1.0 - destColor.a;
    float t = 0.0;
    
    // Check initial bounds for volume 0
    bool inside = (t < tMax) && (remainOpacity >= 0.0039) && 
                  (all(pos >= lowBounds[0]) && all(pos <= highBounds[0]));
    
    while (inside) {
        // Volume 0 sampling
        if (all(pos >= lowBounds[0]) && all(pos <= highBounds[0]) && 
            (!clipped[0] || isInsideClip(pos, ClippingplanesNormal[0], ClippingplanesOrigins[0]))) {
            float value0 = texture3D(dataSetTexture[0], pos).r;
            float scalar0 = scalarFromValue(value0);
            float opacity0 = texture1D(opacityTexture[0], scalar0).a;
            
            if (opacity0 > 0.0) {
                vec4 color0 = shade(0, value0) * opacity0;
                destColor = destColor + color0 * remainOpacity;
                remainOpacity = remainOpacity * (1.0 - opacity0);
            }
        }
        
        // Additional volumes sampling
        for (int xx = 1; xx < Number_Of_Volumes; xx++) {
            vec3 currentPos = posX[xx-1];
            if (all(currentPos >= lowBounds[xx]) && all(currentPos <= highBounds[xx]) && 
                (!clipped[xx] || isInsideClip(currentPos, ClippingplanesNormal[xx], ClippingplanesOrigins[xx]))) {
                float value = texture3D(dataSetTexture[xx], currentPos).r;
                float scalar = scalarFromValue(value);
                float opacity = texture1D(opacityTexture[xx], scalar).a;
                
                if (opacity > 0.0) {
                    vec4 color = shade(xx, value) * opacity;
                    destColor = destColor + color * remainOpacity;
                    remainOpacity = remainOpacity * (1.0 - opacity);
                }
            }
        }
        
        // Advance ray
        t += 1.0;
        pos = pos + rayDir;
        for (int iii = 0; iii < Number_Of_Volumes - 1; iii++) {
            posX[iii] = posX[iii] + rayDir; // Simplified; actual transform may vary
        }
        
        // Check bounds for next step
        inside = (t < tMax) && (remainOpacity >= 0.0039) && 
                 (all(pos >= lowBounds[0]) && all(pos <= highBounds[0]));
    }
    
    gl_FragColor = destColor;
    gl_FragColor.a = 1.0 - remainOpacity;
}
```

**Parameters:**
- `MAX_NUMBER_OF_ADDITIONAL_VOLUMES`: 3 on Apple, 4 otherwise [S1, unofficial].
- `AutoAdjustSampleDistances`: Boolean (default 1). Enables adaptive sampling [S2, unofficial].
- `ImageSampleDistance`: Base sample distance in XY (default 1.0). Clamped [0.1, 100.0] [S2, unofficial].
- `MinimumImageSampleDistance`: Lower bound for adaptive sampling (default 1.0) [S2, unofficial].
- `MaximumImageSampleDistance`: Upper bound for adaptive sampling (default 10.0) [S2, unofficial].
- `SampleDistance`: Distance between samples along ray (default 1.0) [S2, unofficial].
- `MaxMemoryInBytes`: Max GPU memory for textures. Defaults to 128 MB if not probed [S2, unofficial].
- `MaxMemoryFraction`: Fraction of `MaxMemoryInBytes` used (default 0.75) [S1, S2, unofficial].
- `ReportProgress`: Boolean (default true). Uses `glFinish()` to report progress. Disable for small datasets to avoid slowdown [S1, S2, unofficial].
- `MaskBlendFactor`: Blending weight for mask CTF (default 1.0) [S2, unofficial].
- `MaskType`: Enum (Binary or LabelMap, default LabelMap) [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Hardware Loop Limits:** Requires nested while loop structure due to GPU shader loop constraints [S1, unofficial].
- **Non-Zero Extents:** GPU textures require extents starting from zero. Workaround adjusts origin and shifts extents [S2, unofficial].
- **Invalid Cropping:** If cropping planes define zero/negative volume, render fails silently [S2, unofficial].
- **ReportProgress:** `glFinish()` blocks GPU, slowing small datasets. Should be disabled [S2, unofficial].
- **FinalColorWindow:** Negative values reverse color application [S2, unofficial].

## Subtopics

### [OpenGL GPU Multi Volume Ray Cast Mapper](2919-opengl-gpu-multi-volume-ray-cast-mapper.md)
The OpenGL GPU Multi Volume Ray Cast Mapper is an OpenGL-specific subclass of the GPU Multi-Volume Ray Casting Mapper that handles hardware extension checks, OpenGL object lifecycle management, and clipping/cropping plane setup for GPU-accelerated volume rendering. It serves as the concrete implementation responsible for drawing the volume ray cast image to the screen by managing fragment shader programs, framebuffer objects, and texture uploads required for rendering multiple volumes simultaneously. It builds on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) by providing the specific OpenGL API bindings and state management necessary for hardware-accelerated ray casting.

### [Label Map Masking for Volume Rendering](4788-label-map-masking-for-volume-rendering.md)
Label Map Masking is a technique that confines volume rendering to specific regions using either a binary mask or a multi-label map, supporting blending between standard and mask-specific color transfer functions. It sets itself apart from sibling mappers by explicitly defining how mask values (binary or labeled) dictate the selection of color transfer functions (CTFs) and their blending weights, rather than simply acting as a visibility gate.

### [GPU Volume Ray Casting (VTK Composite Blend)](6912-gpu-volume-ray-casting-vtk-composite-blend.md)
GPU Volume Ray Casting (VTK Composite Blend) is a hardware-accelerated direct volume rendering technique implemented in VTK via `vtkGPUVolumeRayCastMapper` that performs ray casting on the GPU to visualize 3D scalar fields. It transforms raw volumetric data into a 3D environment by accumulating color and opacity along rays using composite blending. It sets itself apart from its siblings by explicitly utilizing the **Composite** blend mode for standard alpha blending during the ray casting process, distinguishing it from the OpenGL-specific multi-volume mapper [2919-opengl-gpu-multi-volume-ray-cast-mapper.md] and the Slicer-specific mapper [7450-vtkslicergpuraycastvolumemapper.md].

### [vtkSlicerGPURayCastVolumeMapper](7450-vtkslicergpuraycastvolumemapper.md)
Renders a volume using ray casting on a 3D texture. It builds on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) to provide a specific implementation for the 3D Slicer environment, supporting adaptive sampling, masking, and AMR modes via the parent's pipeline hooks. It is distinguished from its siblings by its specific integration with CTK transfer functions and its handling of specific OpenGL uniform layouts for ray casting parameters.

### Mask Type Configuration
Methods to set the mask type for volume rendering, either Binary or LabelMap [S2, unofficial]. To set binary mask: assign `this->MaskType = vtkGPUMultiVolumeRayCastMapper::BinaryMaskType`. To set label map mask: assign `this->MaskType = vtkGPUMultiVolumeRayCastMapper::LabelMapMaskType` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples per ray and the number of volumes. Adaptive sampling (`AutoAdjustSampleDistances`) can reduce samples based on allocated render time [S2, unofficial].
- **Memory Complexity:** Proportional to the total size of the 3D textures for all volumes, limited by `MaxMemoryInBytes` [S1, S2, unofficial].
- **Measured Performance:** Performance depends on GPU hardware, resolution, and scene complexity. `ReportProgress` can significantly slow down rendering of small datasets due to `glFinish()` blocking [S2, unofficial].
- **Scaling:** Supports up to 5 volumes (1 primary + 4 additional). Scaling to more volumes is limited by `MAX_NUMBER_OF_ADDITIONAL_VOLUMES` [S1, unofficial].
- **Strengths:** Simultaneous rendering of multiple volumes, hardware acceleration via shaders, adaptive sampling, masking support [S1, S2, unofficial].
- **Limitations:** Limited to 5 volumes, requires GPU with shader support, non-zero extents require workaround, silent failure on invalid cropping [S1, S2, unofficial].
- **When to Use:** When multiple volumes need to be rendered simultaneously with accurate compositing and shading [S1, unofficial].
- **When Not to Use:** When more than 5 volumes are needed, or when GPU hardware lacks required extensions [S3, unofficial].

## Background
The GPU Multi-Volume Ray Casting Mapper was developed by Carlos Falcón and Karl Krissian as an extension to the Visualization Toolkit (VTK) to enable multi-volume rendering on the GPU [S1, unofficial]. It builds on the single-volume `vtkGPUVolumeRayCastMapper` and uses GLSL shaders for ray traversal and compositing, chosen for their speed and broad device support compared to OpenCL [S1, unofficial]. The project includes a demo for testing purposes and modifies several VTK classes to handle multiple inputs and transforms [S1, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
[S3] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-27 (unofficial)
