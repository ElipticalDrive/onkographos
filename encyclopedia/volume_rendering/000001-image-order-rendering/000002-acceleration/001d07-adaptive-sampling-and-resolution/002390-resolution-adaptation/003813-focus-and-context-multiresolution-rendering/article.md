# Focus and Context Multiresolution Rendering

Also called: Interactive lenses multiresolution, Time-critical multiresolution rendering

## Overview
Focus and Context Multiresolution Rendering is a technique to balance processing capacity between a high-detail focus region and a lower-detail context region to improve interactivity in time-critical applications [S1, unofficial]. It sets itself apart from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by explicitly separating the scene into two distinct rendering passes with different resolutions and complexity levels, rather than adapting a single view's resolution based on distance or error metrics. The goal is not seamless integration but a tradeoff between achievable quality and speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, a transfer function, a camera viewpoint, and a defined "focus" region (typically a probe or specific object of interest).
*   **Output:** A composite image where the focus region is rendered at high resolution and the context region at lower resolution.

**Data Structures:**
The method utilizes two distinct rendering configurations, often defined by parameters such as those in Table 1 of the source [S1, unofficial]:

| Parameter | Focus Configuration | Context Configuration |
| :--- | :--- | :--- |
| **Volume Resolution** | $4 \times 256^3$ (or full resolution) | $1 \times 128^3$ (subsampled) |
| **Lookup Table Size** | $5 \times 256^2$ | $4 \times 128^2$ |
| **Rendering Model** | Local lighting (shaded) | Unshaded |
| **Sampling Rate** | 4 | 1 |
| **Viewport Size** | 100% (of focus area) | 50% (of context area) |

**Steps:**
1.  **Decomposition:** The scene is divided into a focus region and a context region. The focus is typically attached to a data probe or specific object, while the context provides orientation or external view [S1, unofficial].
2.  **Context Rendering:**
    *   Render the context volume into a smaller viewport (e.g., 50% of the standard size) [S1, unofficial].
    *   Use a simpler rendering model (unshaded) and lower sampling rate (e.g., 1) to reduce fragment count [S1, unofficial].
    *   If the transfer function contains only low-frequency components, sparse sampling may be used for the context [S1, unofficial].
    *   Apply a Gaussian filter to the resulting context image to reduce aliasing artifacts introduced by the lower resolution and viewport scaling [S1, unofficial].
3.  **Focus Rendering:**
    *   Render the focus volume into the remaining viewport area (or the full viewport if the context is composited separately) [S1, unofficial].
    *   Use high-resolution data, larger lookup tables, local lighting, and higher sampling rates (e.g., 4) [S1, unofficial].
    *   The focus is placed closest to the user in the depth buffer [S1, unofficial].
4.  **Compositing:**
    *   Enlarge and composite the rendered context image with the focus image [S1, unofficial].
    *   If the context is transparent, it is used for orientation; if opaque, it provides an external view [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_focus_context(volume, transfer_function, camera, focus_region, context_region):
    # Step 1: Render Context
    # Render context into a smaller viewport to reduce fragment count
    context_viewport_scale = 0.5
    context_resolution = downsample(volume, factor=2) # e.g., 128^3 from 256^3
    context_lut = reduce_lut(transfer_function, resolution=128)
    
    # Render context with unshaded model and low sampling
    context_image = ray_march(
        volume=context_resolution,
        lut=context_lut,
        sampling_rate=1,
        shading_model="unshaded",
        viewport_scale=context_viewport_scale
    )
    
    # Step 2: Filter Context to reduce aliasing
    context_image = gaussian_filter(context_image, sigma=1.0)
    
    # Step 3: Render Focus
    # Render focus at full resolution with high detail
    focus_lut = transfer_function # Full resolution LUT
    focus_image = ray_march(
        volume=volume,
        lut=focus_lut,
        sampling_rate=4,
        shading_model="local_lighting",
        viewport_scale=1.0,
        clip_region=focus_region
    )
    
    # Step 4: Composite
    # Enlarge context image to match focus viewport size if necessary
    context_image_resized = resize(context_image, target_size=focus_image.shape)
    
    # Composite based on context mode (transparent or opaque)
    if context_mode == "transparent":
        # Alpha blend context over focus or vice versa depending on depth
        final_image = composite_transparent(focus_image, context_image_resized)
    else:
        # Opaque context might replace focus or be rendered behind
        final_image = composite_opaque(focus_image, context_image_resized)
        
    return final_image
```

**Edge Cases and Pitfalls:**
*   **Overhead:** Rendering slices multiple times and setting up clip textures/depth buffers for each pass introduces overhead [S1, unofficial].
*   **Seamlessness:** The method does not aim for seamless integration; visible boundaries between focus and context are acceptable as a trade-off for speed [S1, unofficial].
*   **Hardware Variability:** Performance can vary significantly depending on the graphics hardware, particularly regarding the efficiency of slicing 3D textures from different directions [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method reduces the number of fragments processed in the fragment pipeline by rendering the context at a lower resolution and sampling rate [S1, unofficial].
*   **Performance:** Achieves 2-6 times faster rendering rates than single high-resolution mode by reducing focus fragments aggressively [S1, unofficial].
*   **Measured Performance:**
    *   Hardware: 2.0GHz Pentium 4, 2GB memory, ATI Radeon 9800 Pro [S1, unofficial].
    *   512^2 Viewport:
        *   100% Focus Size: 3.39 FPS
        *   50% Volume Size: 4.22 FPS
        *   25% Volume Size: 7.55 FPS
        *   12.5% Volume Size: 12.99 FPS
        *   6.25% Volume Size: 20.87 FPS
    *   1024^2 Viewport:
        *   100% Focus Size: 1.06 FPS
        *   50% Volume Size: 1.28 FPS
        *   25% Volume Size: 2.31 FPS
        *   12.5% Volume Size: 3.97 FPS
        *   6.25% Volume Size: 6.33 FPS
    [S1, unofficial]
*   **Strengths:** Simple, generic approach; speedup depends on rendering parameters, not transfer functions; effective for interactivity in time-critical applications [S1, unofficial].
*   **Limitations:** Not seamless; overhead from multiple rendering passes; performance sensitive to hardware-specific texture slicing efficiency [S1, unofficial].
*   **When to Use:** When interactivity is critical and the user needs to explore a specific region of interest (focus) while maintaining orientation (context).
*   **When Not to Use:** When seamless visual continuity is required or when the transfer function has high-frequency components that require full resolution sampling even in the context [S1, unofficial].

## Background
*   **Origin:** The technique was presented in "A Focus and Context Interface for Interactive Volume Rendering" by I. Kits et al. [S1, unofficial].
*   **History:** It was developed to address the limitations of hardware-accelerated volume rendering in time-critical applications, where balancing processing capacity between detail and speed is essential [S1, unofficial].
*   **Usage:** Used in interactive volume visualization tools, particularly for exploring multidimensional or multifield datasets where clutter reduction and orientation are key [S1, unofficial].

## Sources
[S1] A Focus and Context Interface for Interactive Volume Rendering, https://www.sci.utah.edu/~ikits/publications/Ikits_Unpublished_04.pdf, fetched 2026-10-27 (unofficial)
