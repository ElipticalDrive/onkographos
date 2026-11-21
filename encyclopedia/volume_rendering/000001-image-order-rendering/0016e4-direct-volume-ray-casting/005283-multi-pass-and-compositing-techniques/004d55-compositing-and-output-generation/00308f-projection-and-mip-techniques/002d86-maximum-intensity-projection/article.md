# Maximum Intensity Projection

Also called: MIP, MIP rendering, Max Intensity Projection

## Overview
Maximum Intensity Projection (MIP) is a volume rendering technique that projects the maximum scalar value encountered along a viewing ray onto the 2D image plane [S1, S2, unofficial][S3][S4, S5, S6, S7, S8, S9, S10, S11, S12, S13, S14, S15, S16, S17, S18, S19, S20, unofficial][S21][S22, S23, S24, S25, S26, S27, S28, S29, S30, S31, S32, S33, S34, S35, S36, unofficial][S37][S38, book][S39][S40, unofficial][S41][S42, S43, S44, S45, S46, S47, unofficial][S48][S49, S50, S51, S52, S53, S54, S55, unofficial][S56]. Unlike standard alpha blending which accumulates emission and absorption [S1, S2, unofficial][S3][S4, S5, S6, S7, S8, S9, S10, S11, S12, S13, S14, S15, S16, S17, S18, S19, S20, unofficial][S21][S22, S23, S24, S25, S26, S27, S28, S29, S30, S31, S32, S33, S34, S35, S36, unofficial][S37][S38, book][S39][S40, unofficial][S41][S42, S43, S44, S45, S46, S47, unofficial][S48][S49, S50, S51, S52, S53, S54, S55, unofficial][S56], MIP ignores intermediate values and opacity accumulation, making it order-independent [S2, S4, S5, S10, S12, S25, unofficial][S39][S45, S49, S50, unofficial]. It is primarily used in medical imaging to highlight high-intensity structures such as bones in CT scans or contrast-enhanced blood vessels in MRI/CT angiography [S1, S2, unofficial][S3][S4, S6, S7, S8, S9, S11, S12, S13, S14, S15, S16, S17, S18, S19, S20, unofficial][S21][S22, S23, S24, S25, S26, S27, S28, S29, S30, S31, S32, S33, S34, S35, S36, unofficial][S37][S38, book][S39][S40, unofficial][S41][S42, S43, S44, S45, S46, S47, unofficial][S48][S49, S50, S51, S52, S53, S54, S55, unofficial][S56].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset $V$ containing scalar values (density/intensity) $s$, and a viewing ray $R$ defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel.
*   **Output:** A 2D image where each pixel color is determined by the maximum scalar value $s_{max}$ found along the ray, mapped through a transfer function (or direct intensity mapping) [S4, S16, S22, S34, S36, S42, S53, unofficial].

**Data Structures:**
*   **Volume Texture:** Typically stored as a 3D texture. Common layouts include `rg8unorm` where the red channel holds the low bits and the green channel holds the high bits of a 16-bit value [S32, unofficial], or single-channel float formats (`RedFormat`, `FloatType`) [S14, unofficial].
*   **Accumulator:** A single variable `max_val` (or `maximum_intensity`) initialized to the minimum possible value (e.g., $-\infty$, $-999999.0$, or $0.0$ depending on data range) [S2, S13, S16, S17, S22, S24, S27, S32, S34, S42, S43, S44, S52, unofficial]. No opacity accumulator is needed [S2, S10, S12, S25, S50, unofficial].

**Numbered Steps:**
1.  **Ray Setup:** For each pixel, compute the ray origin and direction in volume space. This often involves unprojecting screen coordinates to 3D space [S20, S32, unofficial].
2.  **Intersection:** Compute the entry (`near`) and exit (`far`) points of the ray with the volume's axis-aligned bounding box (AABB) using the slab method [S20, S32, unofficial]. If the ray misses the volume (`near > far` or `far < 0`), output background color [S20, S32, unofficial].
3.  **Initialization:** Initialize `max_val` to the minimum scalar value and `max_pos` to the ray origin or invalid [S2, S13, S16, S17, S22, S24, S27, S32, S34, S42, S43, S44, S52, unofficial].
4.  **Ray Marching:** Iterate along the ray from `near` to `far` with a step size $\Delta t$ [S14, S16, S20, S24, S32, S34, S44, S52, unofficial].
    *   Sample the volume at the current position $P = \mathbf{o} + t \cdot \mathbf{d}$ [S14, S16, S20, S22, S24, S34, S43, S44, S52, unofficial].
    *   Retrieve the scalar value $s$ from the volume texture [S14, S16, S22, S24, S34, S43, S44, S52, unofficial].
    *   Update the maximum: `max_val = max(max_val, s)` [S2, S10, S13, S22, S34, S42, S43, S44, S52, unofficial].
    *   Optionally, track the position `max_pos` where `max_val` was updated [S22, S35, S52, unofficial].
    *   *Early Termination:* Some implementations break early if `max_val` exceeds a threshold (e.g., $0.75 \times \text{volume\_maximum}$) [S16, unofficial] or if the ray length is exhausted [S17, S34, unofficial]. Note: Standard MIP does not use opacity-based early termination like alpha blending [S2, S10, S12, S25, S50, unofficial].
5.  **Color Mapping:** Map `max_val` to a color. This can be a direct intensity-to-color mapping (e.g., window/level) [S32, S45, S53, unofficial] or a transfer function lookup [S22, S34, S35, S42, S53, unofficial].
    *   If lighting is desired (e.g., DepthShaded MIP), compute gradients and lighting at `max_pos` [S35, S52, unofficial].
6.  **Output:** Write the mapped color to the pixel buffer [S2, S16, S22, S34, S35, S42, S44, S52, S53, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mip_pixel(ray_origin, ray_direction, volume_texture, step_size):
    # 1. Intersection with AABB
    inv_dir = 1.0 / ray_direction
    bot = inv_dir * (bbox_min - ray_origin)
    top = inv_dir * (bbox_max - ray_origin)
    rmin = min(top, bot)
    rmax = max(top, bot)
    near = max(rmin.x, rmin.y, rmin.z)
    far = min(rmax.x, rmax.y, rmax.z)

    # Handle miss
    if near > far or far < 0:
        return background_color

    # Clamp near to 0
    near = max(0, near)
    
    # 2. Initialize
    max_val = -infinity
    max_pos = ray_origin + ray_direction * near
    
    # 3. Ray Marching
    t = near
    while t < far:
        # Sample volume
        sample_pos = ray_origin + ray_direction * t
        # Extract scalar value (e.g., red channel or single channel)
        s = texture_sample(volume_texture, sample_pos).r
        
        # Update maximum
        if s > max_val:
            max_val = s
            max_pos = sample_pos
            
        t += step_size

    # 4. Color Mapping
    # Example: Window/Level mapping or Transfer Function lookup
    # color = transfer_function(max_val)
    # For simple grayscale MIP:
    color = map_intensity_to_color(max_val)
    
    return color
```

**Parameters:**
*   **Step Size:** Determines sampling density. Can be fixed (e.g., 0.5 units) [S24, unofficial] or adaptive. In interactive modes, larger steps (e.g., 50) are used [S16, unofficial]. Default values often range from 1 to 200 samples per ray [S14, S18, S34, unofficial].
*   **Window Width/Level:** Controls contrast and brightness of the final intensity mapping [S32, S45, S53, unofficial]. Typical GUI values: Width $\approx 0.018$ (range 0–0.05), Level $\approx 0.498$ (range 0.48–0.52) [S32, unofficial].
*   **Thresholds:** Early exit thresholds (e.g., $0.75 \times \text{max\_density}$) [S16, unofficial] or density bounds ($minDensityThreshold$, $maxDensityThreshold$) to ignore noise [S22, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth Ambiguity:** MIP loses all depth information [S2, S4, S11, S12, S16, S20, S36, unofficial][S37][S40, S50, unofficial]. A high-intensity structure in the foreground can obscure lower-intensity structures behind it, leading to misinterpretation of spatial relationships [S4, S11, S12, S16, S40, unofficial].
*   **Order Independence:** Since the operation is `max()`, the order of sample processing does not matter [S2, unofficial][S38, book][S45, S49, unofficial]. This allows for parallelization and acceleration structures.
*   **Implementation Specifics:** Standard OpenGL blending cannot compute `max` without extensions (e.g., `GL_EXT_blend_minmax`) [S12, unofficial][S48]. Hardware sampling requires explicit LOD setting (e.g., `SampleLevel` with LOD 0) for 3D textures with a single mipmap level to avoid gradient calculation errors [S15, unofficial].

## Subtopics

### [Local Maximum Intensity Projection](6054-local-maximum-intensity-projection.md)
Projects the first local maximum above a threshold instead of the global maximum to preserve depth cues and reduce occlusion.

### [Fast Maximum Intensity Projection (MIP)](6635-fast-maximum-intensity-projection-mip.md)
Accelerates MIP by decomposing the density domain into intervals and using shear-warp projection to skip low-density samples.

### [MIP with Offset Lookup](6637-mip-with-offset-lookup.md)
Uses binary volume masks and bitwise operations to identify the highest density interval along a ray, skipping low-density voxels entirely.

### [Priority Queue MIP with Min/Max Grids](10338-priority-queue-mip-with-min-max-grids.md)
Combines hierarchical spatial acceleration (min/max grids) with a priority queue to skip nodes whose maximum value is lower than the current ray's maximum.

### [DepthShaded Maximum Intensity Projection](11191-depthshaded-maximum-intensity-projection.md)
Enhances depth perception by blending the MIP color with a depth-derived shading cue, retaining the global maximum but adding spatial context.

### [Maximum Activity Projection](12564-maximum-activity-projection.md)
Used in nuclear medicine; applies simulated exponential attenuation before selecting the maximum to weight voxels by depth and prevent rotation illusions.
*   #### Morphological Maximum Intensity Projection (MMIP) with Morphological Closing
    Prevents hole formation in non-axial views caused by undersampling by applying a morphological closing operation (e.g., 2x2x2 structuring element) to the final 2D projection image.
*   #### Closest Vessel Projection
    A local projection technique for vascular diagnosis that selects the first local maximum above a dataset-specific threshold. Requires careful threshold tuning.
*   #### FVR Approximation of Maximum Intensity Projection
    Approximates MIP using Fourier Volume Rendering by transforming voxel values $v \to b^v$ (e.g., $b=2$ or $1.1$), performing FVR projection, and applying a base-$b$ logarithm. Suffers from artifacts due to non-linear compression of resampling filters.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples along the ray [S2, S55, unofficial]. With acceleration structures (e.g., Priority Queue, Fast MIP), average case can be significantly lower [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per ray accumulator (excluding volume storage) [S2, unofficial].
*   **Performance:** Very fast to calculate; often used as a default or preview rendering mode [S21][S22, unofficial]. Suitable for real-time interactive visualization [S21, S39].
*   **Strengths:** Highlights high-intensity structures (bones, contrast agents) clearly [S6, S12, S32, unofficial][S39]. No transfer function required for basic intensity visualization [S4, S12, unofficial]. Order-independent, allowing flexible rendering order [S45, S49, unofficial].
*   **Limitations:** Complete loss of depth information [S4, S11, S12, S16, S36, unofficial][S37][S40, S50, unofficial]. Misleading spatial correlation; foreground objects can hide background objects [S4, S11, S12, S40, unofficial]. Less effective for MR images where contrast is lower or noise is higher compared to CT [S44, unofficial].

## Background
MIP was introduced as a simple compositing technique for direct volume rendering, particularly for medical imaging applications [S1, S2, unofficial][S3][S4, S5, S6, S7, S8, S9, S10, S11, S12, S13, S14, S15, S16, S17, S18, S19, S20, unofficial][S21][S22, S23, S24, S25, S26, S27, S28, S29, S30, S31, S32, S33, S34, S35, S36, unofficial][S37][S38, book][S39][S40, unofficial][S41][S42, S43, S44, S45, S46, S47, unofficial][S48][S49, S50, S51, S52, S53, S54, S55, unofficial][S56]. It is widely used in Magnetic Resonance Angiography (MRA) and Computed Tomography Angiography (CTA) to visualize blood vessels without the need for complex transfer functions [S1, S2, unofficial][S3][S4, S6, S7, S8, S9, S11, S12, S13, S14, S15, S16, S17, S18, S19, S20, unofficial][S21][S22, S23, S24, S25, S26, S27, S28, S29, S30, S31, S32, S33, S34, S35, S36, unofficial][S37][S38, book][S39][S40, unofficial][S41][S42, S43, S44, S45, S46, S47, unofficial][S48][S49, S50, S51, S52, S53, S54, S55, unofficial][S56]. The technique simplifies the volume rendering integral to a maximum operation [S11, S12, S19, S40, unofficial]. Variants and acceleration methods have been developed to address its limitations, such as depth loss [S1, unofficial].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
[S2] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S3] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S4] Volume Rendering Volume Rendering. Lecture 21. Acknowledgements. These slides are collected from many sources. A particularly valuable source is the IEEE Visualization conference tutorials. Sources fr, https://www.slideserve.com/hayes-levine/volume-rendering-powerpoint-ppt-presentation, fetched 2026-10-25 (unofficial)
[S5] Real-Time Volume Graphics [03] GPU-Based Volume Rendering Real-Time Volume Graphics [03] GPU-Based Volume Rendering. Volume Rendering. Image order approach:. Data Set. Image Plane. Eye. For each pixel, https://www.slideserve.com/risa/real-time-volume-graphics-03-gpu-based-volume-rendering, fetched 2026-10-25 (unofficial)
[S6] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S7] Direct Volume Rendering Data, https://www.scribd.com/document/1050958395/Direct-Volume-Rendering-Data, fetched 2026-10-25 (unofficial)
[S8] Volume Ray Casting Techniques, https://www.scribd.com/document/102700078/03-raycasting, fetched 2026-10-25 (unofficial)
[S9] Direct Volume Rendering (DVR): Ray-casting | PPT - SlideShare, https://www.slideshare.net/slideshow/direct-volume-rendering-dvr-raycasting/265104120, fetched 2026-10-26 (unofficial)
[S10] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S11] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S12] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S13] joshcodd/volume-rendering, https://github.com/joshcodd/volume-rendering, fetched 2026-10-26 (unofficial)
[S14] Konstiu/VolumeVisualization, https://github.com/Konstiu/VolumeVisualization, fetched 2026-10-26 (unofficial)
[S15] BernhardKidalka/D3DVolumeRaycaster, https://github.com/BernhardKidalka/D3DVolumeRaycaster, fetched 2026-10-26 (unofficial)
[S16] rhnfzl/raycasting-volume-rendering, https://github.com/rhnfzl/raycasting-volume-rendering, fetched 2026-10-26 (unofficial)
[S17] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S18] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S19] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S20] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S21] QuickVol: A lightweight browser tool for immersive visualizations of volumetric data., https://doi.org/10.1016/j.isci.2024.111379, fetched 2026-10-26
[S22] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S23] cjcamp16/sciviz-paraview, https://github.com/cjcamp16/sciviz-paraview, fetched 2026-10-26 (unofficial)
[S24] indira654/implementing-basic-X-Ray-and-MIP-volume-rendering-using-ray-casting, https://github.com/indira654/implementing-basic-X-Ray-and-MIP-volume-rendering-using-ray-casting, fetched 2026-10-26 (unofficial)
[S25] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S26] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S27] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S28] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S29] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S30] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S31] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
[S32] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
[S33] mneela/Volume-Rendering, https://github.com/mneela/Volume-Rendering, fetched 2026-10-27 (unofficial)
[S34] 1vx-437312114/webgl_volume_renderer, https://github.com/1vx-437312114/webgl_volume_renderer, fetched 2026-10-27 (unofficial)
[S35] urielfederez/Volume-Data-Visualization, https://github.com/urielfederez/Volume-Data-Visualization, fetched 2026-10-27 (unofficial)
[S36] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S37] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S38] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S39] Real-Time Volume Rendered MRI for Interventional Guidance, https://pmc.ncbi.nlm.nih.gov/articles/PMC2570028/, fetched 2026-10-27
[S40] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S41] VStudio - A Framework for Interactive Volume Visualization, https://www.cg.tuwien.ac.at/courses/Visualisierung/2008-2009/Beispiel1/Hausmair/index.html, fetched 2026-10-27
[S42] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S43] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
[S44] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
[S45] PDF Volume Rendering - Isenberg, https://tobias.isenberg.cc/graphics/uploads/Class/07-volume-rendering.pdf, fetched 2026-10-28 (unofficial)
[S46] PDF Direct Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/DVR.pdf, fetched 2026-10-28 (unofficial)
[S47] Volume Rendering, Maximum Intensity Projection and ..., https://ryanrossi.com/sv3.php, fetched 2026-10-28 (unofficial)
[S48] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S49] Direct Volume Rendering, https://cscheid.net/writing/data_visualization/direct_volume_rendering/, fetched 2026-10-29 (unofficial)
[S50] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S51] Indirect volume rendering, https://slidetodoc.com/direct-volume-rendering-what-is-volume-rendering-accumulate/, fetched 2026-10-29 (unofficial)
[S52] https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
[S53] GENERAL IMPLEMENTATION ASPECTS OF THE GPU- ..., http://sv-journal.org/2011-1/02/, fetched 2026-10-30 (unofficial)
[S54] Volume Rendering in Unity3D - Speaker Deck, https://speakerdeck.com/mlavik1/volume-rendering-in-unity3d, fetched 2026-10-30 (unofficial)
[S55] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
[S56] Comparison of Morphological Pyramids for Multiresolution MIP Volume Rendering, https://doi.org/10.2312/vissym/vissym02/061-070, fetched 2026-10-29
