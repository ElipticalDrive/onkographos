# Image-Parallel Ray Casting via 2D Texture Mapping

Also called: 2D Texture Slicing, 2D Texture Volume Rendering, Volume Rendering with 2D Textures, 2D Texture-Mapped Volume Rendering, 2D Texture Mapping Volume Rendering, Volume Rendering with Texture Slicing, VR Basado en Texturas 2D

## Overview
Image-Parallel Ray Casting via 2D Texture Mapping is an early GPU volume rendering approach that replaces trilinear interpolation with bilinear interpolation by storing the volume as a stack of 2D textures [S1][S4, unofficial][S9]. It solves the problem of rendering volumetric data on hardware lacking native 3D texture support by projecting parallel slices of the dataset onto the image plane and compositing them using hardware alpha-blending [S2, S9]. It differs from its parent, the [Planes Method](3027-planes-method.md), which uses 3D textures, by requiring three full copies of the volume data (one for each axis) to maintain interactivity, and from [Texture-Mapped Volume Rendering with CPU Shading](3423-texture-mapped-volume-rendering-with-cpu-shading.md) by relying on hardware for the core resampling and compositing steps [S2][S8, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset $V$ of dimensions $W \times H \times D$ (width, height, depth).
*   **Output:** A 2D image frame buffer.

**Data Structures:**
The volume is stored in GPU memory as three separate stacks of 2D textures, one stack for each major axis (X, Y, Z). Each stack contains $D$ (or $W$ or $H$) textures.
*   **Texture Format:** Typically 8-bit luminance (`GL_LUMINANCE`) or 8-bit indexed color for palette-based transfer functions [S5][S7, unofficial].
*   **Texture Parameters:**
    *   Wrap mode: `GL_CLAMP` [S7, unofficial].
    *   Filter: `GL_LINEAR` (bilinear interpolation) [S7, unofficial].
*   **Geometry:** Proxy geometry consists of quads (rectangles) aligned with the chosen base plane. Vertices are typically normalized to $[-1, 1]$ in X and Y, with Z determined by the slice index [S7, unofficial].

**Algorithm Steps:**

1.  **Volume Preparation:**
    Partition the volume data into slices along all three axes. Store each slice as a separate 2D texture in GPU memory. This requires three full copies of the volume data in texture memory [S2][S4, S8, unofficial].
    *   *Note:* If memory is constrained, textures can be paged in/out, but this incurs a performance penalty [S5].

2.  **View Direction Analysis:**
    Determine the "base plane" by computing the angle between the viewing direction vector $\vec{v}$ and the three volume axes (X, Y, Z). Select the axis with the minimum angle (i.e., the axis most parallel to the viewing direction, meaning the slices perpendicular to it are most parallel to the image plane) [S8, unofficial].
    *   *Tie-breaking:* If the angles are equal (e.g., at 45 degrees), an arbitrary choice is made [S8, unofficial].

3.  **Slice Selection:**
    Select the texture stack corresponding to the chosen base plane [S6, unofficial][S11, book].

4.  **Rendering Loop (Back-to-Front or Front-to-Back):**
    Iterate through the selected slices in depth order (typically back-to-front for standard alpha blending) [S9][S11, book]. For each slice $i$:
    a.  **Bind Texture:** Bind the 2D texture corresponding to slice $i$ [S7, unofficial].
    b.  **Generate Geometry:** Create a quad representing the slice. The quad's vertices map to the texture coordinates $(0,0)$ to $(1,1)$ [S7, unofficial].
        *   *Vertex Position:* The quad is positioned at the slice's depth in view space. For a stack of $N$ slices, the Z-coordinate might be calculated as $1.0 - \frac{i}{N}$ (normalized) [S7, unofficial].
        *   *Normal:* The normal points towards the viewer (e.g., $(0,0,-1)$ in view space) [S7, unofficial].
    c.  **Rasterization:** The graphics hardware rasterizes the quad. During this step, bilinear interpolation is performed on the 2D texture to sample color/opacity values for each pixel covered by the quad [S4, S8, unofficial].
    d.  **Compositing:** The sampled values are blended into the frame buffer using the hardware's alpha-blending capabilities (e.g., `GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA`) [S4, unofficial][S9].

5.  **Shading and Classification (Optional/Advanced):**
    *   **Gradients:** Pre-computed gradients can be stored in RGB channels of the texture and bilinearly interpolated. The fragment shader must normalize these gradients after projection for accurate lighting [S8, unofficial].
    *   **Classification:** Can be performed pre-interpolation (in the texture) or post-interpolation (in the fragment shader) [S8, unofficial].
    *   **Opacity Correction:** To mitigate artifacts from discrete sampling, opacities can be scaled linearly according to the distance between samples [S8, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
def init_volume(volume_data, width, height, depth):
    # Create three stacks of textures, one for each axis
    # Stack X: slices perpendicular to X axis
    # Stack Y: slices perpendicular to Y axis
    # Stack Z: slices perpendicular to Z axis
    stacks = [create_2d_texture_stack(volume_data, axis) for axis in [X, Y, Z]]
    return stacks

def render(stacks, view_direction, width, height):
    # 1. Determine base plane
    # Calculate angles between view_direction and axes (1,0,0), (0,1,0), (0,0,1)
    angle_x = angle_between(view_direction, (1, 0, 0))
    angle_y = angle_between(view_direction, (0, 1, 0))
    angle_z = angle_between(view_direction, (0, 0, 1))
    
    # Select axis with minimum angle
    min_angle = min(angle_x, angle_y, angle_z)
    if min_angle == angle_x:
        stack = stacks[0]
    elif min_angle == angle_y:
        stack = stacks[1]
    else:
        stack = stacks[2]
        
    # 2. Render slices in back-to-front order
    # Assuming stack is ordered from back (far) to front (near)
    for i in range(len(stack)):
        texture_id = stack[i]
        
        # Bind texture
        bind_texture(texture_id)
        
        # Set texture parameters (if not set once at init)
        set_texture_wrap(GL_CLAMP)
        set_texture_filter(GL_LINEAR)
        
        # Generate quad geometry
        # Vertices in normalized device coordinates or view space
        # Mapping texture coords (0,0) to (1,1)
        draw_quad(slice_index=i, total_slices=len(stack))
        
        # Hardware performs rasterization, bilinear interpolation, and blending
        # Blending mode: GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA
```

**Edge Cases and Pitfalls:**
*   **Axis Switching:** When the view direction crosses the 45-degree threshold between axes, the active texture stack changes abruptly. This causes "popping" artifacts (sudden changes in pixel intensity) because the sampling density and direction change discontinuously [S4, S8, unofficial].
*   **Aliasing:** Lack of interpolation between slices (only bilinear within slices) can cause aliasing if the volume or transfer function has high frequencies [S8, unofficial].
*   **Edge-On View:** If the view direction is parallel to the primary axes of the pre-computed slices, the projection distorts the slice views, degrading image quality. If parallel to the slices, no image is generated [S9].
*   **Precision:** The method is equivalent to standard ray casting only if precision is not lost due to integer arithmetic in the compositing step [S9].

## Subtopics

### [Distorted Video Volumes (Horseshoe Mapping)](7004-distorted-video-volumes-horseshoe-mapping.md)
Renders deformed volumetric data by distorting texture coordinates to allow standard planar slicing on non-planar data.
*   *No other children listed.*

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of slices and $P$ is the number of pixels covered by the projected volume. The rendering speed depends only on image size, not scene complexity [S3, unofficial].
*   **Memory Complexity:** $3 \times W \times H \times D$ (in terms of texture storage), as three full copies of the volume are required [S2][S4, unofficial].
*   **Performance:** On an nVidia GeForce 7800 GT, rendering a 35x35x35 lattice with slice spacing 0.2 and ~90,000 pixels achieved 197 FPS (no noise) to 30 FPS (complex noise) [S10, unofficial]. 2D texture slicing generally outperforms 3D texture slicing for large volumes due to better cache locality and hardware optimization for 2D textures [S8, unofficial].
*   **Strengths:** Works on hardware with only 2D texture support (e.g., early GPUs, some mobile devices) [S1][S8, unofficial]. Allows real-time modification of colormaps via palette modification if using indexed colors [S5].
*   **Limitations:** High memory usage (3 copies). Visual artifacts (popping, aliasing) when view direction changes or is edge-on. Less flexible than ray-casting methods [S1, S5].
*   **When to Use:** When 3D texture hardware is unavailable, or for very large volumes where 3D texture memory limits are prohibitive and 2D paging is acceptable [S5][S8, unofficial].
*   **When Not to Use:** When high-quality, artifact-free rendering across all view angles is required, or when memory bandwidth is a bottleneck [S4, S8, unofficial].

## Background
The method was devised by Cabral et al. [1994] and ran on SGI Reality Engine workstations [S6, unofficial]. It was one of the first texture-based volume rendering techniques, leveraging the graphics hardware's texture mapping capabilities before 3D texture mapping became standard [S6, S8, unofficial]. It is historically significant as a bridge between software ray casting and modern GPU-based volume rendering, particularly for systems lacking 3D texture units [S2][S8, unofficial].

## Sources
[S1] State-of-the-Art in GPU-Based Large-Scale Volume Visualization, https://vcg.seas.harvard.edu/publications/20150513-stateof-the-art-in-gpu-based-large-scale-volume-visualization/paper, fetched 2026-10-24
[S2] Shear-image order ray casting volume rendering, https://doi.org/10.1145/641480.641510, fetched 2026-10-31
[S3] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S4] Visualisation de données volumiques massives : application aux données sismiques, https://hal.univ-lorraine.fr/tel-01752737v1, fetched 2026-10-26 (unofficial)
[S5] Interactive Multi-volume Visualization, https://doi.org/10.1007/3-540-46080-2_11, fetched 2026-10-26
[S6] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S7] 2D-texture based volume rendering, https://stackoverflow.com/questions/23047355/2d-texture-based-volume-rendering, fetched 2026-10-27 (unofficial)
[S8] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
[S9] Interactive Volume Rendering Using 3D Texture-Mapping ..., https://web.cs.ucdavis.edu/~okreylos/PhDStudies/Winter2000/TextureMapping.html, fetched 2026-10-27
[S10] PDF Real-time Procedural Volumetric Fire - UC Davis, https://web.cs.ucdavis.edu/~hamann/FullerKrishnanMahrousHamannJoyFirePaperFor_I3D2007AsSubmitted11012006.pdf, fetched 2026-10-27 (unofficial)
[S11] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
[S12] Interactive Volume Visualization, https://web.cs.ucdavis.edu/~okreylos/ResDev/VolVis/MainPage.html, fetched 2026-10-28
[S13] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
