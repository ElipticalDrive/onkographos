# Focus and Context Multiresolution Rendering

Also called: Interactive lenses multiresolution, Time-critical multiresolution rendering

## Overview
Focus and Context Multiresolution Rendering is a technique to balance processing capacity between a high-detail focus region and a lower-detail context region to improve interactivity in time-critical applications [S1, unofficial]. It sets itself apart from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by explicitly decoupling the rendering pipeline into two distinct passes with different resolutions, sampling rates, and shading models, rather than dynamically adjusting a single ray's step size based on local metrics. The goal is not seamless integration but a tradeoff between quality and speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, a transfer function (TF), a camera view, and a focus region definition (typically a probe or bounding box).
*   **Output:** A composite 2D image where the focus region is rendered at high fidelity and the context region at reduced fidelity.

**Data Structures:**
The method utilizes two distinct rendering configurations, often defined by parameters such as those in Table 1 of the source [S1, unofficial]:

| Parameter | Focus Configuration | Context Configuration |
| :--- | :--- | :--- |
| **Volume Resolution** | $4 \times 256^3$ (or full resolution) | $1 \times 128^3$ (subsampled) |
| **Lookup Table Size** | $5 \times 256^2$ | $4 \times 128^2$ |
| **Rendering Model** | Local lighting (shaded) | Unshaded |
| **Sampling Rate** | 4 (dense) | 1 (sparse) |
| **Viewport Size** | 100% (or defined focus area) | 50% (or remaining area) |

**Steps:**
1.  **Decomposition:** Divide the rendering task into two passes: Focus and Context. The focus region is defined by a data probe or spatial constraint. The context is the remainder of the view [S1, unofficial].
2.  **Context Rendering:**
    *   Render the volume using the Context Configuration.
    *   Draw the context into a smaller viewport (e.g., 50% of the screen) [S1, unofficial].
    *   The context can be rendered transparently (for orientation) or opaquely (for external view) [S1, unofficial].
    *   If the transfer function contains only low-frequency components, the context can be sampled sparsely [S1, unofficial].
3.  **Context Post-Processing:**
    *   Enlarge the rendered context image to fill its designated portion of the final frame buffer.
    *   Apply a Gaussian filter to the enlarged context image to reduce aliasing artifacts introduced by the lower resolution and sparse sampling [S1, unofficial].
4.  **Focus Rendering:**
    *   Render the volume using the Focus Configuration.
    *   Render the focus region at full resolution and with complex shading (local lighting) [S1, unofficial].
    *   The focus may be attached to a data probe showing multiple attributes, possibly combined with polygonal visualization (icons, streamlines) [S1, unofficial].
5.  **Compositing:**
    *   Composite the processed context image and the high-resolution focus image into the final output.
    *   Note: The integration between regions is not seamless; sharp boundaries may be visible, which is an accepted trade-off for performance [S1, unofficial].

**Pitfalls:**
*   **Overhead:** Performance gains are offset by the overhead of rendering slices multiple times and setting up clip textures and depth buffers for each pass [S1, unofficial].
*   **Aliasing:** Without the Gaussian filter, the upsampled context region will exhibit significant aliasing artifacts [S1, unofficial].
*   **Transfer Function Complexity:** The sparse sampling of the context is only valid if the transfer function has low-frequency components. High-frequency TFs will require higher sampling rates, negating the speedup [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_focus_and_context(volume, tf, camera, focus_region, screen_width, screen_height):
    # 1. Setup Context Render Target (smaller viewport)
    context_viewport_width = screen_width // 2
    context_viewport_height = screen_height // 2
    context_framebuffer = create_framebuffer(context_viewport_width, context_viewport_height)
    
    # 2. Render Context Pass
    # Use subsampled volume (128^3) and unshaded TF
    render_volume(
        volume=volume.downsample(128),
        tf=tf.downsample(128),
        camera=camera,
        viewport=(0, 0, context_viewport_width, context_viewport_height),
        shading_model="unshaded",
        sampling_rate=1
    )
    
    # 3. Post-process Context
    # Enlarge to full context area (e.g., left half of screen)
    context_image = context_framebuffer.read()
    enlarged_context = resize(context_image, (screen_width // 2, screen_height))
    
    # Apply Gaussian blur to reduce aliasing
    blurred_context = gaussian_filter(enlarged_context, sigma=1.0)
    
    # 4. Setup Focus Render Target (full resolution or focus area)
    focus_framebuffer = create_framebuffer(screen_width, screen_height)
    
    # 5. Render Focus Pass
    # Use full resolution volume and shaded TF
    render_volume(
        volume=volume,
        tf=tf,
        camera=camera,
        viewport=focus_region.get_viewport_bounds(screen_width, screen_height),
        shading_model="local_lighting",
        sampling_rate=4
    )
    focus_image = focus_framebuffer.read()
    
    # 6. Composite
    final_image = create_framebuffer(screen_width, screen_height)
    # Place blurred context in its designated area (e.g., left half)
    final_image.blit(blurred_context, dest=(0, 0))
    # Place focus image in its designated area (e.g., right half or overlay)
    # Note: Exact compositing logic depends on focus region definition
    final_image.blit(focus_image, dest=focus_region.get_blit_offset())
    
    return final_image
```

## Operating characteristics
*   **Performance:** Achieves 2-6 times faster rendering rates than single high-resolution mode by reducing focus fragments aggressively [S1, unofficial].
*   **Measured Performance (ATI Radeon 9800 Pro, 2.0GHz Pentium 4, 2GB RAM):**
    *   **512x512 Viewport:**
        *   100% Focus Size: 3.39 FPS
        *   50% Volume Size: 4.22 FPS
        *   25% Volume Size: 7.55 FPS
        *   12.5% Volume Size: 12.99 FPS
        *   6.25% Volume Size: 20.87 FPS
    *   **1024x1024 Viewport:**
        *   100% Focus Size: 1.06 FPS
        *   50% Volume Size: 1.28 FPS
        *   25% Volume Size: 2.31 FPS
        *   12.5% Volume Size: 3.97 FPS
        *   6.25% Volume Size: 6.33 FPS
    *   *Note:* Results reflect peak performance with acceleration techniques (early ray termination, empty space skipping) turned off to isolate the effect of the multiresolution decomposition [S1, unofficial].
*   **Strengths:** Simple, generic approach; speedup depends on rendering parameters, not transfer functions; effective for time-critical applications.
*   **Limitations:** Not seamless; overhead from multiple render passes and depth buffer setup; performance variability on ATI cards due to slicing direction [S1, unofficial].
*   **When to use:** Interactive exploration of multidimensional or multifield datasets where orientation (context) is less critical than local detail (focus).
*   **When not to use:** When seamless visual continuity is required, or when the transfer function has high-frequency components that cannot be safely downsampled.

## Background
Originated in unpublished research by I. Kits et al. [S1, unofficial]. The method addresses the limitations of hardware-accelerated volume rendering where performance is constrained by fragment pipeline throughput and cache utilization. It builds on the principle that not all parts of a volume require the same level of detail for effective interaction.

## Sources
[S1] A Focus and Context Interface for Interactive Volume Rendering, https://www.sci.utah.edu/~ikits/publications/Ikits_Unpublished_04.pdf, fetched 2026-10-27 (unofficial)
