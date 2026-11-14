# GPU Multi-Volume Ray Casting Mapper

Also called: vtkGPUMultiVolumeRayCastMapper, GPU Multi Volume Ray Cast Mapper, Multi-volume ray casting

## Overview
The GPU Multi-Volume Ray Casting Mapper is a volume rendering technique that renders multiple volume datasets simultaneously by extending the single-volume `vtkGPUVolumeRayCastMapper` to accept multiple `vtkImageData` datasets and `vtkVolumeProperty` objects, using GPU shaders for ray traversal and compositing [S1, unofficial]. It sets itself apart from its siblings by explicitly compositing multiple 3D scalar fields along a single ray, using coordinate transformation matrices to align samples from different volumes to a common ray space [S1, unofficial]. Unlike the standard `vtkGPUVolumeRayCastMapper` which handles one volume, this mapper manages the synchronization of ray positions across $N$ volumes, applying per-volume transformations and clipping planes within the fragment shader loop [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two or more `vtkImageData` datasets (scalar fields) and corresponding `vtkVolumeProperty` objects (transfer functions). The primary input is at port 0; additional inputs are at ports $1$ to $N-1$.
*   **Outputs:** A 2D color buffer (framebuffer) with the composited image.

**Data Structures:**
*   **Texture Uniforms:**
    *   `sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES]`: 3D textures holding the scalar data for each volume.
    *   `sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES]`: 1D textures holding the opacity transfer function for each volume.
    *   `mat4 P1toPN[MAX_NUMBER_OF_VOLUMES-1]`: Transformation matrices. `P1toPN[i]` transforms texture coordinates from the first volume (Volume 1) to the $(i+2)$-th volume. Computed as $T_{world \to tex, N} \cdot T_{world \to tex, 1}^{-1} \cdot T_{user}$, where $T_{user}$ is an optional user-defined transform [S1, unofficial].
    *   `vec3 lowBounds[MAX_NUMBER_OF_VOLUMES]`, `vec3 highBounds[MAX_NUMBER_OF_VOLUMES]`: Bounding box limits for each volume in texture coordinates, adjusted for cropping.
    *   `vec3 ClippingplanesNormal[MAX_NUMBER_OF_VOLUMES]`, `vec3 ClippingplanesOrigins[MAX_NUMBER_OF_VOLUMES]`: Clipping plane definitions in texture coordinates.
    *   `int clipped[MAX_NUMBER_OF_VOLUMES]`: Flags indicating if clipping is active for each volume.
*   **State Variables:**
    *   `TransformedInput`, `TransformedInput2`: Shallow copies of input data used to track modification times.
    *   `ClippedCroppingRegionPlanes[6]`: Array storing cropping planes (min/max for X, Y, Z) clipped by volume bounds.

**Parameters:**
*   `MAX_NUMBER_OF_ADDITIONAL_VOLUMES`: Compile-time constant. 3 on Apple platforms, 4 otherwise [S1, unofficial].
*   `MAX_NUMBER_OF_VOLUMES`: 4 on Apple platforms, 5 otherwise [S1, unofficial].
*   `ImageSampleDistance`: Base sample distance in image space. Default 1.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `MinimumImageSampleDistance`: Lower bound for adaptive sampling. Default 1.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `MaximumImageSampleDistance`: Upper bound for adaptive sampling. Default 10.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `AutoAdjustSampleDistances`: Boolean flag (0/1). Enables automatic adjustment of `ImageSampleDistance` based on allocated render time [S2, unofficial].
*   `MaxMemoryInBytes`: Maximum GPU memory for volume data. Probed from GPU info; defaults to 128 MB if unavailable [S2, unofficial].
*   `MaxMemoryFraction`: Fraction of `MaxMemoryInBytes` used. Default 0.75. Range [0.1, 1.0] [S2, unofficial].
*   `ReportProgress`: Boolean flag. If true, calls `glFinish()` to report progress. Should be false for small datasets to avoid slowdown [S2, unofficial].
*   `MaskType`: Enum (`BinaryMaskType` or `LabelMapMaskType`). Default `LabelMapMaskType` [S2, unofficial].
*   `MaskBlendFactor`: Blending weight for mask color transfer function. Default 1.0. Range [0.0, 1.0] [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Probe GPU list for dedicated video memory. If 0, use dedicated system memory. If both 0, set `MaxMemoryInBytes` to 128 MB. Ignore shared system memory [S2, unofficial].
    *   Validate inputs: Check for null renderer, null volume, and null inputs. If `Cropping` is true, check that `CroppingRegionPlanes[0] < Planes[1]`, `Planes[2] < Planes[3]`, and `Planes[4] < Planes[5]`. If invalid, set `goodSoFar = 0` silently [S2, unofficial].
    *   Transform Setup: For each additional volume $N$, compute `P1toPN` matrix combining world-to-texture-N, inverse world-to-texture-1, and user transform [S1, unofficial].
    *   Extent Adjustment: If input extents do not start at zero, adjust the origin by adding `extents[2*cc]*spacing[cc]` and shift extents to start from zero for both inputs [S2, unofficial].

2.  **Render Entry:**
    *   If `GeneratingCanonicalView` is true, call `CanonicalViewRender` and return (skips timing/events) [S2, unofficial].
    *   Otherwise, invoke `VolumeMapperRenderStartEvent`, start timer, validate render, call `GPURender`, stop timer, and invoke `VolumeMapperRenderEndEvent` [S2, unofficial].

3.  **Fragment Shader Execution (`trace()` function):**
    *   **Initialization:**
        *   Calculate initial ray position `pos` for Volume 1.
        *   Calculate initial positions `posX[]` for additional volumes using `posX[iii] = vec3(P1toPN[iii] * vec4(pos, 1.0))` [S1, unofficial].
        *   Initialize `destColor` via `initialColor()`. Set `remainOpacity = 1.0 - destColor.a` [S1, unofficial].
        *   Initialize `rayDir` (incremental vector in texture space) and `tMax` (max ray length) [S1, unofficial].

    *   **Ray Marching Loop:**
        While `inside` is true:
        1.  `inside` is true if `t < tMax`, `remainOpacity >= 0.0039` (1/255), and `pos` is within `lowBounds[0]` and `highBounds[0]` [S1, unofficial].
        2.  **Volume 0 Sampling:**
            *   If `pos` is within bounds and not clipped (or clipping check passes):
                *   `value[0] = texture3D(dataSetTexture[0], pos)`
                *   `scalar[0] = scalarFromValue(value[0])`
                *   `opacity[0] = texture1D(opacityTexture[0], scalar[0])`
        3.  **Additional Volumes Sampling (for `iii` from 0 to `Number_Of_Volumes-2`):**
            *   Let `vol = iii + 1`.
            *   If `posX[vol-1]` is within bounds and not clipped:
                *   `value[vol] = texture3D(dataSetTexture[vol], posX[vol-1])`
                *   `scalar[vol] = scalarFromValue(value[vol])`
                *   `opacity[vol] = texture1D(opacityTexture[vol], scalar[vol])`
        4.  **Compositing:**
            *   For each volume `vol` sampled:
                *   If `opacity[vol].a > 0.0`:
                    *   `color[vol] = shade(vol, value[vol]) * opacity[vol].a`
                    *   `destColor = destColor + color[vol] * remainOpacity`
                    *   `remainOpacity = remainOpacity * (1.0 - opacity[vol].a)`
        5.  **Advance Ray:**
            *   `pos = pos + rayDir`
            *   `posX[iii] = posX[iii] + rayDir` (for all additional volumes)
            *   `t += 1.0`

    *   **Final Output:**
        *   `gl_FragColor = destColor`
        *   `gl_FragColor.a = 1.0 - remainOpacity` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader / Setup (Simplified)
# Computes rayDir, tMax, and passes P1toPN matrices to fragment shader

# Fragment Shader: trace()
uniform sampler3D dataSetTexture[MAX_NUMBER_OF_VOLUMES];
uniform sampler1D opacityTexture[MAX_NUMBER_OF_VOLUMES];
uniform mat4 P1toPN[MAX_NUMBER_OF_VOLUMES-1];
uniform vec3 lowBounds[MAX_NUMBER_OF_VOLUMES];
uniform vec3 highBounds[MAX_NUMBER_OF_VOLUMES];
uniform int Number_Of_Volumes;

void main() {
    // 1. Initialization
    vec3 pos = initial_pos_for_volume_0; // Computed from camera/ray generation
    vec3 posX[MAX_NUMBER_OF_VOLUMES-1];
    for (int i = 0; i < Number_Of_Volumes - 1; i++) {
        posX[i] = (P1toPN[i] * vec4(pos, 1.0)).xyz;
    }
    
    vec3 rayDir = compute_ray_direction(); // In texture space
    float tMax = compute_max_ray_length();
    vec4 destColor = initial_color();
    float remainOpacity = 1.0 - destColor.a;
    float t = 0.0;
    
    // 2. Ray Marching Loop
    // Note: Hardware loop limits may require nested structure or careful unrolling
    while (t < tMax && remainOpacity >= 0.0039) {
        bool inside = (pos.x >= lowBounds[0].x && pos.x <= highBounds[0].x &&
                       pos.y >= lowBounds[0].y && pos.y <= highBounds[0].y &&
                       pos.z >= lowBounds[0].z && pos.z <= highBounds[0].z);
        
        if (inside) {
            // Sample Volume 0
            float val0 = texture3D(dataSetTexture[0], pos).r;
            float scalar0 = scalar_from_value(val0);
            float opac0 = texture1D(opacityTexture[0], scalar0).a;
            
            if (opac0 > 0.0) {
                vec3 col0 = shade_volume(0, val0) * opac0;
                destColor += col0 * remainOpacity;
                remainOpacity *= (1.0 - opac0);
            }
            
            // Sample Additional Volumes
            for (int i = 0; i < Number_Of_Volumes - 1; i++) {
                int vol = i + 1;
                bool insideVol = (posX[i].x >= lowBounds[vol].x && posX[i].x <= highBounds[vol].x &&
                                  posX[i].y >= lowBounds[vol].y && posX[i].y <= highBounds[vol].y &&
                                  posX[i].z >= lowBounds[vol].z && posX[i].z <= highBounds[vol].z);
                
                if (insideVol) {
                    float val = texture3D(dataSetTexture[vol], posX[i]).r;
                    float scalar = scalar_from_value(val);
                    float opac = texture1D(opacityTexture[vol], scalar).a;
                    
                    if (opac > 0.0) {
                        vec3 col = shade_volume(vol, val) * opac;
                        destColor += col * remainOpacity;
                        remainOpacity *= (1.0 - opac);
                    }
                }
            }
        }
        
        // Advance
        pos += rayDir;
        for (int i = 0; i < Number_Of_Volumes - 1; i++) {
            posX[i] += rayDir;
        }
        t += 1.0;
    }
    
    gl_FragColor = destColor;
    gl_FragColor.a = 1.0 - remainOpacity;
}
```

## Subtopics

### [OpenGL GPU Multi Volume Ray Cast Mapper](2919-opengl-gpu-multi-volume-ray-cast-mapper.md)
The OpenGL-specific subclass handling hardware extension checks, OpenGL object lifecycle, and clipping/cropping plane setup. It manages fragment shader programs, framebuffer objects, and texture uploads for rendering multiple volumes simultaneously.

### [Label Map Masking for Volume Rendering](4788-label-map-masking-for-volume-rendering.md)
Confines volume rendering to specific regions using a binary or multi-label mask, supporting blending between standard and mask-specific color transfer functions. It maps mask values to distinct color transfer function components and blending weights.

### [GPU Volume Ray Casting (VTK Composite Blend)](6912-gpu-volume-ray-casting-vtk-composite-blend.md)
A hardware-accelerated direct volume rendering technique implemented in VTK via `vtkGPUVolumeRayCastMapper` that uses the **Composite** blend mode for standard alpha blending during ray casting. It distinguishes itself from the multi-volume mapper by focusing on single-volume composite blending.

### [vtkSlicerGPURayCastVolumeMapper](7450-vtkslicergpuraycastvolumemapper.md)
Renders a volume using ray casting on a 3D texture with specific parameters for distance-based color blending and ICPE (Intensity Contrast Preserving Enhancement) scaling.

### Mask Type Configuration
Methods to set the mask type for volume rendering, either Binary or LabelMap. To set binary mask: `this->MaskType = vtkGPUMultiVolumeRayCastMapper::BinaryMaskType`. To set label map mask: `this->MaskType = vtkGPUMultiVolumeRayCastMapper::LabelMapMaskType` [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of volumes. The ray marching loop iterates for each sample, and for each sample, it performs texture lookups and compositing for all $N$ volumes.
*   **Memory Complexity:** Proportional to the sum of the memory required for $N$ 3D textures and $N$ 1D opacity textures. Limited by `MaxMemoryInBytes` [S2, unofficial].
*   **Performance:** Uses GLSL shaders, which are faster and more widely supported than OpenCL for this purpose [S1, unofficial]. Adaptive sampling (`AutoAdjustSampleDistances`) can improve performance by reducing sample density when render time is constrained [S2, unofficial].
*   **Strengths:** Simultaneous rendering of multiple volumes with correct depth ordering and compositing. Hardware-accelerated sampling and shading.
*   **Limitations:** Hardware loop limits may require nested loop structures or careful unrolling [S1, unofficial]. Non-zero extents in input data require manual adjustment of origin and extents [S2, unofficial]. `ReportProgress` using `glFinish()` can significantly slow down rendering of small datasets if enabled [S2, unofficial].
*   **When to Use:** When visualizing multiple co-registered or spatially related volume datasets (e.g., multi-modal medical imaging) simultaneously.
*   **When Not to Use:** When rendering a single volume (use `vtkGPUVolumeRayCastMapper` for better optimization), or when input data has more than one independent component (unsupported) [S3, unofficial].

## Background
The GPU Multi-Volume Ray Casting Mapper was developed as an extension to the Visualization Toolkit (VTK) to support multi-volume rendering on the GPU. It was modified by Carlos Falcón and Karl Krissian to extend the single-volume `vtkGPUVolumeRayCastMapper` [S1, unofficial]. The implementation relies on GLSL shaders for ray traversal and compositing, chosen for their speed and device support compared to OpenCL [S1, unofficial]. It is used in VTK-based applications requiring simultaneous visualization of multiple scalar fields.

## Sources
[S1] VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
[S3] VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-27 (unofficial)
