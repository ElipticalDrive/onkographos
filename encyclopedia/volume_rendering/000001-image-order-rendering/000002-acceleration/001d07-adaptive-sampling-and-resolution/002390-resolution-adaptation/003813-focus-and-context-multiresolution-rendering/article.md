# Focus and Context Multiresolution Rendering

Also called: Interactive lenses multiresolution, Time-critical multiresolution rendering

## Overview
Focus and Context Multiresolution Rendering is a technique to balance processing capacity between a high-detail focus region and a lower-detail context region to improve interactivity in time-critical applications [S1, unofficial]. It sets itself apart from siblings like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by decoupling resolution not just by distance, but by semantic importance (focus vs. context), often utilizing viewport resizing and simplified rendering models for the context to aggressively reduce fragment count [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, a transfer function (TF), a camera view, and a defined "focus" region (e.g., attached to a data probe or specific interest point).
*   **Output:** A composite image where the focus region is rendered at high resolution/quality and the context region is rendered at lower resolution/quality and composited.

**Data Structures:**
The method relies on two distinct rendering configurations, typically defined by parameters such as those in Table 1 of the source [S1, unofficial]:

| Parameter | Focus Configuration | Context Configuration |
| :--- | :--- | :--- |
| **Volume Data** | $4 \times 256^3$ (or full resolution) | $1 \times 128^3$ (subsampled) |
| **Lookup Table** | $5 \times 256^2$ | $4 \times 128^2$ |
| **Rendering Model** | Local lighting (shaded) | Unshaded |
| **Sampling Rate** | 4 | 1 |
| **Viewport Size** | 100% (or specific focus area) | 50% (or remaining area) |

*Note: The volume size for context is often reduced along one axis or globally subsampled. The lookup table size is reduced to match the context volume resolution.*

**Steps:**
1.  **Decomposition:** Divide the rendering task into two passes: Focus and Context.
2.  **Context Rendering:**
    *   Render the volume data using the Context Configuration.
    *   The context is drawn into a smaller viewport (e.g., 50% of the screen or a specific region) [S1, unofficial].
    *   If the context is transparent (for orientation), it may be sampled sparsely if the transfer function contains only low-frequency components [S1, unofficial].
    *   If the context is opaque (for external view), it is rendered fully but with lower resolution [S1, unofficial].
    *   **Filtering:** Apply a Gaussian filter to the context image to reduce aliasing artifacts resulting from the lower resolution and viewport enlargement [S1, unofficial].
3.  **Focus Rendering:**
    *   Render the volume data using the Focus Configuration.
    *   The focus is placed closest to the user (mimicking typical interface usage) [S1, unofficial].
    *   This pass handles high-quality rendering, complex transfer functions, and local lighting [S1, unofficial].
4.  **Compositing:**
    *   Enlarge and composite the context image into the final frame.
    *   Overlay the focus image. The goal is not seamless integration but a tradeoff between quality and speed [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_focus_context(volume_data, tf, camera, focus_region, screen_width, screen_height):
    # 1. Setup Context Pass
    # Use subsampled volume and smaller LUT
    context_volume = subsample(volume_data, factor=2) # Example: 128^3 from 256^3
    context_lut = reduce_lut(tf, resolution=128)
    
    # Define context viewport (e.g., 50% size)
    context_viewport_w = screen_width // 2
    context_viewport_h = screen_height // 2
    
    # Render context into a smaller framebuffer
    context_framebuffer = render_volume(
        volume=context_volume,
        tf=context_lut,
        camera=camera,
        viewport=(context_viewport_w, context_viewport_h),
        sampling_rate=1,
        shaded=False
    )
    
    # Apply Gaussian filter to reduce aliasing
    context_framebuffer = gaussian_filter(context_framebuffer, sigma=1.0)
    
    # 2. Setup Focus Pass
    # Use full resolution volume and larger LUT
    focus_volume = volume_data
    focus_lut = tf
    focus_viewport_w = screen_width
    focus_viewport_h = screen_height
    
    # Render focus into full framebuffer (or specific focus area)
    # Note: In practice, focus might be rendered to a separate buffer 
    # and composited, or rendered last to cover context.
    focus_framebuffer = render_volume(
        volume=focus_volume,
        tf=focus_lut,
        camera=camera,
        viewport=(focus_viewport_w, focus_viewport_h),
        sampling_rate=4,
        shaded=True,
        lighting_model="local"
    )
    
    # 3. Composite
    # The focus region is typically rendered on top or composited 
    # based on depth/clip planes.
    final_image = composite(context_framebuffer, focus_framebuffer)
    
    return final_image
```

**Parameters:**
*   **Context Rendering Mode:** Transparent (for orientation) or Opaque (for external view) [S1, unofficial].
*   **Sampling Rate:** Focus uses higher sampling (e.g., 4), Context uses lower (e.g., 1) [S1, unofficial].
*   **Viewport Size:** Context can be rendered in a fraction of the screen (e.g., 50%) to reduce fragment count [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Overhead:** Rendering slices multiple times and setting up clip textures/depth buffers for each pass introduces overhead [S1, unofficial].
*   **Aliasing:** Without filtering, the enlarged context image will exhibit aliasing artifacts [S1, unofficial].
*   **Seamlessness:** The method does not aim for seamless integration; visible boundaries may exist [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Performance depends on the number of fragments processed. By reducing the context viewport and sampling rate, the fragment count is significantly reduced [S1, unofficial].
*   **Measured Performance:** On a 2.0GHz Pentium 4, 2GB memory, ATI Radeon 9800 Pro [S1, unofficial]:
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
*   **Speedup:** Achieves 2-6 times faster rendering rates than single high-resolution mode by reducing focus fragments aggressively [S1, unofficial].
*   **Strengths:** Simple, generic approach; speedup depends on rendering parameters, not transfer functions [S1, unofficial].
*   **Limitations:** Overhead from multiple passes; not seamless; performance variability on different hardware (e.g., ATI cards) [S1, unofficial].

## Background
Originated in the paper "A Focus and Context Interface for Interactive Volume Rendering" [S1, unofficial]. It was developed to address the need for interactivity in time-critical volume rendering applications by balancing processing capacity between focus and context regions.

## Sources
[S1] A Focus and Context Interface for Interactive Volume Rendering, https://www.sci.utah.edu/~ikits/publications/Ikits_Unpublished_04.pdf, fetched 2026-10-27 (unofficial)
