# GPU Multi-Volume Ray Casting Mapper

Also called: vtkGPUMultiVolumeRayCastMapper, GPU Multi Volume Ray Cast Mapper, Multi-volume ray casting

## Overview
The GPU Multi-Volume Ray Casting Mapper is a volume rendering technique that renders multiple volume datasets simultaneously by extending the single-volume `vtkGPUVolumeRayCastMapper` to accept multiple `vtkImageData` datasets and `vtkVolumeProperty` objects, using GPU shaders for ray traversal and compositing [S1, S2, unofficial]. It sets itself apart from its siblings by explicitly compositing multiple 3D scalar fields along a single ray using coordinate transformation matrices to align samples from different volumes to a common ray space [S1, unofficial]. Unlike surface-based approaches or single-volume renderers, it renders the entire volume data directly without intermediate mesh extraction [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two or more `vtkImageData` datasets (scalar fields) and corresponding `vtkVolumeProperty` objects (transfer functions). The primary input is at port 0; additional inputs are at ports 1 to $N$. A mask input (`vtkImageData`) is optional.
*   **Outputs:** A 2D framebuffer (color and opacity) representing the composite view of all volumes.

**Data Structures:**
*   **Transforms:** `AdditionalInputUserTransform[N]` stores user-defined transformations for additional volumes. `P1toPN[N-1]` stores pre-computed matrices converting texture coordinates from the first volume to the $N$-th volume.
*   **Bounds:** `lowBounds[N]` and `highBounds[N]` store the bounding box (clipped by cropping) for each volume in texture coordinates.
*   **Textures:** `dataSetTexture[N]` (sampler3D) for scalar data; `opacityTexture[N]` (sampler1D) for opacity transfer functions.
*   **Clipping:** `ClippingplanesNormal[N]`, `ClippingplanesOrigins[N]`, `clipped[N]` (boolean flags) for per-volume clipping planes.
*   **Masking:** `MaskInput` (vtkImageData*), `MaskBlendFactor` (float), `MaskType` (enum).

**Parameters:**
*   `MAX_NUMBER_OF_ADDITIONAL_VOLUMES`: 3 on Apple platforms, 4 otherwise [S1, S2, unofficial].
*   `ImageSampleDistance`: Base sample distance in image space. Default 1.0. Clamped [0.1, 100.0] [S2, unofficial].
*   `MinimumImageSampleDistance`: Lower bound for adaptive sampling. Default 1.0 [S2, unofficial].
*   `MaximumImageSampleDistance`: Upper bound for adaptive sampling. Default 10.0 [S2, unofficial].
*   `AutoAdjustSampleDistances`: Boolean. Enables automatic adjustment of `ImageSampleDistance` based on render time [S2, unofficial].
*   `MaxMemoryInBytes`: Maximum GPU memory for textures. Defaults to dedicated video memory, or 128MB if unavailable [S2, unofficial].
*   `MaxMemoryFraction`: Fraction of `MaxMemoryInBytes` used. Default 0.75 [S1, S2, unofficial].
*   `ReportProgress`: Boolean. If true, calls `glFinish()` to report progress. Should be false for small datasets to avoid slowdown [S2, unofficial].
*   `MaskBlendFactor`: Blending weight for mask CT vs standard CT. Range [0.0, 1.0]. Default 1.0 [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Probe GPU for dedicated video memory. If 0, use dedicated system memory. If both 0, set `MaxMemoryInBytes` to 128MB [S2, unofficial].
    *   Validate inputs: Check for null renderer, volume, or inputs. Check that cropping planes are valid (min < max for all axes); if not, fail silently [S1, S2, unofficial].
    *   Compute `P1toPN` matrices: For each additional volume $N$, compute $T_{1 \to N} = T_{world \to tex_N} \cdot T_{world \to tex_1}^{-1} \cdot T_{user}$ [S1, unofficial].

2.  **Data Preparation:**
    *   Check if input data has changed (MTime). If so, create shallow copies into `TransformedInput` and `TransformedInput2` [S2, unofficial].
    *   **Extent Adjustment Workaround:** GPU ray casting requires extents starting from zero. Adjust origin by adding $extent_{2i} \cdot spacing_i$. Adjust extents by subtracting $extent_{2i}$ from min and max [S2, unofficial].
    *   Compute `ClippedCroppingRegionPlanes` by intersecting user-defined cropping planes with volume bounds [S1, S2, unofficial].

3.  **Shader Setup:**
    *   Upload `dataSetTexture`, `opacityTexture`, `P1toPN`, `lowBounds`, `highBounds`, clipping planes, and mask data to GPU uniforms [S1, S2, unofficial].

4.  **Ray Casting Loop (Fragment Shader `trace()`):**
    *   Initialize `pos` (entry position in volume 1 texture space), `rayDir` (incremental vector), `tMax` (max ray length), `destColor` (initial color), `remainOpacity = 1.0 - destColor.a` [S1, unofficial].
    *   Calculate initial `posX[i]` for additional volumes: $posX[i] = P1toPN[i] \cdot pos$ [S1, unofficial].
    *   **While Loop:** Continue while `t < tMax`, `remainOpacity >= 0.0039` (1/255), and `pos` is within `lowBounds[0]` and `highBounds[0]` [S1, unofficial].
        *   **Sample Volume 0:**
            *   If `pos` is inside bounds and not clipped:
                *   $value[0] = texture3D(dataSetTexture[0], pos)$
                *   $scalar[0] = scalarFromValue(value[0])$
                *   $opacity[0] = texture1D(opacityTexture[0], scalar[0])$
                *   $color[0] = shade(0, value[0]) \cdot opacity[0].a$
                *   $destColor = destColor + color[0] \cdot remainOpacity$
                *   $remainOpacity = remainOpacity \cdot (1.0 - opacity[0].a)$
        *   **Sample Additional Volumes ($i = 1$ to $N-1$):**
            *   If `posX[i-1]` is inside bounds and not clipped:
                *   $value[i] = texture3D(dataSetTexture[i], posX[i-1])$
                *   $scalar[i] = scalarFromValue(value[i])$
                *   $opacity[i] = texture1D(opacityTexture[i], scalar[i])$
                *   $color[i] = shade(i, value[i]) \cdot opacity[i].a$
                *   $destColor = destColor + color[i] \cdot remainOpacity$
                *   $remainOpacity = remainOpacity \cdot (1.0 - opacity[i].a)$
        *   **Advance Ray:**
            *   $pos = pos + rayDir$
            *   $posX[i] = posX[i] + rayDir$ for all $i$ (assuming uniform spacing/transform) or recompute based on $t$ [S1, unofficial].
            *   $t += 1.0$
    *   **Final Output:**
        *   $gl\_FragColor = destColor$
        *   $gl\_FragColor.a = 1.0 - remainOpacity$ [S1, unofficial].

5.  **Masking (Optional):**
    *   If `MaskInput` is provided:
        *   **Binary Mask:** Render only where mask value is 255 (UCHAR) [S1, unofficial].
        *   **Label Map Mask:** Assumes 3 labels (0, 1, 2). Label 0 uses standard CT. Labels 1 and 2 blend standard CT with mask-specific CT using `MaskBlendFactor` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multi_volume(renderer, volumes, properties, mask_input=None):
    # 1. Initialization
    max_mem = probe_gpu_memory()
    if max_mem == 0:
        max_mem = 128 * 1024 * 1024  # 128MB default
    
    # 2. Validation
    if not validate_inputs(volumes, properties):
        return
    
    # 3. Transform Setup
    P1toPN = []
    for i in range(1, len(volumes)):
        T_world_to_tex_i = compute_world_to_texture_transform(volumes[i])
        T_world_to_tex_0 = compute_world_to_texture_transform(volumes[0])
        T_user = get_additional_input_user_transform(i)
        P1toPN.append(T_world_to_tex_i * inverse(T_world_to_tex_0) * T_user)
    
    # 4. Data Preparation
    transformed_volumes = []
    for vol in volumes:
        if vol.changed():
            transformed_volumes.append(adjust_extents_to_zero(vol))
        else:
            transformed_volumes.append(vol)
    
    clipped_bounds = compute_clipped_bounds(transformed_volumes)
    
    # 5. Upload to GPU
    upload_textures(transformed_volumes)
    upload_uniforms(P1toPN, clipped_bounds, properties)
    
    # 6. Ray Casting (Fragment Shader Logic)
    for pixel in screen_pixels:
        pos = compute_entry_position(pixel, volumes[0])
        ray_dir = compute_ray_direction(pixel, camera)
        t_max = compute_max_ray_length(pixel, volumes[0])
        dest_color = vec4(0.0, 0.0, 0.0, 0.0)
        remain_opacity = 1.0
        
        # Initial positions for additional volumes
        pos_x = [P1toPN[i] * vec4(pos, 1.0) for i in range(len(P1toPN))]
        
        t = 0.0
        while t < t_max and remain_opacity > 0.0039:
            # Check bounds for volume 0
            if is_inside(pos, clipped_bounds[0]) and not is_clipped(pos, clipping_planes[0]):
                val_0 = texture3D(data_set_texture[0], pos)
                scalar_0 = scalar_from_value(val_0)
                opac_0 = texture1D(opacity_texture[0], scalar_0).a
                col_0 = shade(0, val_0) * opac_0
                
                dest_color += col_0 * remain_opacity
                remain_opacity *= (1.0 - opac_0)
            
            # Check bounds for additional volumes
            for i in range(len(P1toPN)):
                if is_inside(pos_x[i], clipped_bounds[i+1]) and not is_clipped(pos_x[i], clipping_planes[i+1]):
                    val_i = texture3D(data_set_texture[i+1], pos_x[i])
                    scalar_i = scalar_from_value(val_i)
                    opac_i = texture1D(opacity_texture[i+1], scalar_i).a
                    col_i = shade(i+1, val_i) * opac_i
                    
                    dest_color += col_i * remain_opacity
                    remain_opacity *= (1.0 - opac_i)
            
            # Advance
            pos += ray_dir
            for i in range(len(P1toPN)):
                pos_x[i] += ray_dir  # Simplified; actual implementation may vary
            t += 1.0
            
        # Masking
        if mask_input:
            mask_val = get_mask_value(pixel, mask_input)
            if mask_val == 0:  # Binary mask: 0 is outside
                dest_color = vec4(0.0, 0.0, 0.0, 0.0)
            elif mask_val > 0:  # Label map blending
                # Blend based on MaskBlendFactor
                pass
        
        gl_frag_color = dest_color
        gl_frag_color.a = 1.0 - remain_opacity
```

## Subtopics

### [OpenGL GPU Multi Volume Ray Cast Mapper](2919-opengl-gpu-multi-volume-ray-cast-mapper.md)
The OpenGL GPU Multi Volume Ray Cast Mapper is an OpenGL-specific subclass of the GPU Multi-Volume Ray Casting Mapper that handles hardware extension checks, OpenGL object lifecycle management, and clipping/cropping plane setup for GPU-accelerated volume rendering. It serves as the concrete implementation responsible for drawing the volume ray cast image to the screen by managing fragment shader programs, framebuffer objects, and texture uploads required for rendering multiple volumes simultaneously. It builds on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) by providing the specific OpenGL API bindings and state management necessary for hardware-accelerated ray casting.

### [GPU Volume Ray Casting (VTK Composite Blend)](6912-gpu-volume-ray-casting-vtk-composite-blend.md)
GPU Volume Ray Casting with VTK Composite Blend is a hardware-accelerated direct volume rendering technique that visualizes 3D scalar fields by accumulating color and opacity along rays cast through the volume. It sets itself apart from sibling mappers by explicitly using the `Composite` blend mode within VTK's `vtkGPUVolumeRayCastMapper`, which performs standard front-to-back alpha blending to integrate voxel contributions into the final pixel color. Unlike surface-based approaches, it renders the entire volume data directly without intermediate mesh extraction.

### [vtkSlicerGPURayCastVolumeMapper](7450-vtkslicergpuraycastvolumemapper.md)
Renders a volume using ray casting on a 3D texture. It builds on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) to provide a specific implementation for the 3D Slicer environment, supporting adaptive sampling, masking, and AMR modes via the parent's pipeline hooks. It is distinguished from its siblings by its specific integration with CTK transfer functions and its handling of specific OpenGL uniform layouts for ray casting parameters.

### Mask Type Configuration
Methods to set the mask type for volume rendering, either Binary or LabelMap [S2, unofficial]. To set binary mask: assign `MaskType = vtkGPUMultiVolumeRayCastMapper::BinaryMaskType`. To set label map mask: assign `MaskType = vtkGPUMultiVolumeRayCastMapper::LabelMapMaskType` [S2, unofficial].

### Label Map Masking for Volume Rendering
A masking technique that confines volume rendering to specific regions using either a binary mask or a label map. Supports blending between standard and mask-specific color transfer functions [S1, unofficial]. Binary mask processing assumes UCHAR datatype with values 255 (inside) and 0 (outside); rendering is confined strictly to regions where mask value is 255 [S1, unofficial]. Label map processing assumes UCHAR datatype with exactly 3 labels (0, 1, 2). Label 0 uses component 0's CT. Label 1 uses component 1's CT blended with component 0's CT using `MaskBlendFactor`. Label 2 uses component 2's CT blended with component 0's CT using `MaskBlendFactor` [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(P \cdot S \cdot V)$, where $P$ is the number of pixels, $S$ is the number of samples per ray, and $V$ is the number of volumes. Adaptive sampling reduces $S$ dynamically [S2, unofficial].
*   **Memory Complexity:** $O(V \cdot D)$, where $D$ is the size of each volume dataset. Limited by `MaxMemoryInBytes` [S2, unofficial].
*   **Performance:** Hardware loop limits require a nested while loop structure instead of a single loop over all samples and volumes [S1, unofficial]. `ReportProgress` uses `glFinish()`, which blocks the GPU and can significantly slow down rendering of small datasets; it should be disabled for small datasets [S2, unofficial].
*   **Strengths:** Supports multiple volumes simultaneously, adaptive sampling, masking, and clipping/cropping. Uses GLSL shaders for performance and broad device support [S1, unofficial].
*   **Limitations:** Non-zero extents require a workaround [S2, unofficial]. Invalid cropping planes fail silently [S2, unofficial]. Hardware rendering is unsupported if input data has more than one independent component or if required extensions are missing [S3, unofficial].
*   **When to Use:** When visualizing multiple co-registered volume datasets (e.g., multi-modal medical imaging) is required.
*   **When Not to Use:** For single-volume rendering where simpler mappers suffice, or when hardware lacks required OpenGL extensions [S3, unofficial].

## Background
The GPU Multi-Volume Ray Casting Mapper was developed by Carlos Falcón and Karl Krissian as an extension to the Visualization Toolkit (VTK) [S1, unofficial]. It was created to address the limitations of the original `vtkGPUVolumeRayCastMapper`, which only supported single-volume rendering. The project uses GLSL shaders instead of OpenCL for general GPU programming because shaders are faster and supported by more devices [S1, unofficial]. It is used in various visualization applications requiring multi-volume rendering, such as medical imaging and scientific visualization [S1, S2, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
[S3] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-27 (unofficial)
