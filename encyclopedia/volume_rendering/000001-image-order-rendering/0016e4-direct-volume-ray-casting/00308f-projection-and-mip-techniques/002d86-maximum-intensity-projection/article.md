# Maximum Intensity Projection

Also called: Max Intensity Projection, Ray Traversal -MIP, Local Maximum Intensity Projection (LMIP)

## Overview
Maximum Intensity Projection (MIP) is a volume rendering technique that projects the maximum scalar intensity encountered along each viewing ray onto a 2D image plane [S1, S2, S4, S7, S13, S18, S19, S27, S33, unofficial]. It solves the problem of visualizing high-density structures, such as contrast-enhanced blood vessels in angiography or bones in CT scans, by emphasizing the brightest voxels while ignoring intermediate values [S3, S8, S18, S24, S29, unofficial]. Unlike standard alpha-blending which accumulates opacity and color, MIP is order-independent and discards all spatial depth information, making it distinct from its sibling [Minimum Intensity Projection](5299-minimum-intensity-projection.md) and [Forward Projection (aTx)](1492-forward-projection-atx.md) [S2, S31, S33, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset $V$ with dimensions $D_x \times D_y \times D_z$, a view direction $\mathbf{d}$, and a camera origin $\mathbf{o}$ defining rays $r(t) = \mathbf{o} + t\mathbf{d}$. A transfer function $T$ maps scalar values to color/opacity [S2, S19, S26, unofficial].
*   **Output:** A 2D image where each pixel $(u,v)$ contains the color corresponding to the maximum scalar value found along the ray passing through that pixel [S7, S13, S27, unofficial].

**Data Structures:**
*   **Volume Texture:** A 3D texture (e.g., `Texture3D<float>` or `texture_3d<f32>`) storing voxel intensities. Formats may include `rg8unorm` where red/green channels encode a 16-bit value, or single-channel float [S11, S18, unofficial].
*   **Accumulator:** A single scalar variable `max_val` (or `maximum_intensity`) initialized to the minimum possible value (e.g., $-\infty$, $0.0$, or $-1.0$) per pixel [S2, S8, S12, S13, S18, S26, S27, S28, unofficial].
*   **Ray Bounds:** Entry ($t_{start}$) and exit ($t_{end}$) distances along the ray, typically computed via AABB intersection [S1, S11, S18, unofficial].

**Algorithm Steps:**
1.  **Ray Generation:** For each pixel $(u,v)$, compute the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ in world/volume space [S11, S18, unofficial].
2.  **Intersection:** Calculate the entry point $t_{start}$ and exit point $t_{end}$ of the ray with the volume's axis-aligned bounding box (AABB) [S1, S11, S18, unofficial]. If the ray misses the volume ($t_{start} \ge t_{end}$), output background color [S11, unofficial].
3.  **Discretization:** Determine the number of samples $N$ or step size $\Delta t$. A random offset is often added to the initial sample position to prevent banding artifacts [S1, S11, S18, S28, unofficial].
4.  **Traversal and Aggregation:** Iterate from $t = t_{start}$ to $t_{end}$ with step $\Delta t$:
    *   Compute sample position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ [S16, S28, unofficial].
    *   Sample the volume value $s = V(\mathbf{p})$ using trilinear interpolation [S6, S7, S33, S40].
    *   Update the accumulator: $I_{max} \leftarrow \max(I_{max}, s)$ [S1, S2, S27, S28, unofficial].
    *   **Early Termination (Optional):** If $I_{max}$ exceeds a threshold (e.g., $0.75 \times \text{volume\_maximum}$), terminate the ray early [S4, S8, unofficial].
5.  **Classification:** Map the final $I_{max}$ to color using the transfer function: $\text{Color} = T(I_{max})$ [S2, S4, S14, S19, S26, unofficial].
6.  **Output:** Write the resulting color to the pixel buffer [S8, S12, S27, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mip(volume_texture, transfer_function, image_width, image_height, 
               ray_origin, ray_direction, step_size, threshold=None):
    # Initialize output image
    output_image = [[0.0] * 4 for _ in range(image_height * image_width)]
    
    # Precompute volume bounds and max value if needed for threshold
    vol_max = get_volume_maximum(volume_texture) # Implementation detail [S8, unofficial]
    
    for y in range(image_height):
        for x in range(image_width):
            # 1. Ray setup (simplified; assumes ray_origin/direction are precomputed per pixel)
            # In practice, ray_origin and ray_direction depend on (x,y)
            current_ray_o = get_ray_origin(x, y, ray_origin)
            current_ray_d = get_ray_direction(x, y, ray_direction)
            
            # 2. Intersection with AABB
            t_near, t_far = intersect_aabb(current_ray_o, current_ray_d, volume_bounds)
            
            # Edge case: Ray misses volume
            if t_near >= t_far:
                continue
                
            # 3. Initialization
            max_val = -float('inf')
            max_pos = None
            
            # Optional: Random offset to prevent banding [S1, unofficial]
            offset = random.uniform(0, step_size)
            
            # 4. Traversal
            t = t_near + offset
            while t < t_far:
                # Sample position
                pos = current_ray_o + current_ray_d * t
                
                # Sample volume (trilinear interpolation) [S6, S33, unofficial]
                # Assuming volume_texture.sample(pos) returns a scalar intensity
                intensity = volume_texture.sample(pos)
                
                # Aggregation: Maximum
                if intensity > max_val:
                    max_val = intensity
                    max_pos = pos
                
                # Early termination check [S4, S8, unofficial]
                if threshold is not None and max_val > threshold:
                    break
                
                t += step_size
            
            # 5. Classification
            # If no valid value found (e.g., ray entirely outside valid data range)
            if max_val == -float('inf'):
                color = [0.0, 0.0, 0.0, 1.0]
            else:
                # Map max intensity to color via transfer function [S2, S19, unofficial]
                color = transfer_function(max_val)
                
            output_image[y * image_width + x] = color
            
    return output_image
```

**Parameters:**
*   **Step Size / Number of Steps:** Controls sampling density. Typical values range from 1 to 50 pixels per voxel depending on performance needs [S8, S9, S13, unofficial].
*   **Threshold:** A value to trigger early termination. Commonly set to $0.75 \times \text{volume\_maximum}$ [S4, S8, unofficial].
*   **Window Width/Level:** Used in post-processing to adjust contrast and brightness of the final MIP image [S18, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth Loss:** MIP completely loses depth information; a high-intensity structure in the back can occlude a lower-intensity structure in the front [S5, S8, S21, S24, S32, unofficial].
*   **Order Independence:** The result is independent of the order in which samples are processed, allowing for parallelization and acceleration structures [S2, unofficial][S23, book][S31, unofficial].
*   **Hardware Support:** Standard OpenGL blending does not support maximum selection; the `GL_EXT_blend_minmax` extension is required for hardware-accelerated MIP [S6, unofficial].
*   **Degenerate Rays:** Rays parallel to axes or missing the volume must be handled to avoid invalid memory access or infinite loops [S11, S18, unofficial].

## Subtopics

### [Local Maximum Intensity Projection](6054-local-maximum-intensity-projection.md)
Selects the first local intensity maximum above a threshold rather than the global maximum, preserving some depth cues.

### [Fast Maximum Intensity Projection (MIP)](6635-fast-maximum-intensity-projection-mip.md)
Accelerates MIP by decomposing the density domain into intervals and using shear-warp projection to avoid unnecessary resampling.

### [MIP with Offset Lookup](6637-mip-with-offset-lookup.md)
Uses binary volume masks and bitwise operations to skip low-density intervals, focusing only on voxels in the highest density interval.

### Morphological Maximum Intensity Projection (MMIP) with Morphological Closing
A variant for non-axial views that prevents hole formation due to undersampling by applying a final morphological closing operation to the 2D projection image after voxel projection [S36]. It uses a 2x2x2 structuring element for morphological sampling [S36].

### Closest Vessel Projection
A local projection technique often used in slab imaging for vascular diagnosis. It requires setting a dataset-specific threshold for the local maximum, making it more difficult to apply than standard MIP [S22].

### FVR Approximation of Maximum Intensity Projection
Approximates MIP using Fourier Volume Rendering (FVR) by preprocessing the dataset with $v = b^v$ (e.g., $b=2$), performing FVR, and taking the base-$b$ logarithm. This suffers from artifacts due to non-linear compression of the resampling filter tails [S35, unofficial].

## Operating characteristics
*   **Complexity:** Time complexity is $O(N \cdot P)$ where $N$ is the number of samples per ray and $P$ is the number of pixels. Memory complexity is $O(V)$ for the volume storage.
*   **Performance:** MIP is generally faster than full alpha-blending because it avoids accumulation and opacity calculations [S6, S32, unofficial]. Acceleration structures (e.g., multilevel grids, priority queues) can significantly speed up search by skipping regions with lower maximums [S2, unofficial].
*   **Strengths:** Excellent for highlighting high-intensity structures like blood vessels or bones; simple implementation; order-independent [S3, S18, S29, unofficial].
*   **Limitations:** No depth information; prone to occlusion artifacts where distant bright structures hide closer dim ones [S5, S8, S21, S24, unofficial].
*   **When to Use:** Angiography, bone visualization, quick inspection of high-density features [S1, S7, S24, S29, unofficial].
*   **When Not to Use:** When depth perception is critical or when structures have similar intensities but different spatial relationships [S5, S32, unofficial].

## Background
MIP was introduced as a simplified volume rendering technique for medical imaging, particularly for Magnetic Resonance Angiography (MRA) and CT angiography [S1, S4, S7, S18, unofficial]. It emerged as an alternative to complex integration of the radiative transfer equation, providing a fast way to visualize contrast-enhanced vessels [S6, S24, unofficial]. The technique is widely supported in visualization toolkits like VTK (`vtkVolumeRayCastMIPFunction`) [S4, S31, unofficial] and is a standard feature in medical imaging workstations [S7, unofficial][S25].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
[S2] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S3] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S4] Direct Volume Rendering (DVR): Ray-casting | PPT - SlideShare, https://www.slideshare.net/slideshow/direct-volume-rendering-dvr-raycasting/265104120, fetched 2026-10-26 (unofficial)
[S5] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S6] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S7] BernhardKidalka/D3DVolumeRaycaster, https://github.com/BernhardKidalka/D3DVolumeRaycaster, fetched 2026-10-26 (unofficial)
[S8] rhnfzl/raycasting-volume-rendering, https://github.com/rhnfzl/raycasting-volume-rendering, fetched 2026-10-26 (unofficial)
[S9] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S10] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S11] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S12] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S13] indira654/implementing-basic-X-Ray-and-MIP-volume-rendering-using-ray-casting, https://github.com/indira654/implementing-basic-X-Ray-and-MIP-volume-rendering-using-ray-casting, fetched 2026-10-26 (unofficial)
[S14] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S15] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S16] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S17] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S18] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
[S19] 1vx-437312114/webgl_volume_renderer, https://github.com/1vx-437312114/webgl_volume_renderer, fetched 2026-10-27 (unofficial)
[S20] urielfederez/Volume-Data-Visualization, https://github.com/urielfederez/Volume-Data-Visualization, fetched 2026-10-27 (unofficial)
[S21] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S22] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S23] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S24] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S25] VStudio - A Framework for Interactive Volume Visualization, https://www.cg.tuwien.ac.at/courses/Visualisierung/2008-2009/Beispiel1/Hausmair/index.html, fetched 2026-10-27
[S26] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S27] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
[S28] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
[S29] PDF Direct Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/DVR.pdf, fetched 2026-10-28 (unofficial)
[S30] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S31] Direct Volume Rendering, https://cscheid.net/writing/data_visualization/direct_volume_rendering/, fetched 2026-10-29 (unofficial)
[S32] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S33] Indirect volume rendering, https://slidetodoc.com/direct-volume-rendering-what-is-volume-rendering-accumulate/, fetched 2026-10-29 (unofficial)
[S34] https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
[S35] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
[S36] Comparison of Morphological Pyramids for Multiresolution MIP Volume Rendering, https://doi.org/10.2312/vissym/vissym02/061-070, fetched 2026-10-29
