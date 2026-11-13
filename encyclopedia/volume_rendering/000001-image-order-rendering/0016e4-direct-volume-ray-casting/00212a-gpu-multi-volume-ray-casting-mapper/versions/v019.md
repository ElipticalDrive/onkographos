# GPU Multi-Volume Ray Casting Mapper

Also called: vtkGPUMultiVolumeRayCastMapper, GPU Multi Volume Ray Cast Mapper, Multi-volume ray casting

## Overview
The GPU Multi-Volume Ray Casting Mapper is a hardware-accelerated direct volume rendering technique that renders multiple 3D scalar fields simultaneously by compositing them along a single ray [S1, S2, S3, unofficial]. It extends the single-volume `vtkGPUVolumeRayCastMapper` to accept multiple `vtkImageData` datasets and `vtkVolumeProperty` objects, using GPU shaders for ray traversal and compositing [S1, unofficial]. It sets itself apart from its siblings by explicitly managing coordinate transformations to align samples from different volumes to a common ray space and supporting features like masking, canonical view generation, and adaptive sampling [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two or more `vtkImageData` datasets (scalar volumes) and corresponding `vtkVolumeProperty` objects (transfer functions). The primary input is at port 0; additional inputs are at ports 1 to $N-1$. A mask input (`vtkImageData`) is optional for label map or binary masking [S1, S2, unofficial].
*   **Outputs:** A 2D color buffer (fragment shader output `gl_FragColor`) containing the compositied color and opacity for each pixel [S1, unofficial].

**Data Structures:**
*   **Volume Data:** Stored in GPU-accessible 3D textures (`sampler3D dataSetTexture[N]`) [S1, S2, unofficial].
*   **Opacity Textures:** 1D textures (`sampler1D opacityTexture[N]`) mapping scalar values to opacity [S1, unofficial].
*   **Transformations:** An array `P1toPN[N-1]` of $4 \times 4$ matrices. `P1toPN[i]` transforms texture coordinates from the first volume (Volume 1) to the $(i+1)$-th additional volume [S1, unofficial].
    *   Computed as: $T_{1 \to N} = T_{world \to N} \cdot T_{world \to 1}^{-1} \cdot T_{user}$, where $T_{user}$ is an optional user-defined transform [S1, unofficial].
*   **Bounding Boxes:** `lowBounds[N]` and `highBounds[N]` (vec3) storing the clipped bounding box for each volume [S1, unofficial].
*   **Clipping Planes:** `ClippingplanesNormal[N]`, `ClippingplanesOrigins[N]`, and `clipped[N]` (int flags) passed as uniforms [S1, unofficial].
*   **Masking:** `MaskInput` (optional), `MaskType` (Binary or LabelMap), and `MaskBlendFactor` [S2, unofficial].

**Parameters:**
*   `MAX_NUMBER_OF_ADDITIONAL_VOLUMES`: Compile-time constant. 3 on Apple platforms, 4 otherwise [S1, unofficial].
*   `ImageSampleDistance`: Base sample distance in image space. Default 1.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `MinimumImageSampleDistance`: Lower bound for adaptive sampling. Default 1.0 [S2, unofficial].
*   `MaximumImageSampleDistance`: Upper bound for adaptive sampling. Default 10.0 [S2, unofficial].
*   `AutoAdjustSampleDistances`: Boolean (0/1). Enables automatic adjustment of `ImageSampleDistance` based on render time [S1, S2, unofficial].
*   `MaxMemoryInBytes`: Maximum GPU memory for volume data. Defaults to dedicated video memory, or 128 MB if unavailable [S1, S2, unofficial].
*   `MaxMemoryFraction`: Fraction of `MaxMemoryInBytes` used. Default 0.75. Range [0.1, 1.0] [S1, S2, unofficial].
*   `ReportProgress`: Boolean. If true, calls `glFinish()` to report progress. Should be false for small datasets to avoid slowdown [S1, S2, unofficial].
*   `FinalColorWindow` / `FinalColorLevel`: Scaling parameters for final color. Default 1.0 / 0.5 [S2, unofficial].
*   `ClippedCroppingRegionPlanes[6]`: Array of doubles for X, Y, Z min/max clipping planes. Defaults: Min=VTK_DOUBLE_MAX, Max=VTK_DOUBLE_MIN [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Probe GPU for dedicated memory. If 0, use dedicated system memory. If both 0, set `MaxMemoryInBytes` to 128 MB [S2, unofficial].
    *   Validate inputs: Check for null renderer, volume, or input data. Check cropping planes validity (min < max) [S2, unofficial].
    *   If `GeneratingCanonicalView` is true, call `CanonicalViewRender` and return [S2, unofficial].
    *   Invoke `VolumeMapperRenderStartEvent` and start timer [S2, unofficial].

2.  **Data Preparation:**
    *   If input data has changed (`LastInput` or `TransformedInput` MTime), create shallow copies into `TransformedInput` [S2, unofficial].
    *   **Extent Adjustment Workaround:** GPU ray casting requires extents starting from zero. Adjust origin by adding `extents[2*cc]*spacing[cc]` and shift extents by subtracting `extents[2*cc]` from both min and max for each axis `cc` [S2, unofficial].
    *   Compute `P1toPN` matrices for each additional volume [S1, unofficial].
    *   Compute `ClippedCroppingRegionPlanes` by intersecting user-defined cropping planes with volume bounds [S1, unofficial].
    *   Upload textures and uniforms to GPU [S1, S2, unofficial].

3.  **Ray Casting Loop (Fragment Shader `trace()`):**
    *   Initialize `pos` (entry position in Volume 1 texture space), `rayDir` (incremental vector), `tMax` (max ray length), `destColor` (initial color), and `remainOpacity = 1.0 - destColor.a` [S1, unofficial].
    *   Calculate initial `posX[i]` for each additional volume $i$:
        $$ posX[i] = P1toPN[i] \cdot vec4(pos, 1.0) $$
    *   **While** `inside` is true:
        *   `inside` is true if `t < tMax`, `remainOpacity >= 0.0039` (1/255), and `pos` is within `lowBounds[0]` and `highBounds[0]` [S1, unofficial].
        *   For each additional volume $i$ from 0 to $N-2$:
            *   Update `posX[i]` using `rayDir` and `P1toPN[i]` [S1, unofficial].
        *   For Volume 0:
            *   If `pos` is within bounds and not clipped:
                *   Sample value: $v_0 = \text{texture3D}(dataSetTexture[0], pos)$
                *   Compute scalar: $s_0 = \text{scalarFromValue}(v_0)$
                *   Sample opacity: $\alpha_0 = \text{texture1D}(opacityTexture[0], s_0).a$
                *   Compute color: $C_0 = \text{shade}(0, v_0) \cdot \alpha_0$
                *   Composite:
                    $$ destColor = destColor + C_0 \cdot remainOpacity $$
                    $$ remainOpacity = remainOpacity \cdot (1.0 - \alpha_0) $$
        *   For each additional volume $j$ (1 to $N-1$):
            *   If `posX[j-1]` is within bounds and not clipped:
                *   Sample value: $v_j = \text{texture3D}(dataSetTexture[j], posX[j-1])$
                *   Compute scalar: $s_j = \text{scalarFromValue}(v_j)$
                *   Sample opacity: $\alpha_j = \text{texture1D}(opacityTexture[j], s_j).a$
                *   Compute color: $C_j = \text{shade}(j, v_j) \cdot \alpha_j$
                *   Composite:
                    $$ destColor = destColor + C_j \cdot remainOpacity $$
                    $$ remainOpacity = remainOpacity \cdot (1.0 - \alpha_j) $$
        *   Advance ray:
            $$ pos = pos + rayDir $$
            $$ t = t + 1.0 $$
    *   Final Output:
        $$ gl\_FragColor = destColor $$
        $$ gl\_FragColor.a = 1.0 - remainOpacity $$
    *   [G] The `shade()` function typically computes lighting (e.g., Phong) using gradients derived from the volume data.

4.  **Post-Render:**
    *   Stop timer, record time [S2, unofficial].
    *   Invoke `VolumeMapperRenderEndEvent` [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Hardware Loop Limits:** GPUs have limits on loop iterations. A nested while loop structure is required instead of a single loop over all samples and volumes [S1, unofficial].
*   **Non-Zero Extents:** VTK's GPU mapper cannot handle extents starting from non-zero values. The extent adjustment workaround is mandatory [S2, unofficial].
*   **Invalid Cropping:** If cropping planes define a region with zero or negative volume, rendering fails silently (`goodSoFar=0`) [S2, unofficial].
*   **ReportProgress:** Using `glFinish()` blocks the GPU and slows down small datasets. Disable for small volumes [S2, unofficial].
*   **FinalColorWindow:** Setting to a negative value reverses color application [S2, unofficial].

## Subtopics

### [OpenGL GPU Multi Volume Ray Cast Mapper](2919-opengl-gpu-multi-volume-ray-cast-mapper.md)
The OpenGL GPU Multi Volume Ray Cast Mapper is an OpenGL-specific subclass that handles hardware extension checks, OpenGL object lifecycle management, and clipping/cropping plane setup [2919-opengl-gpu-multi-volume-ray-cast-mapper.md]. It manages fragment shader programs, framebuffer objects, and texture uploads for rendering multiple volumes simultaneously. Choose this when using the VTK OpenGL rendering backend.

### [Label Map Masking for Volume Rendering](4788-label-map-masking-for-volume-rendering.md)
Label Map Masking confines volume rendering to specific regions using a binary or multi-label map, supporting blending between standard and mask-specific color transfer functions [4788-label-map-masking-for-volume-rendering.md]. It sets itself apart by explicitly defining how mask values dictate CTF selection and blending weights, rather than just acting as a visibility gate. Choose this when region-specific visualization with distinct transfer functions is required.

### [GPU Volume Ray Casting (VTK Composite Blend)](6912-gpu-volume-ray-casting-vtk-composite-blend.md)
GPU Volume Ray Casting (VTK Composite Blend) is a hardware-accelerated direct volume rendering technique implemented in VTK via `vtkGPUVolumeRayCastMapper` that performs ray casting on the GPU to visualize 3D scalar fields [6912-gpu-volume-ray-casting-vtk-composite-blend.md]. It sets itself apart by explicitly utilizing the **Composite** blend mode for standard alpha blending during the ray casting process. Choose this for standard single-volume rendering with composite blending.

### [vtkSlicerGPURayCastVolumeMapper](7450-vtkslicergpuraycastvolumemapper.md)
Renders a volume using ray casting on a 3D texture [7450-vtkslicergpuraycastvolumemapper.md]. It builds on the GPU Multi-Volume Ray Casting Mapper to provide a specific implementation for the 3D Slicer environment, supporting adaptive sampling, masking, and AMR modes. It is distinguished by its integration with CTK transfer functions and specific OpenGL uniform layouts. Choose this when using the 3D Slicer application.

## Operating characteristics
*   **Time Complexity:** $O(P \cdot S \cdot N)$, where $P$ is the number of pixels, $S$ is the number of samples per ray, and $N$ is the number of volumes [S1, S2, unofficial]. Adaptive sampling reduces $S$ dynamically [S2, unofficial].
*   **Memory Complexity:** $O(V \cdot D^3)$, where $V$ is the number of volumes and $D$ is the volume dimension, stored in 3D textures [S1, S2, unofficial]. Limited by `MaxMemoryInBytes` [S2, unofficial].
*   **Measured Performance:** Performance scales with GPU memory bandwidth and shader execution units. Adaptive sampling improves frame rates for large datasets by reducing sample count when allocated time is exceeded [S2, unofficial].
*   **Strengths:** Hardware acceleration via GLSL shaders; supports multiple volumes simultaneously; adaptive sampling for performance; masking support [S1, S2, unofficial].
*   **Limitations:** Limited by hardware loop iterations; requires non-zero extent adjustment; silent failure on invalid cropping; `ReportProgress` can cause slowdowns [S1, S2, unofficial].
*   **When to Use:** Real-time visualization of multiple co-registered volumes on GPU-supported hardware [S1, unofficial].
*   **When Not to Use:** When volumes have non-zero extents that cannot be adjusted; when hardware lacks required OpenGL extensions; for very small datasets where CPU overhead dominates [S2, unofficial].

## Background
The GPU Multi-Volume Ray Casting Mapper was developed as an extension to the Visualization Toolkit (VTK) to support simultaneous rendering of multiple volume datasets [S1, unofficial]. It was modified by Carlos Falcón and Karl Krissian to add multi-volume support, including transformation matrices and shader modifications for compositing [S1, unofficial]. It builds on the single-volume `vtkGPUVolumeRayCastMapper` and uses GLSL shaders for GPU programming, chosen for their speed and broad device support over OpenCL [S1, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
[S3] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-27 (unofficial)
